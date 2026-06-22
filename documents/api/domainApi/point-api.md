## 10. 포인트 (Point)

> 적립: FINAL_DELIVERY 시 결제금액 1% 자동 적립
만료: 적립일로부터 1년, 스케줄러 매월 1일 일괄 처리
차감: 낙관적 락 (@Version + @Retryable 3회)
환불: 주문 취소 시 전액 복구
>

### GET /api/users/me/points — 포인트 잔액 + 이력

**권한** BUYER

**Query** `type` (EARN / USE / EXPIRE), `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "balance": 32500,
    "expiringSoon": {
      "amount": 3000,
      "expireAt": "2025-06-01"
    },
    "history": {
      "content": [
        {
          "historyId": 100,
          "amount": -5000,
          "type": "USE",
          "description": "주문 #1001 사용",
          "createdAt": "2025-05-12T14:30:00"
        },
        {
          "historyId": 99,
          "amount": 12000,
          "type": "EARN",
          "description": "주문 #1000 적립 (1%)",
          "expireAt": "2026-05-10",
          "createdAt": "2025-05-10T10:00:00"
        }
      ],
      "page": 0, "size": 20, "totalElements": 45, "totalPages": 3
    }
  }
}
```

---

### POST /api/users/me/points/charge — 포인트 충전 (테스트용)

**권한** BUYER | 낙관적 락

**Request**

```json
{ "amount": 10000 }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "balance": 42500,
    "earnedAmount": 10000,
    "expireAt": "2026-05-12"
  }
}
```

> 실제 서비스는 결제 적립만 존재. 테스트 편의를 위한 임시 API.
>

**에러** `POINT_001` 최소 금액 미달 (1,000원) | `POINT_002` 최대 금액 초과