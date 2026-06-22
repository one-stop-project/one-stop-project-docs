# Subscription 도메인 설계문서

## 1. 도메인 목적

Subscription 도메인은 사용자의 구독 상품 가입, 정기결제, 구독 상태 전이, 결제 실패 복구, 구독 혜택 적용 기준을 담당한다.

일반 주문/결제와 달리 구독은 시간의 흐름에 따라 상태가 계속 바뀌므로 상태 머신 기반으로 설계한다.

## 2. 주요 책임

| 책임 | 설명 |
|---|---|
| 구독 가입 | 사용자가 구독 상품을 신청 |
| 구독 상태 관리 | TRIALING, ACTIVE, PAST_DUE, CANCELED, ENDED |
| 정기결제 처리 | 결제 예정일 기준 자동 결제 |
| Billing 이력 관리 | 결제 시도, 성공/실패, 실패 사유 기록 |
| 결제 실패 복구 | 재시도 또는 유예 상태 관리 |
| 혜택 적용 기준 | 구독 상태에 따른 혜택 제공 여부 판단 |

## 3. 상태 모델

| 상태 | 의미 |
|---|---|
| TRIALING | 체험 또는 초기 구독 상태 |
| ACTIVE | 정상 구독 상태 |
| PAST_DUE | 결제 실패 후 유예 상태 |
| CANCELED | 사용자 또는 정책에 의해 취소된 상태 |
| ENDED | 구독 기간 종료 상태 |

상태 전이 예시:

```text
TRIALING → ACTIVE
ACTIVE → PAST_DUE
PAST_DUE → ACTIVE
PAST_DUE → CANCELED
ACTIVE → CANCELED
CANCELED → ENDED
```

## 4. 주요 흐름

### 4.1 구독 가입

```text
구독 신청
↓
구독 상품 조회
↓
사용자 기존 구독 여부 확인
↓
초기 결제 또는 빌링키 등록
↓
Subscription 생성
↓
상태 TRIALING 또는 ACTIVE 설정
```

### 4.2 정기결제 배치

```text
결제 예정일 도래 구독 조회
↓
구독 상태 확인
↓
중복 처리 방지
↓
SubscriptionBilling 결제 시도 이력 생성
↓
PG 빌링키 결제 요청
↓
성공 시 ACTIVE 유지 및 다음 결제일 갱신
↓
실패 시 PAST_DUE 전이 및 실패 사유 기록
```

### 4.3 구독 취소

```text
사용자 취소 요청
↓
구독 상태 확인
↓
취소 가능 여부 검증
↓
CANCELED 상태 전이
↓
혜택 종료 기준 반영
```

## 5. 데이터 모델

### Subscription

| 필드 | 설명 |
|---|---|
| id | 구독 식별자 |
| userId | 구독 사용자 |
| planId | 구독 상품 |
| status | 현재 구독 상태 |
| startedAt | 시작 시각 |
| endedAt | 종료 시각 |
| nextBillingAt | 다음 결제 예정일 |
| canceledAt | 취소 시각 |

### SubscriptionBilling

| 필드 | 설명 |
|---|---|
| id | 결제 시도 식별자 |
| subscriptionId | 구독 ID |
| amount | 결제 금액 |
| status | 결제 성공/실패 상태 |
| pgTransactionId | PG 거래 ID |
| failureReason | 실패 사유 |
| retryCount | 재시도 횟수 |
| requestedAt/respondedAt | 요청/응답 시각 |

## 6. 기술 선택 근거

### 상태 머신

구독은 단순 boolean 값으로 표현하기 어렵다. 결제 실패, 유예, 취소, 종료 등 상태별 정책이 다르므로 명시적인 상태 머신으로 관리한다.

### Billing 이력 분리

구독의 현재 상태와 결제 시도 이력은 다른 개념이다. 결제 이력을 별도 테이블로 분리해 실패 사유와 재시도 이력을 추적한다.

### Scheduler/Batch

정기결제는 사용자 요청이 아니라 시간 기반으로 발생하므로 스케줄러 또는 배치가 적합하다. 결제 예정일이 도래한 구독을 조회하고 순차적으로 처리한다.

## 7. 정합성 고려사항

- 동일 구독에 대해 같은 회차 결제가 중복 발생하면 안 된다.
- Billing 이력은 결제 시도마다 남긴다.
- 결제 성공 후 다음 결제 예정일이 갱신되어야 한다.
- 결제 실패 시 구독 상태와 실패 이력이 함께 기록되어야 한다.
- 배치 재실행 시 이미 처리된 Billing은 중복 처리하지 않아야 한다.

## 8. 예외 처리

| 상황 | 처리 |
|---|---|
| 이미 활성 구독 존재 | 중복 구독 제한 |
| 결제 실패 | PAST_DUE 전이 및 실패 이력 저장 |
| 재시도 초과 | CANCELED 또는 ENDED 전이 |
| 구독 없음 | SUBSCRIPTION_NOT_FOUND |
| 취소 불가 상태 | INVALID_SUBSCRIPTION_STATUS |

## 9. 한계 및 후속 개선

| 한계 | 후속 개선 |
|---|---|
| 재시도 정책 단순 | 실패 사유별 backoff 정책 적용 |
| 배치 모니터링 제한 | 구독 결제 배치 대시보드 추가 |
| 상태 전이 테스트 부족 가능 | 상태 전이 테스트 매트릭스 강화 |
| PG 장애 시 대응 정책 필요 | 보상/재시도/알림 정책 고도화 |
