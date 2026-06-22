## 결제

- Mock 결제 기반 구현 (실제 PG사 연동 없음)
- TossPayments API 구조를 참고한 2단계 결제 플로우 적용
    - 1단계: 주문 생성 및 재고 선점
        - 주문 상태: PENDING_PAYMENT
    - 2단계: 결제 승인 및 포인트/쿠폰 차감
        - Order 상태: PAID
        - Payment 상태: PAID
        - 결제 승인 완료 후 `DeliveryService.createDeliveriesForPayment()`를 통해 주문 상품 접수 및 배송 생성을 처리
        - 주문 상품(order_item)별 Delivery 생성
        - 최초 배송 상태: ACCEPT
        - DeliveryHistory에 ACCEPT 이력 저장
- 결제 금액 불일치 시 결제 승인을 거절하며, 결제 승인 트랜잭션은 롤백된다.
- 이 경우 Payment는 생성되지 않으며, `Payment.status = FAILED`로 저장하지 않는다.
- 동일 orderId에 대한 중복 결제 승인 불가
- 결제 승인 시 동일 주문에 대한 중복 승인 요청을 방지하기 위해 `Order`를 비관적 락으로 조회한 뒤 결제 가능 여부를 검증한다.
- 이미 PAID 상태인 주문 재결제 불가
- 현재 구현은 Mock 결제 승인 성공 플로우를 기준으로 한다.
- 별도의 PG 결제 실패 콜백, 결제 실패 API, Payment 실패 저장 플로우는 현재 구현 범위에 포함하지 않는다.
- 결제 승인 요청 중 주문 상태 오류, 결제 금액 불일치, 포인트 사전 검증 실패, 포인트 차감 실패, 쿠폰 사용 실패, 배송 생성 실패 등이 발생하면 결제 승인 트랜잭션은 롤백된다.
- 이 경우 Payment는 생성되지 않거나 승인 상태로 저장되지 않으며, `Payment.fail()`은 현재 Mock 결제 흐름에서 사용하지 않는다.
- 결제 승인 실패로 인한 `PENDING_PAYMENT → CANCELLED` 자동 전환은 현재 구현 범위에 포함하지 않는다.
- 결제 대기 상태(`PENDING_PAYMENT`) 주문의 만료 또는 자동 취소는 향후 주문 만료 배치 또는 실제 PG 연동 단계에서 별도 정책으로 확장한다.
- 실제 PG 연동 또는 결제 실패 콜백을 도입하는 경우, 결제 실패 시 `Order.status = CANCELLED`, `Payment.status = FAILED`, 선점 재고/포인트/쿠폰 복구를 별도 실패 보정 트랜잭션으로 처리한다.
- 결제 후 주문 취소
    - PENDING_PAYMENT → PAID → CANCELLED
- 결제 완료 주문이 취소되면 Payment 상태도 CANCELLED로 변경한다.
- 결제 상태
    - READY: 결제 대기
    - PAID: 결제 성공
    - FAILED: 결제 실패
        - 현재 Mock 결제 승인 성공 플로우에서는 사용하지 않는다.
        - 실제 PG 연동 또는 결제 실패 콜백 도입 시 사용한다.
    - CANCELLED: 결제 취소/환불
- 결제 방법(방법에 따라 결제 시간이 다름)
    - MOCK: 현재 구현 용 가짜 결제
    - CARD: 추후 PG(토스 등) 카드 결제 연동용
    - POINT: 포인트 결제용

### 결제 API 요청 제한 정책

- 결제 승인 API는 사용자당 1분 5회로 제한한다.
- 제한 초과 시 `429 Too Many Requests` 응답을 반환한다.
- 식별 기준은 `userId`이다.
- Redis Lua Script 기반 원자적 카운팅으로 처리한다.
- Redis 장애 시 Fail-Open 정책으로 요청을 차단하지 않는다.

### 결제 승인 중복 방어 정책

- 동일 `orderId`에 대한 결제 승인은 한 번만 허용한다.
- 결제 승인 요청이 들어오면 먼저 `Order`를 비관적 락으로 조회한다.
- 락 획득 후 다음 조건을 검증한다.
    1. 요청 사용자가 주문 소유자인지 확인한다.
    2. 주문 상태가 `PENDING_PAYMENT`인지 확인한다.
    3. 이미 동일 주문에 대한 `Payment`가 존재하지 않는지 확인한다.
    4. 요청 결제 금액과 주문 최종 결제 금액이 일치하는지 확인한다.
- 위 검증을 통과한 경우에만 포인트 차감, Payment 생성, Order 상태 변경, 쿠폰 사용, 배송 생성을 수행한다.
- 포인트 실제 차감 전 `paymentPointGuard.validateBeforePaymentApproval()`을 통해 포인트 잔액/이력 정합성을 사전 재검증한다.
- 이미 `PAID` 상태인 주문은 재결제할 수 없다.
- 이미 동일 주문에 대한 `Payment`가 존재하면 결제 승인 요청을 거절하고 예외를 반환한다.
- 이 경우 별도의 `Payment.status = FAILED` 저장은 수행하지 않는다.
- 중복 결제 승인으로 실패한 요청은 포인트를 차감하지 않는다.
- `Payment.order_id`의 UNIQUE 제약은 중복 Payment 저장을 막기 위한 최종 방어 수단으로 둔다.
- 단, UNIQUE 제약은 포인트 차감 이후에 발생할 수 있으므로, 포인트 차감 전 Order 락 기반 검증을 먼저 수행한다.
- 동일 `orderId`에 대한 동시 결제 승인 요청은 Order 비관적 락으로 직렬화되며, 실제 동시 요청 환경에서도 1건만 결제 성공하도록 통합 테스트로 검증한다.
- 동시 결제 승인 요청 중 먼저 커밋된 요청만 Payment 생성, Order PAID 처리, 포인트 차감, 쿠폰 사용, 배송 생성을 수행하며, 이후 요청은 결제 가능 상태 검증에서 실패한다.

## 결제 승인 이벤트 / Outbox 정책

- 결제 승인 완료 후 후속 처리를 위해 `PAYMENT_APPROVED` 이벤트를 생성한다.
- Kafka 이벤트는 결제 승인 트랜잭션 안에서 직접 발행하지 않는다.
- 결제 승인 트랜잭션 안에서는 `outbox_event` 테이블에 이벤트 저장을 시도한다.
- OutboxEvent는 payload 직렬화 및 저장이 정상적으로 수행된 경우 결제 승인 로직과 동일한 DB 트랜잭션 안에서 저장된다.
- OutboxEvent가 저장된 이후 결제 승인 트랜잭션이 롤백되면 OutboxEvent도 함께 롤백된다.
- Kafka 발행은 별도의 Outbox Publisher가 담당한다.

### 결제 승인 OutboxEvent 저장 시점

- `PaymentService.approvePayment()`에서 아래 핵심 도메인 처리가 모두 성공한 뒤 OutboxEvent 저장을 시도한다.
    1. 주문 조회
    2. 주문 소유자 검증
    3. 결제 가능 여부 검증
    4. 포인트 사전 무결성 재검증
        - `paymentPointGuard.validateBeforePaymentApproval()`을 호출하여 포인트 차감 전 포인트 데이터 정합성을 사전 확인한다.
    5. 포인트 실제 차감
    6. Payment 생성 및 승인 처리
    7. Order 상태 `PAID` 변경
    8. 쿠폰 사용 처리
    9. `DeliveryService.createDeliveriesForPayment()` 호출
        - OrderItem 상태 `ORDERED` 변경
        - Delivery 생성
        - DeliveryHistory `ACCEPT` 이력 저장
    10. OutboxEvent 저장 시도
- 1~9번의 핵심 도메인 처리 중 하나라도 실패하면 결제 승인 트랜잭션은 롤백되며 OutboxEvent도 저장하지 않는다.
- 단, OutboxEvent payload 직렬화 실패 또는 OutboxEvent 저장 실패는 결제 승인 성공을 롤백하지 않고 로그로 기록한다.
- `paymentPointGuard`는 포인트 차감 전 포인트 잔액/이력 정합성을 방어하기 위한 사전 검증이며, 실제 포인트 차감은 이후 단계에서 수행한다.

### 결제 승인 OutboxEvent 실패 처리 정책

- OutboxEvent는 결제 승인 후 알림, 정산, 외부 연동 등 후속 비동기 처리를 위한 보조 이벤트이다.
- OutboxEvent payload 직렬화 실패는 결제 성공의 필수 조건으로 보지 않는다.
- payload 직렬화 실패 시 예외를 전파하지 않고 로그만 기록하며, 해당 결제에 대한 OutboxEvent는 저장하지 않는다.
- OutboxEvent 저장 실패도 예외를 전파하지 않도록 방어하며, 장애 추적을 위해 `orderId`, `paymentId`, `eventId`를 로그에 기록한다.
- 동일 `eventId` 중복 저장으로 인한 DB UNIQUE 제약 위반은 `OUTBOX_001`로 변환하여 처리한다.
- OutboxEvent 저장 로직은 사전 중복 조회보다 DB UNIQUE 제약과 `saveAndFlush()` 기반 예외 처리를 우선한다.
- 이는 동시 요청 상황에서 check-then-act 구조로 인해 발생할 수 있는 race condition을 줄이기 위한 정책이다.
- Outbox 처리 실패로 인해 결제는 성공했지만 이벤트가 생성되지 않을 수 있다.
- 이 경우 해당 결제의 후속 비동기 처리, 예를 들어 Kafka 알림 이벤트, 정산, 외부 연동 등은 누락될 수 있으므로 운영 로그 기반 확인 또는 별도 보정 방안을 추후 검토한다.

### 결제 승인 이벤트 식별 정책

- 결제 승인 이벤트의 `event_type`은 `PAYMENT_APPROVED`를 사용한다.
- 결제 승인 이벤트의 `aggregate_type`은 `ORDER`를 사용한다.
- 결제 승인 이벤트의 `aggregate_id`는 `orderId`를 사용한다.
- 결제 승인 이벤트의 Kafka topic은 `payment.approved`를 사용한다.
- 결제 승인 이벤트의 `partition_key`는 `orderId`를 사용한다.
- 동일 주문 기준 이벤트 순서 보장을 위해 Kafka 메시지 Key는 `orderId` 기준으로 설정한다.

```
event_type: PAYMENT_APPROVED
aggregate_type: ORDER
aggregate_id: orderId
topic: payment.approved
partition_key: orderId
```

### 결제 승인 이벤트 Payload 정책

- 결제 승인 이벤트 payload는 JSON 문자열로 저장한다.
- payload에는 Consumer가 후속 처리를 수행하는 데 필요한 최소 정보를 포함한다.
- payload에는 중복 처리 방지를 위해 `eventId`를 포함한다.

예시:

```json
{
  "eventId": "uuid",
  "eventType": "PAYMENT_APPROVED",
  "orderId": 1,
  "paymentId": 10,
  "userId": 3,
  "finalPrice": 2192000,
  "approvedAt": "2026-06-04T10:30:00"
}
```

### OutboxEvent 상태 정책

- `PENDING`
    - 결제 승인 트랜잭션 안에서 OutboxEvent가 저장되었지만 아직 Kafka로 발행되지 않은 상태
    - Outbox Publisher의 폴링 대상이다.
- `PROCESSING`
    - Outbox Publisher가 Kafka 발행을 위해 해당 이벤트를 선점한 상태
    - 동일 이벤트를 여러 Publisher가 동시에 발행하지 않도록 하기 위한 중간 상태이다.
- `PUBLISHED`
    - Kafka 발행에 성공한 상태
    - `processed_at`에 발행 성공 시각을 기록한다.
- `DEAD`
    - Kafka 발행 재시도 횟수를 초과했거나 수동 확인이 필요한 상태
    - `processed_at`에 최종 실패 처리 시각을 기록한다.

### Outbox Publisher 발행 정책

- Outbox Publisher는 `PENDING` 상태 이벤트를 `created_at` 오름차순으로 조회한다.
- 1차 구현 기준 폴링 주기는 1초로 설정한다.
- 1회 폴링 시 최대 100건까지 조회한다.
- 조회한 이벤트는 Kafka 발행 전 `PROCESSING` 상태로 선점한다.
- 선점에 성공한 이벤트만 Kafka로 발행한다.
- Kafka 발행 성공 시 이벤트 상태를 `PUBLISHED`로 변경하고 `processed_at`을 기록한다.
- Kafka 발행 실패 시 `retry_count`를 1 증가시키고 `last_error_message`에 실패 사유를 기록한다.
- 재시도 가능한 이벤트는 `PROCESSING`에서 `PENDING`으로 되돌려 다음 폴링 주기에 다시 발행한다.
- Kafka 발행은 최대 3회까지 재시도한다.
- `retry_count`가 최대 재시도 횟수인 3회를 초과하면 이벤트 상태를 `DEAD`로 변경하고 `processed_at`을 기록한다.
- `PUBLISHED` 또는 `DEAD` 상태의 오래된 이벤트는 추후 배치로 정리할 수 있다.

### Outbox Publisher 이벤트 단위 장애 격리 정책

- Outbox Publisher는 한 번의 폴링에서 여러 개의 `PENDING` 이벤트를 배치로 조회한다.
- 배치 내 특정 이벤트 1건 처리 중 예외가 발생하더라도 전체 배치 루프를 중단하지 않는다.
- 각 이벤트는 개별 try-catch 범위 안에서 처리한다.
- 하나의 이벤트 처리 실패는 로그로 기록하고, 이후 이벤트 처리는 계속 수행한다.
- 실패 로그에는 추적을 위해 `id`, `eventId`, `eventType`, `aggregateType`, `aggregateId`, `topic`을 포함한다.
- 이벤트별 실제 발행, 재시도, DEAD 전이 처리는 `OutboxEventPublishExecutor`가 담당한다.
- `OutboxEventPublisher`는 배치 루프의 장애 격리와 폴링 흐름 제어를 담당한다.

### 이벤트 중복 처리 정책

- OutboxEvent는 고유한 `event_id`를 가진다.
- Kafka 발행 과정에서 장애가 발생하면 동일 이벤트가 중복 발행될 수 있다.
- Consumer는 `eventId` 기준으로 멱등 처리를 고려해야 한다.
- 동일 `eventId` 이벤트가 중복 수신되더라도 후속 처리가 중복 수행되지 않도록 설계한다.

### OutboxEvent 중복 eventId 저장 처리 정책

- OutboxEvent는 `event_id`에 UNIQUE 제약을 둔다.
- 동일 `eventId` 저장 여부는 사전 조회(`findByEventId`)에만 의존하지 않는다.
- 중복 여부를 먼저 조회한 뒤 저장하는 check-then-act 구조는 동시 요청 상황에서 race condition이 발생할 수 있으므로 사용하지 않는다.
- OutboxEvent 저장은 DB UNIQUE 제약을 최종 방어 수단으로 사용한다.
- OutboxEvent 저장 시 `saveAndFlush()`를 사용하여 unique constraint 위반을 저장 메서드 내부에서 조기에 감지한다.
- `event_id` 중복으로 `DataIntegrityViolationException`이 발생하면 `CustomException(ErrorCode.OUTBOX_001)`로 변환한다.
- `OutboxEventService`는 중복 저장 예외를 `OUTBOX_001`로 변환하며, 결제 승인 흐름에서는 해당 예외를 포함한 Outbox 저장 실패 예외를 catch하여 외부로 전파하지 않는다.
- 결제 승인 흐름에서는 중복 OutboxEvent 저장 실패를 결제 성공의 필수 실패 조건으로 보지 않고, 로그로 기록한 뒤 결제 승인 흐름이 롤백되지 않도록 방어한다.
- 중복 저장 실패 로그에는 추적을 위해 `orderId`, `paymentId`, `eventId`를 포함한다.

### Outbox Pattern 적용 이유

- 결제 승인 트랜잭션 안에서 Kafka를 직접 발행하지 않음으로써 DB 저장과 Kafka 발행 사이의 정합성 문제를 줄인다.
- DB 트랜잭션이 롤백되면 OutboxEvent도 함께 롤백되므로 실제 결제 완료가 아닌 이벤트가 발행되는 문제를 방지한다.
- Kafka 발행에 실패하더라도 OutboxEvent가 DB에 남아 있으므로 재시도할 수 있다.
- 결제 승인 후 알림, 정산, 외부 연동 등 비동기 후속 작업을 안정적으로 확장할 수 있다.

### PAYMENT_APPROVED 이벤트 처리 범위

- `PAYMENT_APPROVED` 이벤트는 결제 승인 완료 후 후속 비동기 처리를 위한 이벤트이다.
- 현재 배송 생성은 결제 승인 트랜잭션 안에서 처리한다.
- 따라서 `PAYMENT_APPROVED` 이벤트는 배송 생성 자체가 아니라 알림, 정산, 외부 연동 등 후속 처리를 위해 사용한다.
- Consumer는 `eventId` 기준으로 멱등 처리해야 한다.

### PENDING → PROCESSING 선점 충돌 방지

- 다중 인스턴스 환경에서 동일 이벤트 중복 선점을 방지하기 위해 조건부 UPDATE를 사용한다.
- 쿼리 예시:

```
UPDATE outbox_event
SET status = 'PROCESSING', updated_at = NOW()
WHERE outbox_id = ? AND status = 'PENDING'
```

- UPDATE 결과 affected row가 1이면 선점 성공, 0이면 다른 Publisher가 이미 선점한 것으로 간주하고 건너뛴다.
- DB의 atomic update를 활용하므로 별도 분산 락이 필요하지 않다.

### Kafka 발행 실패 시 재시도 정책

- Kafka 발행 실패 시 retry_count를 1 증가시키고 last_error_message에 실패 사유를 기록한다.
- 실패한 이벤트는 status를 PROCESSING → PENDING으로 되돌려 다음 폴링 주기에 자동 재시도된다.
- 쿼리 예시:

```
UPDATE outbox_event
SET status = 'PENDING', retry_count = retry_count + 1, last_error_message = ?, updated_at = NOW()
WHERE outbox_id = ? AND status = 'PROCESSING';
```

- retry_count가 최대 재시도 횟수(3회)를 초과하면 status를 DEAD로 변경하고 processed_at에 최종 실패 시점을 기록한다.
- 쿼리 예시:

```
UPDATE outbox_event
SET status = 'DEAD', retry_count = retry_count + 1, last_error_message = ?, processed_at = NOW(), updated_at = NOW()
WHERE outbox_id = ? AND status = 'PROCESSING';
```

- 1차 구현에서는 즉시 재시도 방식을 사용한다. (지수 백오프는 추후 검토)

### processed_at 기록 시점

- processed_at은 PUBLISHED 또는 DEAD 상태로 진입할 때 한 번만 기록한다.
- PENDING / PROCESSING 상태에서는 NULL을 유지한다.
- 재시도 중인 이벤트는 processed_at이 NULL이므로 정리 배치 대상에서 제외된다.

### DEAD 이벤트 운영 정책

- DEAD 상태는 자동 재시도가 불가능한 최종 실패 상태이다.
- DEAD 이벤트는 운영 모니터링 대상이며 수동 확인이 필요하다.
- 수동 확인 후 다음 중 하나로 대응한다.
    1. 원인을 해결한 후 status를 PENDING으로 되돌려 재발행
    2. 더 이상 처리가 불필요한 경우 별도 보관 후 폐기
- DEAD 이벤트가 발생하면 로그를 기록하고, Slack 알림을 통해 운영자에게 통지한다.

### DEAD 이벤트 Slack 알림 정책

- OutboxEvent가 Kafka 발행 재시도 한도를 초과하여 `DEAD` 상태로 전이되면 Slack 팀 채널로 운영 알림을 전송한다.
- Slack 알림은 Kafka 발행 실패를 운영자가 빠르게 인지하고 수동 조치할 수 있도록 하기 위한 목적이다.
- Slack 알림 대상은 `OutboxEventPublishExecutor`에서 `DEAD` 상태 전이가 확정된 이벤트이다.
- 알림 메시지에는 다음 정보를 포함한다.
    - eventId
    - eventType
    - aggregateType
    - aggregateId
    - topic
    - partitionKey
    - retryCount
    - lastErrorMessage
    - processedAt
- Slack Webhook URL은 환경변수로 관리하며, GitHub에 직접 커밋하지 않는다.
- Slack 알림 전송 실패가 OutboxEvent 상태 저장에 영향을 주면 안 된다.
- 따라서 Slack 전송 실패 시 예외를 전파하지 않고 로그만 기록한다.
- Slack 알림 실패 여부와 관계없이 OutboxEvent의 `DEAD` 상태 전이는 유지된다.
- DEAD 이벤트는 자동 재시도 대상에서 제외되며, 운영자가 원인 확인 후 필요 시 `PENDING`으로 수동 복구할 수 있다.

### DEAD 이벤트 Slack 알림 안정성 정책

- Slack Webhook 호출은 외부 네트워크 I/O이므로 무한 대기하지 않도록 timeout을 설정한다.
- Slack Webhook 호출용 RestClient는 다음 timeout 정책을 적용한다.
    - connect timeout: 2초
    - read timeout: 3초
- OutboxEvent가 `DEAD` 상태로 전이되면 먼저 DB 트랜잭션 안에서 `DEAD` 상태와 `processedAt`, `lastErrorMessage`를 저장한다.
- Slack 알림은 OutboxEvent의 `DEAD` 상태 저장 트랜잭션이 커밋된 이후에 전송한다.
- Slack 응답 지연으로 인해 OutboxEvent 상태 저장 트랜잭션이나 DB 커넥션이 장시간 점유되지 않도록 한다.
- Slack 알림 전송 실패는 예외를 전파하지 않고 로그만 기록한다.
- Slack 알림 실패 여부와 관계없이 OutboxEvent의 `DEAD` 상태 전이는 유지된다.

### 오래된 이벤트 정리 정책

- PUBLISHED 또는 DEAD 상태이며 processed_at 기준으로 N일이 지난 이벤트는 정리 대상이다.
- 1차 구현에서는 자동 정리 배치를 포함하지 않고, 필요 시 수동 정리 또는 별도 배치로 처리한다.
- 향후 자동 정리를 도입할 경우 매일 새벽 시간대에 배치로 실행한다.
- 정리 시 데이터 보존이 필요한 경우 별도 아카이브 테이블로 이동 후 삭제한다.