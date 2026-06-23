## 12. 구독 (Subscription)

> 스케줄러(매일 자정): ACTIVE + nextPaymentDate 도래분은 자동 결제(성공 → renew, 실패 → EXPIRED), CANCELLED + endAt 경과분은 EXPIRED 처리
중복 신청: ACTIVE 또는 CANCELLED 상태의 구독이 존재하면 재신청 불가
해지: 즉시 해지, endAt까지 혜택 유지, 환불 없음
>

### POST /api/subscriptions — 구독 신청

**권한** BUYER

**Response** `200`

```json
{
  "success": true,
  "data": {
    "subscriptionId": 12,
    "startAt": "2025-05-12T00:00:00",
    "endAt": "2025-06-11T00:00:00",
    "nextPaymentDate": "2025-06-11T00:00:00",
    "status": "ACTIVE",
    "cancelReason": null,
    "daysLeft": 30
  }
}
```

> `endAt`과 `nextPaymentDate`는 모두 가입 시점 + 30일로 동일하게 설정됨
>

**에러** `SUBSCRIPTION_002` 이미 유효한 구독 존재(ACTIVE/CANCELLED)

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
    "endAt": "2025-06-11T00:00:00",
    "nextPaymentDate": "2025-06-11T00:00:00",
    "status": "ACTIVE",
    "cancelReason": null,
    "daysLeft": 9
  }
}
```

**에러** `SUBSCRIPTION_001` 구독 정보 없음

---

### PATCH /api/subscriptions/cancel — 구독 해지

**권한** BUYER

**Request** (`reason` 필수)

```json
{ "reason": "이용 빈도가 낮아서" }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "subscriptionId": 12,
    "startAt": "2025-05-12T00:00:00",
    "endAt": "2025-06-11T00:00:00",
    "nextPaymentDate": "2025-06-11T00:00:00",
    "status": "CANCELLED",
    "cancelReason": "이용 빈도가 낮아서",
    "daysLeft": 9
  }
}
```

> 해지 후에도 `endAt`까지 혜택 유지(`daysLeft` 남은 일수). 만료는 스케줄러가 `endAt` 경과 시 EXPIRED로 전이
>

**에러** `SUBSCRIPTION_001` 구독 정보 없음 | `SUBSCRIPTION_003` 이미 해지됨 | `SUBSCRIPTION_011` 이미 만료됨

---
