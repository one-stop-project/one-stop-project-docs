## 17. 알림 (Notification)

> 실시간 알림은 SSE(Server-Sent Events)로 전달한다.
> 도메인 이벤트 → Notification DB 저장 → Redis Pub/Sub 발행 → 구독 중인 서버가 SSE로 전송한다.
> 현재 알림 내역 조회·읽음 처리 API는 제공하지 않는다.

### GET /api/notifications/subscribe — 알림 구독 (SSE)

**권한** 인증 필요 | `Content-Type: text/event-stream`

인증된 사용자의 SSE 연결을 생성한다. 사용자당 1개의 연결만 유지하며, 재연결 시 기존 연결을 교체한다. 연결 타임아웃은 30분이다.

**이벤트**

| 이벤트 이름 | 시점 | data |
|---|---|---|
| `connect` | 연결 직후 | `"SSE 연결 성공"` |
| `notification` | 알림 발생 시 | 알림 페이로드(JSON) |

**`notification` 이벤트 페이로드**

```json
{
  "notificationId": 101,
  "type": "PAYMENT_APPROVED",
  "title": "결제 완료",
  "message": "주문 #13 결제가 완료되었습니다. (결제 금액: 192,000원)",
  "createdAt": "2025-05-12T14:30:00"
}
```

> `type`: 현재 `PAYMENT_APPROVED`(결제 완료)만 발행한다.

---
