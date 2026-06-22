## 11. 쿠폰 (Coupon)

> 1주문 1쿠폰 제한
선착순 발급: Redis SISMEMBER(중복 체크) + DECR(원자적 차감) + SADD(발급 마킹)
주문 취소 시 복구: 만료 전 AVAILABLE, 만료 후 EXPIRED 유지
>

### GET /api/coupons/available — 발급 가능 쿠폰 목록

**권한** BUYER

**Response** `200`

```json
{
  "success": true,
  "data": [
    {
      "couponId": 1,
      "name": "신규 가입 5,000원 할인",
      "discountType": "FIXED",
      "discountValue": 5000,
      "minOrderPrice": 30000,
      "maxDiscountPrice": null,
      "remainingQuantity": 47,
      "expiredAt": "2025-06-30T23:59:59",
      "alreadyIssued": false
    }
  ]
}
```

---

### POST /api/coupons/{couponId}/issue — 쿠폰 발급 (선착순)

**권한** BUYER | Redis 분산락

**Response** `200`

```json
{
  "success": true,
  "data": {
    "userCouponId": 301,
    "couponId": 2,
    "couponName": "전자기기 10% 할인",
    "status": "AVAILABLE",
    "expiredAt": "2025-05-31T23:59:59"
  }
}
```

**에러** `COUPON_001` 수량 소진 | `COUPON_002` 이미 발급 | `COUPON_003` 발급 기간 아님

---

### GET /api/users/me/coupons — 내 쿠폰 목록

**권한** BUYER

**Query** `status` (AVAILABLE / USED / EXPIRED), `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "userCouponId": 301,
        "couponId": 2,
        "name": "전자기기 10% 할인",
        "discountType": "RATE",
        "discountValue": 10,
        "minOrderPrice": 50000,
        "maxDiscountPrice": 30000,
        "status": "AVAILABLE",
        "expiredAt": "2025-05-31T23:59:59",
        "createdAt": "2025-05-12T10:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 8, "totalPages": 1
  }
}
```

---

