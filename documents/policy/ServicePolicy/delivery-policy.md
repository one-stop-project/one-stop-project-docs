## 배송

- 배송 상태:
    - ACCEPT (배송 접수)
    - INSTRUCT (상품 준비중)
    - DEPARTURE (배송 출발 = 배송지시)
    - DELIVERING (배송중)
    - FINAL_DELIVERY (배송 완료)
    - ORDER_CANCELLED (주문 취소)
- 배송 상태 흐름:
    - ACCEPT → INSTRUCT → DEPARTURE → DELIVERING → FINAL_DELIVERY
    - ACCEPT → ORDER_CANCELLED (판매자 거절 / 구매자 취소)
    - INSTRUCT → ORDER_CANCELLED (구매자 취소)
    - DEPARTURE 이후에는 ORDER_CANCELLED 전환 불가
- 주문 상품(order_item) 상태:
    - PENDING_PAYMENT(결제 대기)
    - ORDERED (주문 접수)
    - CONFIRMED (판매자 확인 완료)
    - SHIPPING (배송 중)
    - DELIVERED (배송 완료)
    - CANCELLED (주문 취소)
    - REJECTED (주문 거절)
- 주문 상품 상태 흐름:
    - PENDING_PAYMENT → ORDERED → CONFIRMED → SHIPPING → DELIVERED
    - PENDING_PAYMENT → CANCELLED
    - PENDING_PAYMENT → ORDERED → CANCELLED
    - PENDING_PAYMENT → ORDERED → CONFIRMED → CANCELLED
    - PENDING_PAYMENT → ORDERED → REJECTED
- 운송장 등록:
    - delivery INSTRUCT → DEPARTURE 전이와 함께
      order_item도 CONFIRMED → SHIPPING으로 전이한다
- 주문 취소와 배송 처리:
    - 결제 전 취소: 배송이 없으므로 배송 상태 변경/배송 이력 저장 없음
    - 결제 후 취소: ACCEPT / INSTRUCT 배송이 존재하므로 ORDER_CANCELLED로 상태 변경하고 배송 이력도 저장
- 판매자 주문 목록에서 취소/거절된 주문상품은 숨기지 않고 상태 그대로 노출
    - 취소: OrderItem.status = CANCELLED, Delivery.status = ORDER_CANCELLED
    - 거절: OrderItem.status = REJECTED, Delivery.status = ORDER_CANCELLED
- 주문거절(판매자)
    - 판매자는 발주 확인(confirm) 전, order_item이 `ORDERED` 상태일 때 이행 불가한 주문을 거절할 수 있다.
    - 거절 단위는 주문 상품(order_item) 개별이다
      (한 주문에 여러 판매자 상품이 포함될 수 있으므로)
    - 처리:
        - 재고 즉시 복구
        - order_item `ORDERED → REJECTED`.
        - 거절된 order_item의 Delivery는 ORDER_CANCELLED로 전이되며 DeliveryHistory에 ORDER_CANCELLED 이력을 저장.
        - OrderCancelHistory에 거절 이력 저장 (cancel_type: SELLER_REJECT, 거절 사유 포함)
    - 전체 거절 시 자동 취소:
        - 거절 후 해당 주문의 모든 order_item이 REJECTED 상태이면 주문 자동 취소 처리
        - Payment 상태: CANCELLED
        - 포인트: 원래 만료일이 유효한 포인트만 복구 (만료된 포인트는 복구하지 않음)
        - 쿠폰: 만료 전 AVAILABLE, 만료 후 EXPIRED
        - Order 상태: CANCELLED
    - 일부 아이템만 거절된 경우:
        - 포인트/쿠폰 복구는 수행하지 않는다
- 배송 완료
    - 배송 완료 시 포인트 자동 적립(실결제 기준 1%), 리뷰 적기 가능

### 결제 승인 시 배송 생성 정책

- 결제 승인 성공 시 `DeliveryService.createDeliveriesForPayment()`에서 주문 상품 접수 및 배송 생성을 처리한다.
- `DeliveryService.createDeliveriesForPayment()`는 결제 승인 트랜잭션 안에서 호출된다.
- 처리 내용은 다음과 같다.
    1. OrderItem 상태를 `PENDING_PAYMENT → ORDERED`로 변경한다.
    2. 주문 상품(order_item)별로 Delivery를 생성한다.
    3. 최초 Delivery 상태는 `ACCEPT`로 설정한다.
    4. DeliveryHistory에 `ACCEPT` 이력을 저장한다.
- 배송 생성 중 주문 상품이 없거나 이미 배송 데이터가 존재하는 경우 결제 승인 트랜잭션은 롤백된다.
- 즉, 배송 생성은 결제 승인 성공을 위한 핵심 도메인 처리에 포함된다.