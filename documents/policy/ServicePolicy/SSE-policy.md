## 실시간 알림 (SSE / Redis Pub/Sub) 정책

### 알림 기본 정책

- 결제 승인 완료 후 구매자에게 실시간 알림을 전송한다.
- 알림은 Kafka Consumer가 `PAYMENT_APPROVED` 이벤트를 수신하여 생성한다.
- 알림 내역은 `notification` 테이블에 저장한다.
- 실시간 전송은 SSE(Server-Sent Events) 방식으로 처리한다.
- 다중 서버 환경에서 SSE 연결이 존재하는 서버와 알림 이벤트를 처리하는 서버가 다를 수 있으므로, 서버 간 알림 전파에는 Redis Pub/Sub을 사용한다.
- SSE 연결이 없는 사용자에게는 실시간 전송이 되지 않을 수 있으나, 알림은 DB에 저장되어 이후 조회할 수 있다.
- 알림 대상은 현재 구매자(userId)만 해당한다.
- 향후 배송 상태 변경, 쿠폰 만료, 재고 부족 등으로 알림 유형을 확장할 수 있다.

### Kafka Consumer 정책

- Consumer Group ID: `one-stop-notification-group`
- 구독 토픽: `payment.approved`
- `PaymentApprovedEventPayload`를 역직렬화하여 알림 생성을 요청한다.
- Consumer는 `eventId`를 기준으로 알림 중복 처리가 가능하도록 Notification 저장 로직에 위임한다.
- 실제 중복 방어는 `notification.event_id` UNIQUE 제약과 Notification 저장 로직에서 수행한다.
- 동일 `eventId`의 알림이 이미 존재하면 중복 저장하지 않고 건너뛴다.
- Consumer 처리 실패 시 로그를 기록한다.
- 알림 Consumer의 처리 결과는 OutboxEvent 상태에는 직접 영향을 주지 않는다.

### Kafka Consumer 에러 처리 정책

- payload 역직렬화 실패는 재시도해도 복구 가능성이 낮으므로 로그만 남기고 스킵한다.
- 알림 저장 등 비즈니스 로직 실패는 예외를 전파하여 Kafka 재처리 대상이 되도록 한다.
- 알림 Consumer의 실패는 OutboxEvent 상태를 직접 변경하지 않는다.

### Consumer 멱등성 정책

- 멱등성 보장은 `notification.event_id` UNIQUE 제약으로 처리한다.
- 별도 `processed_event` 테이블을 두지 않고, `notification` 테이블 자체가 알림 이벤트 처리 기록 역할을 한다.
- 처리 흐름은 다음과 같다.

```
1. Kafka 메시지 수신
2. payload 역직렬화
3. 역직렬화 실패 시 로그 기록 후 스킵
4. eventId로 notification 테이블 조회
5. 이미 존재하면 중복 이벤트로 보고 건너뛰기
6. 존재하지 않으면 Notification 저장 시도
7. UNIQUE 제약 위반 시 중복 이벤트로 보고 건너뛰기
8. Notification 저장 성공 시 Redis Pub/Sub 발행 예약
9. Notification 저장 트랜잭션 커밋 이후 Redis Pub/Sub 메시지 발행
```

- `existsByEventId` 조회는 1차 중복 체크 역할을 한다.
- `notification.event_id` UNIQUE 제약은 동시 수신 상황에 대한 최종 중복 방어 역할을 한다.
- 중복 이벤트로 판단한 경우 Notification 저장 및 Redis Pub/Sub 발행을 수행하지 않는다.

### Notification 저장 정책

- 알림은 실시간 전송 여부와 관계없이 DB에 먼저 저장한다.
- Notification은 알림 내역 조회, 읽음/안읽음 처리, 오프라인 사용자 대응을 위한 영속 데이터이다.
- Redis Pub/Sub은 메시지를 영속 저장하지 않으므로, 알림 정합성은 Notification DB 저장을 기준으로 보장한다.
- Notification 저장이 실패하면 Redis Pub/Sub 메시지를 발행하지 않는다.
- Notification 저장이 성공하고 트랜잭션이 커밋된 이후에만 Redis Pub/Sub 메시지를 발행한다.
- Redis Pub/Sub 발행 실패는 Notification DB 저장 결과에 영향을 주지 않는다.

### Redis Pub/Sub 기반 알림 브로드캐스트 정책

- Redis Pub/Sub 채널명은 `notification:sse`를 사용한다.
- Notification 저장 후 Redis Pub/Sub 채널로 알림 메시지를 발행한다.
- Redis Pub/Sub 메시지는 `NotificationPubSubMessage` 형식으로 전달한다.
- Redis Pub/Sub 메시지에는 SSE 전송에 필요한 최소 정보를 포함한다.

```
notificationId
userId
eventId
type
title
message
createdAt
```

- 모든 서버 인스턴스는 동일한 Redis Pub/Sub 채널을 구독한다.
- 각 서버 인스턴스는 Redis 메시지를 수신한 뒤 `SseConnectionManager.send(userId, response)`를 호출한다.
- `SseConnectionManager`는 자기 서버 메모리에 해당 `userId`의 `SseEmitter`가 있는 경우에만 SSE 알림을 전송한다.
- 해당 사용자의 `SseEmitter`가 현재 서버에 없으면 전송을 시도하지 않고 스킵한다.
- 다중 서버 환경에서는 실제 SSE 연결을 가진 서버만 브라우저에 알림을 전송한다.

### Redis Pub/Sub 발행 시점 정책 - 6.15(월) 수정

- Redis Pub/Sub 발행은 원칙적으로 Notification DB 저장 트랜잭션이 커밋된 이후 수행한다.
- DB 트랜잭션이 롤백되었는데 실시간 알림만 먼저 발행되는 상황을 방지하기 위함이다.
- 트랜잭션 동기화가 활성화된 경우 `afterCommit` 시점에 Redis Pub/Sub 메시지를 발행한다.
- 트랜잭션 동기화가 없는 경우에는 별도의 커밋 대기 지점이 없으므로 즉시 Redis Pub/Sub 메시지를 발행한다.
- 트랜잭션 동기화가 없는 즉시 발행 경로는 테스트 또는 트랜잭션 외부 호출 환경을 방어하기 위한 fallback 처리이다.
- Redis Pub/Sub 메시지 직렬화 실패 시 Redis 발행을 수행하지 않고 로그만 기록한다.
- Redis Pub/Sub 발행 실패 시 예외를 전파하지 않고 로그만 기록한다.
- Redis Pub/Sub 발행 실패는 Notification DB 저장 결과에 영향을 주지 않는다.

### Redis Pub/Sub 수신 정책

- Redis Subscriber는 Redis Pub/Sub 메시지를 수신한 뒤 JSON 역직렬화를 수행한다.
- 역직렬화에 실패한 메시지는 재처리해도 복구 가능성이 낮으므로 로그만 기록하고 스킵한다.
- 정상 메시지는 `NotificationSseResponse`로 변환하여 SSE 전송에 사용한다.
- SSE 전송 실패가 발생해도 예외를 전파하지 않는다.
- SSE 전송 실패는 실시간 전달 실패일 뿐이며, 이미 저장된 Notification DB 데이터에는 영향을 주지 않는다.
- SSE 전송 실패 시 emitter 제거 여부는 `SseConnectionManager` 정책을 따른다.

### SSE 연결 정책

- SSE 구독 엔드포인트: `GET /api/notifications/subscribe`
- 인증된 사용자만 SSE 구독이 가능하다.
- 현재 알림 대상은 구매자(BUYER) 기준이다.
- SseEmitter 타임아웃: 30분 (1800000ms)
- 연결 시 초기 이벤트로 `connect` 이벤트를 전송하여 연결 성공을 알린다.
- 타임아웃, 에러, 완료 시 emitter를 자동 제거한다.
- 동일 사용자가 재연결하면 기존 emitter를 교체한다.
- SSE 연결은 사용자당 서버 인스턴스 기준 1개만 유지한다.
- SseEmitter는 각 서버 인스턴스의 메모리에서 관리한다.

### SseConnectionManager emitter 관리 정책

- `SseConnectionManager`는 서버 인스턴스 메모리의 `ConcurrentHashMap`에서 `userId` 기준으로 `SseEmitter`를 관리한다.
- SseEmitter 타임아웃은 30분으로 설정한다.
- 동일 사용자가 재연결하면 기존 emitter를 `complete()` 처리하고 새 emitter로 교체한다.
- SSE 연결이 완료, 타임아웃, 에러 상태가 되면 해당 emitter를 제거한다.
- 연결 직후 초기 `connect` 이벤트 전송에 실패하면 해당 emitter를 제거하고 `completeWithError()` 처리한다.
- 알림 전송 시 현재 서버 인스턴스에 해당 `userId`의 emitter가 없으면 전송을 시도하지 않고 스킵한다.
- 알림 전송 중 예외가 발생하면 해당 emitter를 제거하고 `completeWithError()` 처리한다.
- 명시적으로 연결을 해제하는 경우 emitter를 제거하고 `complete()` 처리한다.
- emitter 제거 시 `remove(userId, emitter)` 방식으로 현재 저장된 emitter와 동일한 객체인 경우에만 제거한다.
- 따라서 재연결 이후 이전 emitter의 콜백이 늦게 실행되더라도 새 emitter를 잘못 제거하지 않는다.

### SSE 전송 정책

- 실제 SSE 전송은 `SseConnectionManager`를 통해 수행한다.
- Redis Subscriber가 수신한 메시지를 `NotificationSseResponse`로 변환한 뒤 `SseConnectionManager.send(userId, response)`를 호출한다.
- `SseConnectionManager`는 현재 서버 인스턴스 메모리에 해당 `userId`의 emitter가 있는 경우에만 SSE 알림을 전송한다.
- 현재 서버 인스턴스에 해당 `userId`의 emitter가 없으면 전송을 시도하지 않고 스킵한다.
- SSE 알림 전송 중 예외가 발생하면 `SseConnectionManager`가 해당 emitter를 제거하고 `completeWithError()` 처리한다.
- SSE 전송 실패는 실시간 전달 실패일 뿐이며, 이미 저장된 Notification DB 데이터에는 영향을 주지 않는다.
- 오프라인 사용자는 실시간 알림을 받지 못할 수 있으나, Notification DB에 저장된 알림 내역은 이후 조회할 수 있다.

### SSE 알림 메시지 포맷

```json
{
  "notificationId": 1,
  "type": "PAYMENT_APPROVED",
  "title": "결제 완료",
  "message": "주문 #13 결제가 완료되었습니다. (결제 금액: 192,000원)",
  "createdAt": "2026-06-05T20:30:00"
}
```

### 알림 유형

| 유형 | 대상 | 설명 | 구현 상태 |
| --- | --- | --- | --- |
| PAYMENT_APPROVED | 구매자 | 결제 완료 알림 | 구현 완료 |
| DELIVERY_DEPARTED | 구매자 | 배송 출발 알림 | 향후 확장 |
| DELIVERY_COMPLETED | 구매자 | 배송 완료 알림 | 향후 확장 |
| ORDER_RECEIVED | 판매자 | 새 주문 접수 알림 | 향후 확장 |
| STOCK_LOW | 판매자 | 재고 부족 알림 | 향후 확장 |
| COUPON_EXPIRING | 구매자 | 쿠폰 만료 임박 알림 | 향후 확장 |

### 역할 분리

```
Kafka:
- 결제 완료 등 도메인 이벤트 전달
- Outbox Pattern과 함께 이벤트 유실 방지 및 비동기 처리 담당

DB:
- Notification 영속 저장
- 읽음/안읽음 상태 관리
- 오프라인 사용자 알림 내역 보존

Redis Pub/Sub:
- 다중 서버 간 실시간 알림 브로드캐스트
- 모든 서버 인스턴스에 알림 메시지 전파
- 메시지 영속 저장은 수행하지 않음

SSE:
- 브라우저와 서버 간 실시간 알림 전달
- 실제 사용자 연결을 가진 서버에서만 전송 수행
```

### 제약 사항

- Redis Pub/Sub은 메시지를 영속 저장하지 않는다.
- 따라서 실제 알림 내역은 반드시 Notification DB에 저장한다.
- Redis Pub/Sub은 서버 간 실시간 전파 용도로만 사용한다.
- SseEmitter는 각 서버 인스턴스의 메모리에 저장되므로, 서버 재시작 시 기존 SSE 연결은 모두 끊어진다.
- 알림 조회, 읽음 처리 API는 현재 구현 범위에 포함하지 않는다.
- 알림 정리 배치(오래된 알림 삭제)는 현재 구현 범위에 포함하지 않는다.
- 동일 사용자가 여러 서버 인스턴스에 동시에 SSE 연결을 생성한 경우, 각 서버 인스턴스의 emitter 상태에 따라 동일 알림이 중복 전송될 수 있으므로 클라이언트는 일반적으로 하나의 SSE 연결만 유지한다.