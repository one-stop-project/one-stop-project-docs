# Order/Payment 도메인 설계문서

## 1. 도메인 목적

Order/Payment 도메인은 사용자의 주문 생성, 결제 승인, 결제 이력 관리, 후속 이벤트 처리를 담당한다.

커머스 서비스에서 주문과 결제는 금전과 직접 연결되므로, 중복 처리 방지와 데이터 정합성을 최우선으로 설계한다.

## 2. 주요 책임

| 책임 | 설명 |
|---|---|
| 주문 생성 | 장바구니/상품 기준 주문 생성 |
| 주문 상태 관리 | PENDING, PAID, CANCELED 등 상태 관리 |
| 결제 승인 | 외부 PG 승인 결과 검증 및 저장 |
| 중복 결제 방지 | 동일 주문에 대한 중복 결제 요청 차단 |
| 결제 이력 관리 | 결제 성공/실패/취소 기록 |
| 후속 이벤트 분리 | 포인트 적립, 알림, 장바구니 정리 등을 이벤트로 분리 |

## 3. 주요 흐름

### 3.1 주문 생성

```text
사용자 주문 요청
↓
상품/수량/가격 검증
↓
재고 또는 판매 가능 상태 확인
↓
주문 금액 계산
↓
Order PENDING 생성
↓
주문 응답 반환
```

### 3.2 결제 승인

```text
결제 승인 요청
↓
주문 조회
↓
주문 상태 검증
↓
동일 주문 결제 Lock 획득
↓
PG 결제 검증
↓
Payment 기록 저장
↓
Order 상태 PAID 변경
↓
Outbox 이벤트 저장
↓
후속 작업 비동기 처리
```

## 4. 데이터 모델

### Order

| 필드 | 설명 |
|---|---|
| id | 주문 식별자 |
| userId | 주문 사용자 |
| status | 주문 상태 |
| totalAmount | 총 주문 금액 |
| createdAt/updatedAt | 생성/수정 시각 |

### Payment

| 필드 | 설명 |
|---|---|
| id | 결제 식별자 |
| orderId | 연결 주문 |
| paymentKey | PG 결제 식별자 |
| amount | 결제 금액 |
| status | 결제 상태 |
| approvedAt | 승인 시각 |

### Outbox Event

| 필드 | 설명 |
|---|---|
| id | 이벤트 식별자 |
| aggregateType | Order/Payment 등 |
| aggregateId | 대상 ID |
| eventType | 이벤트 유형 |
| payload | 이벤트 데이터 |
| status | 처리 상태 |

## 5. 기술 선택 근거

### Redisson Lock

동일 주문에 대해 결제 승인 요청이 중복으로 들어오면 중복 결제가 발생할 수 있다. 이를 방지하기 위해 주문 단위 분산락을 사용한다.

```text
payment:order:{orderId}
```

Lock 범위는 결제 승인 검증과 주문/결제 상태 변경 구간으로 제한한다.

### Transactional Outbox

결제 성공 이후 알림 발송, 포인트 적립, 장바구니 정리 등을 같은 트랜잭션에 묶으면 외부 장애가 결제 처리에 영향을 줄 수 있다.

따라서 결제 성공 이벤트를 Outbox에 저장하고 후속 작업은 비동기로 처리한다.

### 멱등성

결제 승인 요청은 네트워크 재시도 또는 프론트 중복 클릭으로 여러 번 들어올 수 있다. 동일 주문/동일 paymentKey에 대해 이미 처리된 결제가 있으면 기존 결과를 반환하거나 중복 요청을 차단한다.

## 6. 정합성 고려사항

- 주문 금액과 PG 결제 금액이 일치해야 한다.
- 결제 성공 후 Order와 Payment 상태는 함께 변경되어야 한다.
- 동일 주문에 대한 중복 결제는 차단한다.
- 결제 성공 후 후속 작업 실패는 결제 트랜잭션을 rollback시키지 않는다.
- 외부 PG 성공 후 내부 저장 실패 시 보상 트랜잭션 또는 취소 정책이 필요하다.

## 7. 예외 처리

| 상황 | 처리 |
|---|---|
| 주문 없음 | ORDER_NOT_FOUND |
| 이미 결제된 주문 | PAYMENT_ALREADY_APPROVED |
| 결제 금액 불일치 | PAYMENT_AMOUNT_MISMATCH |
| PG 승인 실패 | PAYMENT_APPROVAL_FAILED |
| Lock 획득 실패 | PAYMENT_IN_PROGRESS |
| 후속 이벤트 처리 실패 | Outbox 재처리 대상 |

## 8. 한계 및 후속 개선

| 한계 | 후속 개선 |
|---|---|
| PG Webhook 멱등성 강화 필요 | webhook event table 및 unique key 적용 |
| Outbox 처리 모니터링 제한 | 실패 이벤트 재처리 UI/배치 추가 |
| 보상 트랜잭션 정책 단순 | 외부 성공/내부 실패 시 자동 취소 고도화 |
| 결제 Lock 범위 조정 필요 | Lock timeout/wait time 운영 지표 기반 튜닝 |
