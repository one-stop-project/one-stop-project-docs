## 10. 포인트 (Point)

> 적립: 주문의 모든 상품 배송 완료 시 결제금액 1% 자동 적립
만료: 적립일로부터 1년, 스케줄러 매일 새벽 3시(KST, `0 0 3 * * *`) 일괄 처리
차감: 낙관적 락 (@Version + @Retryable 3회)
환불: 주문 취소 시 전액 복구
>

### GET /api/users/me/points — 포인트 잔액 + 이력

**권한** BUYER

**Query** `type` (CHARGE / EARN / USE / REFUND / EXPIRE), `page`, `size`

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

### POST /api/users/me/points/charge — 포인트 충전 (운영 방어용)

**권한** ADMIN / SUPER_ADMIN | 낙관적 락

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

> 일반 사용자 충전 경로가 아니라 기존 테스트 충전을 차단하기 위한 방어용 경로다. `local`/`test`/`dev` 프로파일에서만 Bean이 등록된다.
>

**에러** `400` 검증 에러 (amount 누락 또는 1,000~1,000,000원 범위 밖) | `POINT_004` 충전 한도 위반(1,000~1,000,000원, 서비스 가드) | `POINT_005` 동시성 충돌 재시도 초과

---

### POST /api/test/users/me/points/charge — 포인트 충전 (테스트/시연용)

**권한** BUYER

**Header** `X-Test-Api-Key` 필수

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

> `app.test-api.point-charge.enabled=true`일 때만 Bean이 등록되는 부하 테스트/시연 전용 API. 실제 서비스 포인트 적립은 배송 완료 시 자동 적립만 존재한다.
>

**에러** `401` API Key 누락/불일치 (AUTH_007) | `400` 검증 에러 | `POINT_004` 충전 한도 위반(1,000~1,000,000원)