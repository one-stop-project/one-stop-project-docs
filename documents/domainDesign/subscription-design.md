# Subscription 도메인 설계문서

## 1. 도메인 목적

Subscription 도메인은 사용자의 구독 상품 가입, 정기결제, 구독 상태 전이, 결제 실패 복구, 구독 혜택 적용 기준을 담당한다.

일반 주문/결제와 달리 구독은 시간의 흐름에 따라 상태가 계속 바뀌므로 상태 머신 기반으로 설계한다.

> ⚠️ **현재 구현 vs 목표 설계**
> 현재 구현은 단일 `Subscription` 엔티티 + 3-상태(`ACTIVE` / `CANCELLED` / `EXPIRED`), Mock 자동 결제(매일 자정 스케줄러) 기준이다.
> 아래 문서의 `SubscriptionBilling`, `planId`, `billingKey`/PG 빌링키 연동, `TRIALING`/`PAST_DUE` 상태, 재시도·유예 정책은 **현재 미구현 (목표 설계)**이며 각 항목에 라벨로 구분한다.

## 2. 주요 책임

| 책임 | 설명 |
|---|---|
| 구독 가입 | 사용자가 구독 상품을 신청 |
| 구독 상태 관리 | 구현: `ACTIVE`, `CANCELLED`, `EXPIRED` / ⚠️ 목표(미구현): `TRIALING`·`PAST_DUE` 포함 5-상태 |
| 정기결제 처리 | 결제 예정일 기준 자동 결제 (현재 Mock) |
| Billing 이력 관리 | ⚠️ 현재 미구현 (목표 설계) — 결제 시도, 성공/실패, 실패 사유 기록 |
| 결제 실패 복구 | ⚠️ 현재 미구현 (목표 설계) — 재시도/유예 상태 관리 (현재는 실패 시 즉시 EXPIRED) |
| 혜택 적용 기준 | 구독 상태에 따른 혜택 제공 여부 판단 |

## 3. 상태 모델

### 3.1 현재 구현 (ACTIVE / CANCELLED / EXPIRED)

| 상태 | 의미 |
|---|---|
| ACTIVE | 정상 구독 상태 (자동 결제 대상) |
| CANCELLED | 사용자 해지 상태 — endAt까지 혜택 유지, 다음 자동 결제 중단 |
| EXPIRED | 만료 상태 — 혜택 종료, 재가입 가능 |

상태 전이:

```text
ACTIVE → CANCELLED → EXPIRED
ACTIVE → (자동결제 실패) → EXPIRED
```

### 3.2 ⚠️ 현재 미구현 (목표 설계) — 5-상태 모델

> 아래 `TRIALING`/`PAST_DUE` 포함 5-상태 모델과 전이는 PG 빌링키 연동을 전제한 목표 설계이며 현재 코드에는 없다.

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

현재 구현은 유저를 비관적 락으로 조회 → 기존 유효 구독(ACTIVE/CANCELLED) 존재 여부 확인 → `Subscription`을 `ACTIVE`로 생성한다. `startAt = now`, `endAt = nextPaymentDate = now + 30일`이며 별도 초기 결제 단계는 없다(Mock).

```text
구독 신청
↓
사용자 비관적 락 조회
↓
기존 유효 구독(ACTIVE/CANCELLED) 여부 확인
↓
Subscription 생성 (status=ACTIVE, endAt/nextPaymentDate = now+30일)
```

> ⚠️ `초기 결제`·`빌링키 등록`·`TRIALING` 진입은 현재 미구현 (목표 설계).

### 4.2 정기결제 배치

현재 구현은 매일 자정 스케줄러가 `ACTIVE` + `nextPaymentDate <= now`인 구독을 chunk(1000건) 단위로 조회해 Mock 결제 후 성공 시 `renew(now)`(endAt·nextPaymentDate를 `now + 30일`로 갱신, 상태 ACTIVE 유지), 실패 시 `expire()`(EXPIRED)로 처리한다.

```text
nextPaymentDate <= now 인 ACTIVE 구독 조회 (chunk)
↓
Mock 결제 시도
↓
성공 시 renew(now): endAt·nextPaymentDate = now + 30일, ACTIVE 유지
↓
실패 시 expire(): 즉시 EXPIRED
```

> ⚠️ 아래는 목표 설계 — 현재 미구현. `SubscriptionBilling` 결제 시도 이력 생성, PG 빌링키 결제 요청, 실패 시 `PAST_DUE` 유예 전이 및 실패 사유 기록.
>
> ```text
> SubscriptionBilling 결제 시도 이력 생성
> ↓
> PG 빌링키 결제 요청
> ↓
> 실패 시 PAST_DUE 전이 및 실패 사유 기록
> ```

### 4.3 구독 취소

```text
사용자 취소 요청 (reason 필수)
↓
구독 상태 확인 (이미 CANCELLED → SUBSCRIPTION_003, EXPIRED → SUBSCRIPTION_011)
↓
CANCELLED 상태 전이 + cancelReason 저장
↓
endAt까지 혜택 유지
```

## 5. 데이터 모델

### Subscription (현재 구현)

| 필드 | 설명 |
|---|---|
| id | 구독 식별자 (subscription_id) |
| user (user_id) | 구독 사용자 (ManyToOne) |
| status | 현재 구독 상태 (ACTIVE/CANCELLED/EXPIRED) |
| startAt | 시작 시각 |
| endAt | 만료 시각 (혜택 유지 기준) |
| nextPaymentDate | 다음 자동 결제 예정일 |
| cancelReason | 해지 사유 |

> BaseEntity 상속으로 createdAt/updatedAt 보유. `(user_id, status, created_at)` 복합 인덱스(`idx_sub_user`).
> ⚠️ `planId`(구독 상품 구분)는 현재 미구현 — 단일 멤버십만 존재.

### SubscriptionBilling — ⚠️ 현재 미구현 (목표 설계)

> 결제 시도 이력 테이블은 PG 빌링키 연동 시점의 목표 설계이며 현재 코드에는 없다.

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

### Billing 이력 분리 (⚠️ 현재 미구현 — 목표 설계)

구독의 현재 상태와 결제 시도 이력은 다른 개념이다. 결제 이력을 별도 테이블로 분리해 실패 사유와 재시도 이력을 추적한다. (현재 구현은 Mock 결제로 별도 이력 테이블이 없다.)

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
| 이미 유효한 구독 존재 (ACTIVE/CANCELLED) | `SUBSCRIPTION_002` 중복 구독 제한 |
| 자동 결제 실패 | 현재: 즉시 `EXPIRED` / ⚠️ 목표(미구현): `PAST_DUE` 전이 및 실패 이력 저장 |
| 재시도 초과 | ⚠️ 현재 미구현 (목표 설계) — CANCELED 또는 ENDED 전이 |
| 구독 없음 | `SUBSCRIPTION_001` |
| 이미 해지된 구독 해지 시도 | `SUBSCRIPTION_003` |
| 이미 만료된 구독 해지 시도 | `SUBSCRIPTION_011` |
| ACTIVE 아님에서 renew 시도 | `SUBSCRIPTION_004` |

## 9. 한계 및 후속 개선

| 한계 | 후속 개선 |
|---|---|
| 재시도 정책 단순 | 실패 사유별 backoff 정책 적용 |
| 배치 모니터링 제한 | 구독 결제 배치 대시보드 추가 |
| 상태 전이 테스트 부족 가능 | 상태 전이 테스트 매트릭스 강화 |
| PG 장애 시 대응 정책 필요 | 보상/재시도/알림 정책 고도화 |
