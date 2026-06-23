# Coupon/Point 도메인 설계문서

## 1. 도메인 목적

Coupon/Point 도메인은 할인 쿠폰 발급과 포인트 적립/사용/만료를 담당한다.

두 도메인은 모두 금액 또는 혜택과 직접 연결되므로 동시성 제어와 데이터 정합성이 중요하다.

## 2. 주요 책임

| 도메인 | 책임 |
|---|---|
| Coupon | 선착순 쿠폰 발급, 중복 발급 방지, 재고 차감 |
| Point | 포인트 적립, 사용, 환불 복구, 만료 처리 |
| Batch | 포인트 만료 대상 처리 |
| History | 쿠폰/포인트 변경 이력 추적 |

## 3. Coupon 주요 흐름

### 3.1 선착순 쿠폰 발급

```text
쿠폰 발급 요청
↓
사용자별 중복 발급 확인
↓
쿠폰 재고 확인
↓
재고 차감
↓
발급 기록 저장
↓
결과 반환
```

이 흐름은 Redis Lua Script로 원자 처리한다.

## 4. Point 주요 흐름

### 4.1 포인트 적립

```text
주문/결제 성공 이벤트
↓
적립 정책 계산
↓
Point 잔액 증가
↓
PointHistory 적립 이력 저장
```

### 4.2 포인트 사용

```text
포인트 사용 요청
↓
사용 가능 잔액 확인
↓
만료일이 가까운 포인트부터 차감
↓
Point 잔액 감소
↓
PointHistory 사용 이력 저장
```

### 4.3 포인트 만료 배치

```text
만료 대상 조회
↓
remainingAmount > 0 확인
↓
만료 처리
↓
Point 잔액 차감
↓
PointHistory EXPIRE 저장
↓
대상 없을 때까지 반복
```

## 5. 데이터 모델

### Coupon

| 필드 | 설명 |
|---|---|
| id | 쿠폰 식별자 |
| name | 쿠폰명 |
| totalQuantity | 총 수량 |
| remainingQuantity | 잔여 수량 |
| startedAt/endedAt | 발급 가능 기간 |

### UserCoupon

| 필드 | 설명 |
|---|---|
| id | 발급 식별자 |
| couponId | 쿠폰 ID |
| userId | 사용자 ID |
| status | 사용 가능/사용 완료/만료 |

### Point

| 필드 | 설명 |
|---|---|
| id | 포인트 계정 ID |
| userId | 사용자 ID |
| balance | 현재 잔액 |
| version | 낙관적 락 버전 |

### PointHistory

| 필드 | 설명 |
|---|---|
| id | 이력 ID |
| userId | 사용자 ID |
| type | 충전/적립/사용/복구/만료 |
| amount | 변경 금액 |
| remainingAmount | 남은 금액 |
| expireAt | 만료일 |

## 6. 기술 선택 근거

### Redis Lua Script for Coupon

쿠폰 발급은 짧은 시간에 대량 요청이 몰린다. 중복 발급 확인과 재고 차감이 분리되면 Race Condition이 발생할 수 있다.

Redis Lua Script는 중복 확인, 재고 확인, 차감, 발급 기록을 원자적으로 처리할 수 있어 선착순 쿠폰 발급에 적합하다.

### Optimistic Lock for Point

포인트는 사용자별 잔액 데이터이며 동시에 여러 사용/적립 요청이 발생할 수 있다. DB row 단위 정합성이 필요하므로 `@Version` 기반 낙관적 락을 적용한다.

충돌 발생 시 재시도 정책을 통해 일시적인 동시성 충돌을 흡수한다.

### Cursor 기반 Batch

포인트 만료 배치에서 Offset Paging을 사용하면 처리 중 상태 변경으로 인해 데이터 누락이 발생할 수 있다. 따라서 `JpaCursorItemReader`로 만료 대상을 커서로 한 번에 흐르며 읽고, chunk(500) 단위로 처리한다. Writer는 Command의 금액을 신뢰하지 않고 최신 엔티티(Fresh Entity) 기준으로 만료 금액을 재계산한다.

## 7. 정합성 고려사항

- 쿠폰 중복 발급을 방지한다.
- 쿠폰 재고가 음수가 되면 안 된다.
- 포인트 사용 시 잔액 부족을 방지한다.
- 포인트 차감은 만료일 기준으로 처리한다.
- 포인트 만료 배치는 재실행해도 중복 만료되지 않아야 한다.
- 포인트 이력과 잔액 변경은 같은 트랜잭션으로 처리한다.

## 8. 예외 처리

| 상황 | 처리 |
|---|---|
| 쿠폰 재고 없음 | COUPON_SOLD_OUT |
| 쿠폰 중복 발급 | COUPON_ALREADY_ISSUED |
| 쿠폰 기간 아님 | COUPON_NOT_AVAILABLE |
| 포인트 부족 | POINT_NOT_ENOUGH |
| 포인트 동시성 충돌 | 재시도 또는 실패 응답 |
| 만료 대상 없음 | 배치 정상 종료 |

## 9. 한계 및 후속 개선

| 한계 | 후속 개선 |
|---|---|
| Redis 장애 시 쿠폰 정책 필요 | fail-closed/fail-open 정책 명확화 |
| 포인트 배치 모니터링 부족 | Batch Job 결과 대시보드 추가 |
| 포인트 동시성 재시도 한계 | 재시도 횟수/간격 운영 지표 기반 조정 |
| 쿠폰 발급 이벤트 후속 처리 제한 | 발급 이벤트 Outbox 적용 검토 |
