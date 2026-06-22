## 쿠폰

### 쿠폰 기본 정책

- 쿠폰은 BUYER 회원만 발급/사용할 수 있다.
- 쿠폰은 1주문에 1개만 사용할 수 있다.
- 동일 사용자는 동일 쿠폰을 중복 발급받을 수 없다.
- 쿠폰은 ACTIVE 상태이고 유효 기간 내에만 발급/사용할 수 있다.
- INACTIVE 쿠폰은 발급 및 사용이 불가능하다.

### **UserCoupon 상태**

- AVAILABLE: 사용 가능
- USED: 사용 완료
- EXPIRED: 만료

### 쿠폰 할인 정책

- 쿠폰 할인은 배송비를 제외한 상품 금액에만 적용한다.
- 최소 주문금액 검증 기준은 배송비 제외 상품 금액이다.
- 정액 쿠폰은 discountValue만큼 할인한다.
- 정액 쿠폰의 discountValue는 1원 이상이어야 한다.
- 정액 쿠폰 할인금액은 상품 금액을 초과할 수 없다.
- 정액 쿠폰은 maxDiscountPrice를 사용하지 않는다.
- 정률 쿠폰은 상품 금액 × discountValue / 100으로 계산한다.
- 정률 쿠폰의 discountValue는 1 이상 100 이하의 정수로 제한한다.
- 정률 쿠폰은 maxDiscountPrice를 필수로 가진다.
- 정률 쿠폰 할인금액은 maxDiscountPrice를 초과할 수 없다.
- 할인금액은 소수점 이하를 버림 처리한다.

### 쿠폰 발급 정책

- 발급 가능 쿠폰은 ACTIVE 상태이고 startAt <= 현재 시각 <= expiredAt인 쿠폰이다.
- 남은 발급 수량이 0 이하이면 발급할 수 없다.
- 선착순 수량 차감은 설정된 쿠폰 발급 전략에 따라 처리한다.
- 기본 발급 전략은 Redis `DECR` 기반 전략이다.
- Lua Script 전략과 Redis Lock 전략은 성능 비교 및 정합성 검증을 위한 비교 전략으로 관리한다.
- 중복 발급은 Redis Set과 DB Unique 제약으로 방지한다.
- 발급 성공 시 UserCoupon을 AVAILABLE 상태로 저장한다.

### 쿠폰 사용 정책

- 주문 생성 시 userCouponId를 입력받으면 쿠폰 사용 가능 여부를 검증하고 할인금액을 계산한다.
- userCouponId가 null이면 쿠폰 미사용 주문으로 처리하며, 쿠폰 할인금액은 0원이다.
- 주문 생성 시에는 쿠폰을 USED로 변경하지 않고, 주문에 userCouponId와 discountPrice만 저장한다.
- 주문 생성 단계에서는 쿠폰을 선점하지 않는다.
- 동일 쿠폰으로 결제 대기 주문이 여러 개 생성될 수 있으나, 실제 사용 처리는 결제 승인 시점에 최종 검증한다.
- 결제 승인 시 쿠폰이 더 이상 사용 가능하지 않으면 결제 승인을 실패 처리한다.
- 결제 승인 성공 시 쿠폰을 USED로 변경하고 usedAt, usedOrderId를 저장한다.
- 쿠폰 사용 처리는 결제 승인 트랜잭션 안에서 처리한다.

### 쿠폰 복구 정책

- 주문 취소 시 사용한 쿠폰을 복구한다.
- 쿠폰 만료 전이면 AVAILABLE로 복구한다.
- 쿠폰 만료 후이면 EXPIRED로 변경한다.
- 복구된 쿠폰 정보는 CancelOrderResponse.restoredCoupon에 포함한다.

### 쿠폰 기간 정책

- 이번 구현에서는 쿠폰 발급 가능 기간과 사용 가능 기간을 동일하게 관리한다.
- startAt은 쿠폰 발급/사용 시작일이다.
- expiredAt은 쿠폰 발급/사용 만료일이다.
- 쿠폰은 startAt <= 현재 시각 <= expiredAt인 경우에만 발급 및 사용 가능하다.
- 현재 시각이 expiredAt을 초과하면 쿠폰은 만료된 것으로 본다.

### 쿠폰/포인트/구독할인 계산 정책

- 구독 할인 금액을 계산한다.
- 쿠폰 할인금액을 계산한다.
- 사용 포인트 금액을 반영한다.
- 쿠폰 할인금액, 사용 포인트, 구독 할인금액의 합은 배송비를 제외한 상품 금액을 초과할 수 없다.
- 배송비에는 쿠폰 할인, 포인트 사용, 구독 할인을 적용하지 않는다.
- 최종 결제 금액 계산식은 다음과 같다.

```
최종 결제 금액 = 상품금액 - 쿠폰할인 - 사용포인트 - 구독할인 + 배송비
```

### 선착순 발급 정합성 정책

- 중복 발급은 Redis Set과 DB Unique 제약으로 이중 방어한다.
- Redis 기반 선착순 발급은 Redis 처리와 DB 트랜잭션이 서로 다른 저장소에서 수행되므로, DB 처리 실패 또는 트랜잭션 롤백 시 Redis 보상 처리를 반드시 수행한다.
- Redis 보상 처리는 다음 상황에서 수행한다.
    - Lua Script 또는 Redis DECR로 Redis 발급 처리가 성공한 뒤 DB 처리 중 예외가 발생한 경우
    - Redis 발급 처리가 성공했지만 DB 트랜잭션이 커밋되지 못하고 롤백된 경우
- 트랜잭션 롤백 보상은 `TransactionSynchronizationManager.registerSynchronization()`을 사용해 등록한다.
- `afterCompletion(STATUS_ROLLED_BACK)` 콜백에서 Redis 보상을 수행한다.
- 즉시 예외 보상과 롤백 콜백 보상이 중복 실행되지 않도록 `AtomicBoolean` 기반 보상 플래그를 사용한다.
- Redis 발급 처리 장애 또는 트랜잭션 롤백 보상 콜백 등록 실패 시 `COUPON_012` 에러를 사용한다.

### 선착순 발급 수량 증가 정합성 정책

- 쿠폰 발급 가능 여부는 발급 처리 초기에 `ACTIVE` 상태와 유효 기간, 잔여 수량 기준으로 1차 검증한다.
- 단, 발급 가능 여부 검증 이후 관리자에 의해 쿠폰이 비활성화될 수 있으므로, `Coupon.issuedQuantity` 증가 시점에 `ACTIVE` 상태와 잔여 수량을 DB UPDATE 조건으로 다시 검증한다.
- `issuedQuantity` 증가 UPDATE는 다음 조건을 모두 만족하는 경우에만 성공한다.
    - 쿠폰 ID가 일치한다.
    - 쿠폰 상태가 `ACTIVE`이다.
    - 현재 발급 수량이 전체 발급 수량보다 작다.
- `issuedQuantity` 증가 결과가 0건이면 발급 실패로 처리한다.
- 이때 쿠폰을 재조회하여 실패 원인을 구분한다.
    - 쿠폰 상태가 `ACTIVE`가 아니면 비활성 쿠폰 발급 시도로 보고 실패 처리한다.
    - 쿠폰 상태가 `ACTIVE`이면 잔여 수량 소진으로 보고 실패 처리한다.
- Redis 기반 발급 전략에서 Redis 수량 차감 또는 Lua Script 발급 처리가 이미 성공한 뒤 DB `issuedQuantity` 증가에 실패하면 Redis 보상 처리를 수행한다.

### DECR 발급 처리 흐름
1. Redis Set으로 동일 사용자의 중복 발급 여부를 1차 검증한다.
2. Redis stock key가 없으면 DB 기준 잔여 수량으로 초기화한다.
3. 트랜잭션 롤백 시 Redis 보상을 수행할 콜백을 등록한다.
4. Redis `DECR`로 남은 수량을 원자적으로 차감한다.
5. `DECR` 결과가 `null`이면 쿠폰 발급 처리 장애로 간주한다.
6. `DECR` 결과가 0보다 작으면 Redis 수량을 즉시 보상 증가하고 발급 실패 처리한다.
7. `DECR` 성공 이후부터 DB 처리 실패 또는 트랜잭션 롤백 시 Redis 수량 보상 대상이 된다.
8. DB Unique 제약 및 UserCoupon 존재 여부로 중복 발급을 최종 방어한다.
9. UserCoupon을 AVAILABLE 상태로 저장한다.
10. Coupon issuedQuantity를 증가시킨다. 이때 DB UPDATE 조건으로 쿠폰 `ACTIVE` 상태와 잔여 수량을 함께 검증한다.
11. Redis issued-users Set에 사용자 ID를 기록한다.
12. DB 처리 중 예외가 발생하면 Redis 수량을 복구하고, 이번 요청에서 issued-users Set에 기록된 경우 사용자 ID를 제거한다.
13. DB 트랜잭션이 롤백되면 `afterCompletion(STATUS_ROLLED_BACK)` 콜백에서 동일한 Redis 보상을 수행한다.

### Lua Script 발급 처리 흐름

1. Redis stock key가 없으면 DB 기준 잔여 수량으로 초기화한다.
2. 트랜잭션 롤백 시 Redis 보상을 수행할 콜백을 등록한다.
3. Lua Script 내부에서 중복 발급 확인, 수량 차감, issued-users Set 기록, TTL 설정을 원자적으로 처리한다.
4. Lua Script 결과가 실패 값이면 DB 저장을 수행하지 않고 예외 처리한다.
    - `null`: 쿠폰 발급 처리 장애
    - `-1`: 수량 소진
    - `-2`: 이미 발급된 사용자
    - `-3`: Redis stock key 미초기화
5. Lua Script 성공 이후부터 DB 처리 실패 또는 트랜잭션 롤백 시 Redis 보상 대상이 된다.
6. DB Unique 제약 및 UserCoupon 존재 여부로 중복 발급을 최종 방어한다.
7. UserCoupon을 AVAILABLE 상태로 저장한다.
8. Coupon issuedQuantity를 증가시킨다. 이때 DB UPDATE 조건으로 쿠폰 `ACTIVE` 상태와 잔여 수량을 함께 검증한다.
9. DB 처리 중 예외가 발생하면 보상 Lua Script를 실행하여 Redis stock을 복구하고 issued-users Set에서 사용자 ID를 제거한다.
10. DB 트랜잭션이 롤백되면 `afterCompletion(STATUS_ROLLED_BACK)` 콜백에서 동일한 Redis 보상을 수행한다.

### 쿠폰 API 요청 제한 정책

- 쿠폰 발급 API는 사용자당 1분 10회로 제한한다.
- 제한 초과 시 `429 Too Many Requests` 응답을 반환한다.
- 식별 기준은 `userId`이다.
- Redis Lua Script 기반 원자적 카운팅으로 처리한다.
- Redis 장애 시 Fail-Open 정책으로 요청을 차단하지 않는다.

### 쿠폰 상태 전이

- AVAILABLE 상태의 사용자 쿠폰만 주문 생성 시 사용할 수 있다.
- 결제 승인 성공 시 UserCoupon은 AVAILABLE → USED로 변경한다.
- 주문 취소 시 쿠폰 만료 전이면 UserCoupon은 USED → AVAILABLE로 복구한다.
- 주문 취소 시 쿠폰 만료 후이면 UserCoupon은 USED → EXPIRED로 변경한다.
- 만료 시간이 지난 AVAILABLE UserCoupon은 만료 스케줄러를 통해 EXPIRED로 변경한다.
- 만료 시간이 지난 ACTIVE Coupon은 만료 스케줄러를 통해 INACTIVE로 변경한다.
- 단, 스케줄러가 아직 실행되지 않았더라도 조회 또는 사용 검증 시 expiredAt을 초과한 쿠폰은 만료된 것으로 판단한다.
- 이 경우 해당 UserCoupon 상태를 EXPIRED로 변경할 수 있다.

### 쿠폰 만료 스케줄러 정책

- 쿠폰 만료 처리는 매일 새벽 1시 `Asia/Seoul` 기준으로 스케줄러가 수행한다.
- 스케줄러는 `app.scheduler.enabled=true` 설정인 경우에만 활성화한다.
- 만료 처리 기준 시각은 스케줄러 실행 시점의 KST 현재 시각이다.
- 만료 기준은 `expiredAt < now`이다.
- `expiredAt == now`인 경우는 아직 유효한 것으로 간주한다.
- 만료 시간이 지난 쿠폰에 속한 AVAILABLE UserCoupon은 EXPIRED로 일괄 변경한다.
- 만료 시간이 지난 ACTIVE Coupon은 INACTIVE로 일괄 변경한다.
- UserCoupon 만료 처리를 먼저 수행한 뒤 Coupon 비활성화 처리를 수행한다.
- Coupon.status와 관계없이 expiredAt이 지난 쿠폰에 속한 AVAILABLE UserCoupon은 EXPIRED로 정리할 수 있다.
    - 이는 Coupon이 이미 INACTIVE 상태가 되었지만 UserCoupon이 AVAILABLE로 남아 있는 데이터도 보정하기 위함이다.
- 대량 데이터 처리를 고려해 개별 엔티티 조회 방식이 아닌 JPQL bulk update 방식으로 처리한다.
- 스케줄러 실행 중 예외가 발생해도 애플리케이션으로 예외를 전파하지 않고 로그만 기록한다.

### 주문 생성 시 쿠폰 검증 우선순위

1. 본인 쿠폰 여부 (userCoupon.userId == 요청 userId)
2. AVAILABLE 상태 여부
3. 유효 기간 여부 (startAt <= 현재 시각 <= expiredAt)
4. 최소 주문금액 충족 여부
5. 쿠폰 할인금액 계산

검증을 통과한 경우에만 주문에 userCouponId와 discountPrice를 저장한다.

### 쿠폰 사용 동시성 제어 정책

- 주문 생성 시에는 쿠폰을 선점하지 않고, 주문에 `userCouponId`와 `discountPrice`만 저장한다.
- 동일 `UserCoupon`으로 여러 결제 대기 주문이 생성될 수 있으나, 실제 쿠폰 사용 처리는 결제 승인 시점에 최종 검증한다.
- 결제 승인 시 쿠폰 사용 처리는 `UserCoupon`을 비관적 락으로 재조회한 뒤 수행한다.
- 동일 `UserCoupon`이 서로 다른 주문에서 동시에 결제 승인되는 경우, 먼저 락을 획득한 트랜잭션만 쿠폰을 사용할 수 있다.
- 락 획득 후 `AVAILABLE` 상태, 유효 기간, 소유자 여부를 다시 검증한 뒤 `USED`로 변경한다.
- 이미 `USED` 또는 `EXPIRED` 상태인 쿠폰이면 결제 승인을 실패 처리한다.

### 쿠폰 복구 동시성 제어 정책

- 주문 취소 시 쿠폰 복구는 `UserCoupon`을 비관적 락으로 재조회한 뒤 수행한다.
- 쿠폰 사용/복구가 동시에 처리되는 상황에서 `UserCoupon` 상태 변경 충돌을 방지한다.
- 쿠폰 만료 전이면 `USED → AVAILABLE`로 복구한다.
- 쿠폰 만료 후이면 `USED → EXPIRED`로 변경한다.

### 쿠폰 상태 변경 락 정책

- `AVAILABLE → USED`, `USED → AVAILABLE`, `USED → EXPIRED` 상태 변경은 `UserCoupon` 비관적 락 획득 후 처리한다.
- 주문 생성 단계의 쿠폰 검증은 사전 검증이며, 최종 정합성은 결제 승인 시점의 비관적 락 기반 검증으로 보장한다.

### 선착순 쿠폰 발급 전략 정책

- 선착순 쿠폰 발급 로직은 전략 패턴 기반으로 분리하여 관리한다.
- 쿠폰 발급 전략은 설정값을 통해 선택할 수 있다.

```yaml
coupon:
  issue:
    strategy: decr # decr | lua | lock
```

- 기본 발급 전략은 `decr`이다.
- 각 전략의 목적은 다음과 같다.
    - `decr`: Redis DECR 기반 원자적 수량 차감 전략
    - `lua`: Lua Script 기반 Redis 원자 처리 전략
    - `lock`: Redis Lock 기반 순차 처리 전략
- 현재 운영 기본 전략은 `decr`로 유지한다.
- `lua`, `lock` 전략은 성능 비교 및 정합성 검증을 위한 비교 전략으로 관리한다.
- 최종 적용 전략은 부하 테스트 결과, 정합성 검증 결과, 구현 복잡도, 장애 대응 가능성을 종합하여 결정한다.

### DECR 전략

- Redis `DECR` 명령을 사용하여 쿠폰 잔여 수량을 원자적으로 차감한다.
- Redis Set으로 동일 사용자의 중복 발급 여부를 1차 검증한다.
- DB Unique 제약으로 동일 사용자 중복 발급을 최종 방어한다.
- Redis `DECR` 성공 이후 DB 처리 실패 또는 트랜잭션 롤백이 발생하면 Redis 수량을 보상 증가한다.
- 이번 요청에서 Redis issued-users Set에 발급 사용자 기록이 추가된 경우에는 보상 시 사용자 ID를 제거한다.
- 트랜잭션 롤백 시 `afterCompletion(STATUS_ROLLED_BACK)` 콜백에서 Redis 보상을 수행한다.
- 즉시 보상과 롤백 콜백 보상이 중복 실행되지 않도록 보상 플래그를 사용한다.
- 처리 속도가 빠르고 구현이 단순하다는 장점이 있다.
- Redis 처리와 DB 저장 사이의 정합성 보상을 반드시 고려해야 한다.

### DECR 전략 Redis stock key TTL Race 방어 정책

- Redis DECR 기반 쿠폰 발급 전략에서는 `coupon:stock:{couponId}` key를 사용해 쿠폰 잔여 수량을 관리한다.
- `coupon:stock:{couponId}` key는 쿠폰 만료 시각 기준 TTL을 가진다.
- stock key가 TTL 만료로 사라진 직후 `DECR`이 실행되면 Redis는 존재하지 않는 key를 `0`으로 간주하고 `-1` 값을 가진 새 key를 생성할 수 있다.
- 이때 새로 생성된 key에는 TTL이 없으므로, 이후 `setIfAbsent` 기반 초기화가 실패하고 쿠폰이 영구 소진 상태처럼 남을 수 있다.
- 이를 방지하기 위해 DECR 전략에서는 단순 `DECR` 명령을 직접 호출하지 않는다.
- Lua Script를 사용하여 다음 조건을 원자적으로 확인한 뒤 DECR을 수행한다.
    1. stock key가 존재하는지 확인한다.
    2. stock key에 TTL이 설정되어 있는지 확인한다.
    3. 위 조건을 만족하는 경우에만 DECR을 수행한다.
- Lua Script 결과가 `stock key 없음` 또는 `TTL 없는 비정상 key`인 경우, 기존 stock key를 삭제한 뒤 DB 기준 잔여 수량과 쿠폰 만료 시각 기준 TTL로 stock key를 재초기화한다.
- 재초기화 후 안전 DECR을 1회만 재시도한다.
- 재시도 후에도 stock key가 없거나 TTL이 없는 경우 쿠폰 발급 처리 장애로 판단하고 실패 처리한다.
- TTL 없는 `0` 또는 음수 stock key는 정상 쿠폰 재고 key로 보지 않으며, 삭제 후 DB 기준으로 복구한다.
- DB 기준 잔여 수량은 `totalQuantity - issuedQuantity`로 계산한다.
- 재초기화 시점에도 쿠폰이 `ACTIVE` 상태이고 유효 기간 내에 있는지 다시 검증한다.

### Lua Script 전략

- Redis Lua Script 안에서 중복 발급 확인, 수량 차감, 발급 사용자 Set 기록, TTL 설정을 하나의 원자적 작업으로 처리한다.
- Redis 내부 작업을 하나의 Script로 묶기 때문에 Redis 명령 간 race condition을 줄일 수 있다.
- Lua Script 성공 이후 DB 처리 실패 또는 트랜잭션 롤백이 발생하면 보상 Lua Script를 실행한다.
- 보상 Lua Script는 Redis stock을 복구하고 issued-users Set에서 사용자 ID를 제거한다.
- 트랜잭션 롤백 시 `afterCompletion(STATUS_ROLLED_BACK)` 콜백에서 Redis 보상을 수행한다.
- 즉시 보상과 롤백 콜백 보상이 중복 실행되지 않도록 보상 플래그를 사용한다.
- Redis 내부 정합성은 강하지만, DB 저장까지 포함한 전체 트랜잭션 원자성은 별도 보상 처리가 필요하다.
- Lua Script 관리 비용과 디버깅 난이도를 고려한다.

#### Redis Lock 전략

- 쿠폰 단위 Redis Lock을 획득한 뒤 발급 가능 여부, 중복 발급 여부, 수량 증가를 처리한다.
- 동일 쿠폰에 대한 발급 요청을 순차 처리하여 이해하기 쉬운 동시성 제어 구조를 제공한다.
- DB의 issuedQuantity를 기준으로 발급 수량을 관리한다.
- Lock 획득 실패 시 발급을 실패 처리한다.
- 예외 발생 여부와 관계없이 현재 스레드가 Lock을 보유한 경우 반드시 Lock을 해제한다.
- Lock 대기 시간과 Lock 점유 시간으로 인해 처리량이 낮아질 수 있으므로 성능 비교가 필요하다.
- 트랜잭션 커밋 시점과 Lock 해제 시점의 범위는 후속 검토 대상이다.
- DB의 issuedQuantity 증가 시점에는 쿠폰 `ACTIVE` 상태와 잔여 수량을 함께 검증하여, 발급 가능 검증 이후 비활성화된 쿠폰의 발급 수량이 증가하지 않도록 방어한다.

### Lock 전략 Redis unlock 시점 정책

- Redis Lock 기반 쿠폰 발급 전략에서는 동일 쿠폰에 대한 동시 발급 요청을 직렬화하기 위해 쿠폰 단위 Redis Lock을 사용한다.
- `@Transactional` 메서드 내부의 `finally`에서 Redis Lock을 해제하면, DB 트랜잭션 커밋보다 먼저 unlock이 실행될 수 있다.
- 이 경우 다른 요청이 unlock 직후 락을 획득하고 아직 커밋되지 않은 DB 상태를 조회할 수 있다.
- DB UNIQUE 제약이 최종 방어 수단으로 동작하므로 데이터 무결성은 유지된다.
- 다만 unlock과 commit 사이의 짧은 구간에서 일시적으로 과발급 가능 상태처럼 판단될 수 있는 윈도우가 존재한다.
- 이를 방지하기 위해 Redis Lock 해제는 트랜잭션 종료 이후로 지연한다.
- Lock 해제는 `TransactionSynchronization.afterCompletion()` 콜백에서 수행한다.
- `afterCompletion()`은 트랜잭션 commit 또는 rollback 완료 이후 실행되므로, 다른 요청은 이전 트랜잭션 종료 후에만 락을 획득할 수 있다.
- 트랜잭션 동기화 콜백 등록에 실패한 경우, Redis Lock을 즉시 해제하고 쿠폰 발급 처리 장애로 실패 처리한다.
- unlock 실패는 운영 로그로 기록한다.
- 단, Lock lease time이 트랜잭션 처리 시간보다 짧으면 Redisson의 자동 만료로 인해 Lock이 먼저 해제될 수 있으므로, lease time은 트랜잭션 예상 처리 시간을 고려해 설정한다.

#### 전략 선택 기준

- 선착순 쿠폰 발급 전략은 단순 응답 속도만으로 결정하지 않는다.
- 다음 기준을 함께 고려한다.
    - 평균 응답 시간
    - p95 / p99 응답 시간
    - TPS
    - 성공 발급 수
    - 중복 발급 발생 여부
    - 쿠폰 수량 초과 발급 발생 여부
    - Redis stock과 DB issuedQuantity 정합성
    - DB UserCoupon 수와 Redis issued-users Set 크기 일치 여부
    - 장애 또는 예외 발생 시 보상 처리 가능 여부
    - 구현 복잡도와 유지보수성

#### 성능 비교 계획

- `decr`, `lua`, `lock` 세 전략은 동일한 조건에서 성능 비교한다.
- 같은 Redis 인스턴스, 같은 DB, 같은 JVM 옵션, 같은 쿠폰 수량, 같은 요청 수, 같은 사용자 풀을 사용한다.
- 테스트 전 Redis와 DB를 초기화하여 깨끗한 상태에서 시작한다.
- 워밍업 실행 후 본 측정을 진행한다.
- ExecutorService + CountDownLatch 또는 JMeter/k6를 사용하여 동시 요청을 발생시킨다.
- 서버 1개 환경과 서버 2개 환경을 나누어 비교할 수 있다.
- 성능 비교 결과는 후속 이슈에서 정리하며, 최종 전략 선택 근거로 사용한다.