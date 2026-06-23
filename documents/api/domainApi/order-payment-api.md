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
  "items": [               // DIRECT(바로구매) 시 (상품 옵션 ID와 수량)
    { "itemId": 101, "quantity": 1 }
  ],
  "cartItemIds": [12, 15], // CART(장바구니) 결제 시 (선택된 cart_item ID 목록)
  "receiverName": "홍길동",
  "receiverPhone": "010-1234-5678",
  "receiverAddress": "서울시 강남구 테헤란로 123",
  "deliveryMessage": "부재 시 경비실에 맡겨주세요", // 선택, 최대 50자
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
        "status": "PENDING_PAYMENT"
      }
    ],
    "totalPrice": 2908000,
    "discountPrice": 50000,
    "usedPoint": 5000,
    "subscriptionDiscount": 0,
    "deliveryFee": 3000,
    "finalPrice": 2856000,
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
| ORDER_001 | 주문할 상품이 없음 (주문 유형/항목 누락) |
| ORDER_003 | 판매하지 않는(판매 중지) 상품 포함 |
| ORDER_010 | 최소 주문 금액(1,000원) 미만 |
| ORDER_011 | 미승인 판매자의 상품 |
| ORDER_012 | 할인·포인트·구독 할인 합이 상품 금액 초과 |
| PRODUCT_001 | 존재하지 않는 상품 옵션 |
| PRODUCT_002 | 미승인(판매 불가) 상품 |
| INVENTORY_001 | 재고 부족 |
| CART_004 | 장바구니 상품을 찾을 수 없음 (CART 주문) |
| CART_006 | 본인 장바구니가 아님 |
| POINT_001 | 포인트 계정 없음 |
| POINT_002 | 포인트 부족 |

> 쿠폰 검증 실패는 쿠폰 도메인 에러코드(`COUPON_*`)로 응답한다.
> 본문 필드 유효성 위반(수령인/주소 누락, 수량/포인트 음수 등)은 400 검증 에러로 응답한다.
>

---

### POST /api/payments — 결제 승인 (2단계 Mock)

**권한** BUYER

동시성: 주문 비관적 락 (중복 결제 방지) + 포인트 낙관적 락 재시도 (최대 3회) + Outbox 저장 (단일 트랜잭션)

**Request**

```json
{
  "orderId": 1001,
  "amount": 2856000
}
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "orderId": 1001,
    "finalPrice": 2856000,
    "status": "PAID",
    "paidAt": "2025-05-12T14:31:00"
  }
}
```

> Mock 결제 승인 처리 (실제 PG사 연동 없음)
결제 승인 시 배송 생성(ACCEPT)은 동일 트랜잭션에서 동기 처리
결제 승인 완료 이벤트는 Outbox 테이블에 저장(`payment-approved-*`)되어 후속 알림 등에 사용
>

**에러**

| 코드 | 설명 |
| --- | --- |
| ORDER_006 | 주문을 찾을 수 없음 |
| ORDER_007 | 본인 주문이 아님 |
| PAYMENT_001 | 이미 결제된 주문 |
| PAYMENT_002 | 결제 금액이 주문 금액과 불일치 |
| PAYMENT_003 | 이미 처리된 결제 요청 |
| PAYMENT_008 | 취소된 주문 |
| PAYMENT_010 | 결제 처리 중 (재시도 소진) |
| PAYMENT_011 | 결제 대기 상태가 아닌 주문 |
| POINT_001 | 포인트 계정 없음 |
| POINT_002 | 포인트 부족 |

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
        "finalPrice": 2856000,
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
    "finalPrice": 2856000,
    "createdAt": "2025-05-12T14:30:00"
  }
}
```

> order_item.status: PENDING_PAYMENT / ORDERED / CONFIRMED / SHIPPING / DELIVERED / CANCELLED / REJECTED
delivery.status: ACCEPT / INSTRUCT / DEPARTURE / DELIVERING / FINAL_DELIVERY / ORDER_CANCELLED
> 결제 전(PENDING_PAYMENT) 주문은 배송 정보가 없어 `delivery`가 null일 수 있음
>

**에러**

| 코드 | 설명 |
| --- | --- |
| ORDER_006 | 주문을 찾을 수 없음 |
| ORDER_007 | 본인 주문이 아님 |

---

### POST /api/orders/{orderId}/cancel — 주문 취소

**권한** BUYER | delivery.status DEPARTURE 이전까지 가능 (ACCEPT, INSTRUCT 상태에서 취소 가능)

동시성: 비관적 락 (findByIdWithLock) + 배송 상태 기준 취소 가능 검증

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
    "refundAmount": 2856000,
    "restoredPoint": 5000,
    "restoredCoupon": {
      "userCouponId": 5,
      "couponName": "신규가입 5만원 할인",
      "status": "AVAILABLE"
    }
  }
}
```

> 재고 복구 + 포인트 전액 복구 + 쿠폰 복구 (만료 전 AVAILABLE, 만료 후 EXPIRED)
> 쿠폰/포인트 미사용 주문은 `restoredCoupon: null`, `restoredPoint: 0`
>

**에러**

| 코드 | 설명 |
| --- | --- |
| ORDER_006 | 주문을 찾을 수 없음 |
| ORDER_007 | 본인 주문이 아님 |
| ORDER_008 | 취소 불가 상태 (DEPARTURE 이후 / 이미 취소됨) |
| ORDER_013 | 주문 처리 중 (비관적 락 충돌) |
| SHIPPING_005 | 배송 정보를 찾을 수 없음 |
| PAYMENT_009 | 결제 정보를 찾을 수 없음 (결제 완료 주문 취소 시) |

---

