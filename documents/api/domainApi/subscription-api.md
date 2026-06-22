## 12. 구독 (Subscription)

> 스케줄러: 매일 자정 end_at 지난 구독 EXPIRED 일괄 처리
중복 신청: ACTIVE 또는 CANCELLED 상태의 구독이 존재하면 재신청 불가
해지: 즉시 해지, endAt까지 혜택 유지, 환불 없음
>

### GET /api/subscriptions/info — 구독 상품 정보

**권한** 전체

**Response** `200`

```json
{
  "success": true,
  "data": {
    "name": "OneStop 멤버십",
    "price": 4990,
    "billingCycle": "30일",
    "benefits": [
      "무료 배송",
      "새벽 배송",
      "추가 5% 할인",
      "무료 반품"
    ]
  }
}
```

---

### POST /api/subscriptions — 구독 신청

**권한** BUYER

**Response** `201`

```json
{
  "success": true,
  "data": {
    "subscriptionId": 12,
    "startAt": "2025-05-12T00:00:00",
    "endAt": "2025-06-12T00:00:00",
    "nextPaymentDate": "2025-06-11T23:59:59",
    "status": "ACTIVE"
  }
}
```

**에러** `SUBSCRIPTION_002` 이미 활성 구독 존재

---

### GET /api/subscriptions/me — 내 구독 정보

**권한** BUYER

**Response** `200`

```json
{
  "success": true,
  "data": {
    "subscriptionId": 12,
    "startAt": "2025-05-12T00:00:00",
    "endAt": "2025-06-11T23:59:59",
    "nextPaymentDate": "2025-06-11T23:59:59",
    "status": "ACTIVE",
    "daysLeft": 9
  }
}
```

---

### POST /api/subscriptions/me/cancel — 구독 해지

**권한** BUYER

**Request**

```json
{ "reason": "이용 빈도가 낮아서" }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "subscriptionId": 12,
    "status": "CANCELLED",
    "endAt": "2025-06-12T00:00:00",
    "message": "구독이 해지되었습니다. 만료일까지 혜택은 유지됩니다."
  }
}
```

---
