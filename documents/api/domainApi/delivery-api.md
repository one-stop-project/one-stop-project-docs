## 7. 배송 (Delivery)

> **배송 상태 (delivery.status)**`ACCEPT → INSTRUCT → DEPARTURE → DELIVERING → FINAL_DELIVERY / ORDER_CANCELLED`
>
>
> **주문 상품 상태 (order_item.status)** `PENDING_PAYMENT → ORDERED → CONFIRMED → SHIPPING → DELIVERED / CANCELLED / REJECTED`
>
> 두 상태는 별도 관리. delivery는 order_item 1:1 관계.
>

### GET /api/orders/{orderId}/deliveries — 주문의 배송 목록

**권한** BUYER

**Response** `200`

```json
{
  "success": true,
  "data": [
    {
      "deliveryId": 501,
      "orderItemId": 1,
      "itemName": "맥북 프로 14인치 M3 (스페이스 그레이 / 512GB)",
      "sellerName": "애플스토어",
      "status": "DELIVERING",
      "invoiceNumber": "1234567890",
      "deliveryCompany": "CJ대한통운",
      "updatedAt": "2025-05-13T09:00:00"
    },
    {
      "deliveryId": 502,
      "orderItemId": 2,
      "itemName": "에어팟 프로 2세대",
      "sellerName": "애플스토어",
      "status": "INSTRUCT",
      "invoiceNumber": null,
      "deliveryCompany": null,
      "updatedAt": "2025-05-12T15:00:00"
    }
  ]
}
```

> 판매자별로 배송이 분리되어 한 주문에 여러 배송이 존재할 수 있음
>

---

### GET /api/deliveries/{deliveryId}/history — 배송 이력

**권한** BUYER

**Response** `200`

```json
{
  "success": true,
  "data": {
    "deliveryId": 501,
    "currentStatus": "DELIVERING",
    "invoiceNumber": "1234567890",
    "deliveryCompany": "CJ대한통운",
    "history": [
      { "status": "ACCEPT", "statusDesc": "결제완료", "changedAt": "2025-05-12T14:35:00" },
      { "status": "INSTRUCT", "statusDesc": "상품준비중", "changedAt": "2025-05-12T16:00:00" },
      { "status": "DEPARTURE", "statusDesc": "배송지시", "changedAt": "2025-05-13T08:30:00" },
      { "status": "DELIVERING", "statusDesc": "배송중", "changedAt": "2025-05-13T09:00:00" }
    ]
  }
}
```

---

## 8. 판매자 주문 / 배송 관리 (Seller - Order)

### GET /api/seller/orders — 판매자 주문 목록

**권한** SELLER | 본인 상품 order_item만 조회

**Query** `status`, `from`, `to`, `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "orderItemId": 1,
        "orderId": 1001,
        "itemName": "맥북 프로 14인치 M3 (스페이스 그레이 / 512GB)",
        "quantity": 1,
        "price": 2190000,
        "subtotal": 2190000,
        "buyerName": "홍*동",
        "receiverAddress": "서울시 강남구 테헤란로 123",
        "orderItemStatus": "ORDERED",
        "deliveryStatus": "ACCEPT",
        "orderedAt": "2025-05-12T14:30:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 5, "totalPages": 1
  }
}
```

---

### POST /api/seller/orders/{orderItemId}/confirm — 발주 확인 (상품준비중)

**권한** SELLER

상태 전이: `order_item ORDERED → CONFIRMED` / `delivery ACCEPT → INSTRUCT`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "orderItemId": 1,
    "orderItemStatus": "CONFIRMED",
    "deliveryStatus": "INSTRUCT"
  }
}
```

> Kafka `order-confirmed` 발행 → 구매자 알림
>

**에러** `ORDER_010` ORDERED 상태 아님

---

### POST /api/seller/orders/{orderItemId}/reject — 주문 거절

**권한** SELLER

상태 전이: `order_item ORDERED → REJECTED / delivery ACCEPT → ORDER_CANCELLED`

**Request**

```json
{ "reason": "재고 소진" }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "orderItemId": 1,
    "orderItemStatus": "REJECTED",
    "deliveryStatus": "ORDER_CANCELLED",
    "rejectedPrice": 2190000,
    "restoredStock": 1,
    "orderAutoCancelled": false
  }
}
```

>
>
> - 재고 복구 + Delivery ORDER_CANCELLED 전이 + DeliveryHistory 저장
> - OrderCancelHistory 저장 (cancel_type: SELLER_REJECT, 거절 사유 포함)
> - rejectedPrice: 거절된 상품 금액 (가격 × 수량, 프론트용 정보성)
> - orderAutoCancelled: 해당 주문의 모든 상품이 거절되어 주문이 자동 취소된 경우 true
    > (자동 취소 시 포인트/쿠폰 복구 포함)

---

### POST /api/seller/deliveries/{deliveryId}/ship — 운송장 등록 (배송지시)

**권한** SELLER

상태 전이: `delivery INSTRUCT → DEPARTURE / order_item CONFIRMED → SHIPPING`

**Request**

```json
{
  "deliveryCompany": "CJ대한통운",
  "invoiceNumber": "1234567890"
}
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "deliveryId": 501,
    "status": "DEPARTURE",
    "deliveryCompany": "CJ대한통운",
    "invoiceNumber": "1234567890",
    "shippedAt": "2025-05-13T09:00:00"
  }
}
```

> DeliveryHistory 기록 + Kafka `shipping-started` 발행
>

**에러** `SHIPPING_001` INSTRUCT 상태 아님 | `SHIPPING_003` 운송장 번호 없음

---

### PATCH /api/seller/deliveries/{deliveryId}/status — 배송 상태 변경

**권한** SELLER

상태 전이:

- `DEPARTURE → DELIVERING`
- `DELIVERING → FINAL_DELIVERY`

**Request**

```json
{ "status": "DELIVERING" }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "deliveryId": 501,
    "status": "DELIVERING",
    "updatedAt": "2025-05-13T14:00:00"
  }
}
```

> FINAL_DELIVERY 변경 시:
>
> - order_item.status → DELIVERED
> - 포인트 자동 적립 (결제금액 1%)
> - 리뷰 작성 가능 상태로 변경
> - Kafka `shipping-delivered` 발행

**에러** `SHIPPING_002` 허용되지 않는 상태 전이