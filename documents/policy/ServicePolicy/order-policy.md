## 주문

- 주문 유입 경로:
    - DIRECT(바로 구매)
    - CART(장바구니)
- 최소 주문 금액: 1,000원
- 미 승인 판매자 상품 주문 불가
- 주문 생성 시 클라이언트 요청 가격을 신뢰하지 않음
- 서버에서 product_item.price 기준으로 최종 금액 재계산
- 서버 계산 금액과 요청 금액 불일치 시 주문 거절
- 주문 생성 시 userCouponId는 선택값이다.
- userCouponId가 null이면 쿠폰을 사용하지 않는 주문으로 처리한다.
- 쿠폰을 사용하지 않는 경우 쿠폰 검증은 수행하지 않으며, 쿠폰 할인 금액은 0원으로 계산한다.
- userCouponId가 입력된 경우에만 본인 쿠폰 여부, AVAILABLE 상태, 유효 기간, 최소 주문금액을 검증하고 할인 금액을 계산한다.
- 재고 선점:
    - 비관적 락(SELECT FOR UPDATE)
    - item_id ASC 정렬로 데드락 방지
- 주문 취소 가능 배송 상태:
    - Delivery.status = ACCEPT
    - Delivery.status = INSTRUCT

      DEPARTURE 이후 상태에서는 주문 취소 불가

- 주문 취소 시:
    - 재고 복구
        - 재고 복구는 ProductItem 엔티티의 in-memory stock 값을 변경하는 방식이 아니라 DB 원자 update(stock = stock + qty)로 처리한다.
        - 이는 주문 생성에 의한 재고 차감과 주문 취소에 의한 재고 복구가 동시에 발생할 때 stock 덮어쓰기를 방지하기 위함이다.
    - 포인트 전액 복구
    - 쿠폰 복구
- 구매자 주문 취소 성공 시 상태 변경:
    - Order.status = CANCELLED
    - OrderItem.status = CANCELLED
    - Delivery.status = ORDER_CANCELLED
- 단, 결제 전 주문(PENDING_PAYMENT)은 Delivery가 아직 생성되지 않았을 수 있으므로 Delivery 상태 변경 대상이 없을 수 있음
- 주문 상태
    - PENDING_PAYMENT: 결제 대기
    - PAID: 결제 완료
    - CANCELLED: 주문 취소
- 주문 상태 흐름:
    - PENDING_PAYMENT → PAID
    - PENDING_PAYMENT → CANCELLED
        - 사용자 주문 취소 또는 향후 결제 대기 만료/실패 처리 확장 시 사용한다.
        - 현재 Mock 결제 승인 실패 롤백만으로는 자동 전환하지 않는다.
    - PAID → CANCELLED
- 취소/거절 처리 주체(actor_type):
    - BUYER
    - SELLER
    - ADMIN
    - SUPER_ADMIN
    - SYSTEM
- 취소/거절 유형(cancel_type):
    - BUYER_CANCEL
    - SELLER_REJECT
    - ADMIN_CANCEL
    - PAYMENT_FAILED
    - PAYMENT_CANCELLED
    - SYSTEM_CANCEL
- Order / Payment 상태 변경: 동일 트랜잭션 내 처리
- 최종 결제 금액 = 상품금액 - 쿠폰할인 - 사용포인트 - 구독할인 + 배송비
- 이미 CANCELLED 상태인 주문은 중복 취소 불가

### 주문 상태 변경 동시성 제어 정책 - 6.9(화) 추가

- 결제 승인 및 주문 취소처럼 `Order.status`를 변경하는 핵심 처리에서는 `Order`를 비관적 락으로 조회한다.
- 동일 주문에 대한 중복 결제 승인, 중복 주문 취소, 결제 승인과 주문 취소의 동시 처리 충돌을 방지한다.
- 락 획득 후 현재 주문 상태를 다시 검증한 뒤 상태 전이를 수행한다.
- 이미 `PAID` 상태인 주문은 재결제할 수 없다.
- 이미 `CANCELLED` 상태인 주문은 중복 취소할 수 없다.
- 주문 상태 전이와 `Payment` 상태 변경은 동일 트랜잭션 안에서 처리한다.
- 주문 취소 시 동일 주문의 중복 취소는 Order 비관적 락으로 방어한다.
- 단, 다른 주문 생성 트랜잭션과 같은 ProductItem 재고 변경이 동시에 발생할 수 있으므로, 주문 취소 재고 복구는 DB 원자 update(stock = stock + qty)로 처리한다.

### 주문 API 요청 제한 정책

- 주문 생성 API는 사용자당 1분 10회로 제한한다.
- 제한 초과 시 `429 Too Many Requests` 응답을 반환한다.
- 식별 기준은 `userId`이다.
- Redis Lua Script 기반 원자적 카운팅으로 처리한다.
- Redis 장애 시 Fail-Open 정책으로 요청을 차단하지 않는다.

### 주문 금액 계산 및 할인 한도 정책

- 주문 생성 시 클라이언트 요청 가격을 신뢰하지 않고 서버에서 상품 옵션 가격 기준으로 상품 금액을 재계산한다.
- 최종 결제 금액 계산식은 다음과 같다.

```
최종 결제 금액 = 상품금액 - 쿠폰할인 - 사용포인트 - 구독할인 + 배송비
```

- 쿠폰 할인, 사용 포인트, 구독 할인은 배송비를 제외한 상품 금액에만 적용한다.
- 쿠폰 할인 금액, 사용 포인트, 구독 할인 금액의 합은 상품 금액을 초과할 수 없다.

```
쿠폰할인 + 사용포인트 + 구독할인 <= 상품금액
```

- 위 한도를 초과하는 경우 주문 생성을 실패 처리한다.
- 최종 결제 금액이 음수가 되는 경우 주문 생성 정책 위반으로 보고 예외 처리한다.
- 최종 결제 금액을 `0원`으로 보정하지 않는다.
- 이는 잘못된 할인 조합이 조용히 통과되는 것을 방지하기 위한 정책이다.