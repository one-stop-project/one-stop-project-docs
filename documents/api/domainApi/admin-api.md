## 13. 관리자 (Admin)

### GET /api/admin/sellers — 판매자 목록

**권한** ADMIN

**Query** `status` (PENDING / APPROVED / REJECTED), `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "sellerId": 3,
        "shopName": "애플스토어",
        "businessNumber": "1234567890",
        "status": "PENDING",
        "userName": "홍길동",
        "createdAt": "2025-04-28T09:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 8, "totalPages": 1
  }
}
```

---

### PATCH /api/admin/sellers/{sellerId}/approve — 판매자 승인

**권한** ADMIN | 상태 전이: `PENDING → APPROVED`

**Response** `200`

```json
{
  "success": true,
  "data": { "sellerId": 3, "status": "APPROVED", "approvedAt": "2025-05-12T16:00:00" }
}
```

> Kafka `seller-approved` 발행 → 판매자 알림
>

---

### PATCH /api/admin/sellers/{sellerId}/reject — 판매자 반려

**권한** ADMIN | 상태 전이: `PENDING → REJECTED`

**Request**

```json
{ "reason": "사업자번호 확인 불가" }
```

**Response** `200`

```json
{
  "success": true,
  "data": { "sellerId": 3, "status": "REJECTED", "rejectReason": "사업자번호 확인 불가" }
}
```

> Kafka `seller-rejected` 발행 → 판매자 알림
연관 상품 전체 DISCONTINUED 처리
ORDERED/CONFIRMED 주문 자동 취소 + 복구 처리
>

---

### GET /api/admin/products — 전체 상품 목록

**권한** ADMIN

**Query** `status`, `sellerId`, `categoryId`, `keyword`, `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "productId": 42,
        "name": "맥북 프로 14인치 M3",
        "sellerName": "애플스토어",
        "categoryName": "노트북",
        "status": "APPROVE_REQUESTED",
        "createdAt": "2025-04-15T09:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 230, "totalPages": 12
  }
}
```

---

### PATCH /api/admin/products/{productId}/approve — 상품 승인

**권한** ADMIN | 상태 전이: `APPROVE_REQUESTED → APPROVED`

**Response** `200`

```json
{
  "success": true,
  "data": { "productId": 42, "status": "APPROVED" }
}
```

> Kafka `product-approved` 발행 → 판매자 알림
>

---

### PATCH /api/admin/products/{productId}/reject — 상품 반려

**권한** ADMIN | 상태 전이: `APPROVE_REQUESTED → REJECTED`

**Request**

```json
{ "reason": "상품 설명 부적절" }
```

**Response** `200`

```json
{
  "success": true,
  "data": { "productId": 42, "status": "REJECTED", "rejectReason": "상품 설명 부적절" }
}
```

> Kafka `product-rejected` 발행 → 판매자 알림
>

---

### DELETE /api/admin/products/{productId} — 상품 강제 비활성화

**권한** ADMIN | 상태 전이: `* → FORCE_INACTIVE`

**Request**

```json
{ "reason": "신고 누적으로 강제 비활성화" }
```

**Response** `200`

```json
{
  "success": true,
  "data": { "productId": 42, "status": "FORCE_INACTIVE" }
}
```

---

### POST /api/admin/coupons — 쿠폰 생성

**권한** ADMIN

**Request**

```json
{
  "name": "여름 세일 10% 할인",
  "discountType": "RATE",
  "discountValue": 10,
  "minOrderPrice": 50000,
  "maxDiscountPrice": 30000,
  "totalQuantity": 1000,
  "startAt": "2025-06-01T00:00:00",
  "expiredAt": "2025-06-30T23:59:59"
}
```

**Response** `201`

```json
{
  "success": true,
  "data": {
    "couponId": 10,
    "name": "여름 세일 10% 할인",
    "totalQuantity": 1000,
    "issuedQuantity": 0,
    "status": "ACTIVE"
  }
}
```

> Redis SET `coupon:stock:10` = 1000
>

---

### GET /api/admin/coupons — 쿠폰 목록

**권한** ADMIN

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "couponId": 10,
        "name": "여름 세일 10% 할인",
        "totalQuantity": 1000,
        "issuedQuantity": 245,
        "remainingQuantity": 755,
        "status": "ACTIVE",
        "expiredAt": "2025-06-30T23:59:59"
      }
    ],
    "page": 0, "size": 20, "totalElements": 12, "totalPages": 1
  }
}
```

---

### GET /api/admin/search/popular — 인기 검색어

**권한** ADMIN | Redis ZSET

**Query** `date` (기본 today), `limit` (기본 20)

**Response** `200`

```json
{
  "success": true,
  "data": {
    "date": "2025-05-12",
    "keywords": [
      { "rank": 1, "keyword": "맥북", "count": 342 },
      { "rank": 2, "keyword": "에어팟", "count": 215 }
    ]
  }
}
```

---

### GET /api/admin/orders — 전체 주문 현황

**권한** ADMIN

**Query** `status`, `from`, `to`, `keyword`, `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "orderId": 1001,
        "userName": "홍길동",
        "finalPrice": 2853000,
        "status": "PAID",
        "itemCount": 3,
        "createdAt": "2025-05-12T14:30:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 1245, "totalPages": 63
  }
}
```

---

### GET /api/admin/subscriptions — 구독 현황

**권한** ADMIN

**Query** `status` (ACTIVE / EXPIRED / CANCELLED), `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "subscriptionId": 12,
        "userName": "홍길동",
        "plan": "MONTHLY",
        "status": "ACTIVE",
        "endAt": "2025-06-12T00:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 500, "totalPages": 25
  }
}
```

---

### GET /api/admin/dashboard — 관리자 대시보드

**권한** ADMIN

**Query** `from`, `to`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "period": { "from": "2025-05-01", "to": "2025-05-12" },
    "summary": {
      "totalRevenue": 45800000,
      "totalOrders": 1245,
      "newMembers": 87,
      "newSellers": 5,
      "pendingApprovals": { "sellers": 3, "products": 12 }
    },
    "topProducts": [
      {
        "productId": 42,
        "name": "맥북 프로 14인치 M3",
        "salesCount": 34,
        "revenue": 74460000
      }
    ]
  }
}
```

---
