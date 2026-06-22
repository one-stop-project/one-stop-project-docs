## 6. 주문 / 결제 (Order / Payment)

> **결제 플로우 (2단계 Mock)**
1단계: 주문 생성 → 재고 선점 → 주문ID 반환
2단계: Mock 결제 승인 → 포인트/쿠폰 차감 + Outbox 이벤트 (단일 트랜잭션)
실제 PG사 연동 없음, 백엔드 결제 흐름에 집중
>

### POST /api/orders — 주문 생성 (1단계)

**권한** BUYER

동시성: 비관적 락 (재고 선점) | item_id 오름차순 정렬로 데드락 방지

**Request**

```json
{
  "orderType": "CART", // CART(장바구니) 또는 DIRECT(바로구매)
  "cartItemIds": [12, 15], // 장바구니 결제 시 (선택된 장바구니 아이템 ID 목록)
  "directItems": [         // 바로구매 시 (선택한 품목 ID와 수량)
    { "itemId": 101, "quantity": 1 }
  ],
  "receiverName": "홍길동",
  "receiverPhone": "010-1234-5678",
  "receiverAddress": "서울시 강남구 테헤란로 123",
  "userCouponId": 5,
  "usedPoint": 5000
}
```

**Response** `201`

```json
{
  "success": true,
  "data": {
    "orderId": 1001,
    "orderItems": [
      {
        "orderItemId": 1,
        "itemId": 101,
        "itemName": "맥북 프로 14인치 M3 (스페이스 그레이 / 512GB)",
        "price": 2190000,
        "quantity": 1,
        "subtotal": 2190000,
        "sellerName": "애플스토어",
        "status": "ORDERED"
      }
    ],
    "totalPrice": 2908000,
    "discountPrice": 50000,
    "usedPoint": 5000,
    "subscriptionDiscount": 0,
    "finalPrice": 2853000,
    "status": "PENDING_PAYMENT",
    "createdAt": "2025-05-12T14:30:00"
  }
}
```

> `status: PENDING_PAYMENT` — 결제 승인 전 상태
>

**에러**

| 코드 | 설명 |
| --- | --- |
| ORDER_001 | 존재하지 않는 상품 |
| ORDER_002 | 재고 부족 |
| ORDER_003 | 비활성 상품 |
| ORDER_005 | 쿠폰 사용 불가 (만료/사용됨) |
| POINT_001 | 포인트 부족 |

---

### POST /api/payments — 결제 승인 (2단계 Mock)

**권한** BUYER

동시성: 낙관적 락 (포인트 차감) + Redis 분산락 (쿠폰 차감) + Outbox INSERT (단일 트랜잭션)

**Request**

```json
{
  "orderId": 1001,
  "amount": 2853000
}
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "orderId": 1001,
    "finalPrice": 2853000,
    "status": "PAID",
    "paidAt": "2025-05-12T14:31:00"
  }
}
```

> Mock 결제 승인 처리 (실제 PG사 연동 없음)
결제 확정 후 Kafka `order-paid` 이벤트 발행 (Outbox Pattern)
Consumer: 배송 ACCEPT 생성 / 알림 발송
>

**에러**

| 코드 | 설명 |
| --- | --- |
| PAYMENT_001 | 결제 금액 불일치 |
| PAYMENT_002 | 이미 결제된 주문 |

---

### GET /api/orders — 내 주문 목록

**권한** BUYER

**Query** `status`, `from`, `to`, `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "orderId": 1001,
        "finalPrice": 2853000,
        "status": "PAID",
        "itemCount": 3,
        "firstItemName": "맥북 프로 14인치 M3",
        "firstItemThumbnail": "https://cdn.example.com/products/42/thumb.jpg",
        "createdAt": "2025-05-12T14:30:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 12, "totalPages": 1
  }
}
```

---

### GET /api/orders/{orderId} — 주문 단건 조회

**권한** BUYER (본인 주문)

**Response** `200`

```json
{
  "success": true,
  "data": {
    "orderId": 1001,
    "status": "PAID",
    "deliveryFee": 3000, // 추가: ERD의 delivery_fee 반영
    "orderItems": [
      {
        "orderItemId": 1,
        "itemId": 101,
        "itemName": "맥북 프로 14인치 M3",
        "sellerId": 3,
        "quantity": 1,
        "price": 2190000,
        "status": "ORDERED",
        // ★ 배송 정보가 개별 아이템 안에 위치함 (ERD 반영)
        "delivery": {
          "deliveryId": 501,
          "status": "ACCEPT",
          "deliveryCompany": "CJ대한통운",
          "invoiceNumber": "1234567890" 
        }
      }
    ],
    "receiver": {
      "name": "홍길동",
      "phone": "010-1234-5678",
      "address": "서울시 강남구 테헤란로 123"
    },
    "totalPrice": 2908000,
    "discountPrice": 50000,
    "usedPoint": 5000,
    "subscriptionDiscount": 0,
    "finalPrice": 2853000,
    "createdAt": "2025-05-12T14:30:00"
  }
}
```

> order_item.status: ORDERED / CONFIRMED / REJECTED / DELIVERED
delivery.status: ACCEPT / INSTRUCT / DEPARTURE / DELIVERING / FINAL_DELIVERY
>

---

### POST /api/orders/{orderId}/cancel — 주문 취소

**권한** BUYER | delivery.status DEPARTURE 이전까지 가능 (ACCEPT, INSTRUCT 상태에서 취소 가능)

동시성: 낙관적 락 + 상태머신 검증

**Request**

```json
{ "reason": "단순 변심" }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "orderId": 1001,
    "status": "CANCELLED",
    "refundAmount": 2853000,
    "restoredPoint": 5000,
    "restoredCoupon": {
      "userCouponId": 5,
      "status": "AVAILABLE"
    }
  }
}
```

> 재고 복구 + 포인트 전액 복구 + 쿠폰 복구 (만료 전 AVAILABLE, 만료 후 EXPIRED)
Kafka `order-cancelled` 이벤트 발행
>

**에러**

| 코드 | 설명 |
| --- | --- |
| ORDER_008 | 취소 불가 상태 (DEPARTURE 이후) |
| ORDER_009 | 낙관적 락 충돌 |

---

