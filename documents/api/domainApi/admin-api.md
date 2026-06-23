## 13. 관리자 (Admin)

> 모든 엔드포인트 공통 권한: `hasAnyRole('ADMIN', 'SUPER_ADMIN')`
> 단, 관리자 권한 관리(`/api/admin/users/**`)와 보안 조치(`/api/admin/security/**`)는 `SUPER_ADMIN` 전용
>
> 데이터 없는 성공 응답(`Void`)은 `{ "success": true }` 형태로 반환한다.

### GET /api/admin/sellers — 승인 대기 판매자 목록

**권한** ADMIN | SUPER_ADMIN

`PENDING` 상태 판매자 전체를 리스트로 반환한다. (페이징·상태 필터 없음)

**Response** `200`

```json
{
  "success": true,
  "data": [
    {
      "sellerId": 3,
      "userId": 7,
      "shopName": "애플스토어",
      "businessNumber": "123-****-90",
      "status": "PENDING"
    }
  ]
}
```

> `businessNumber`는 마스킹 처리(`앞3자리-****-뒤2자리`)되어 내려간다.
>

---

### PATCH /api/admin/sellers/{sellerId}/approve — 판매자 승인

**권한** ADMIN | SUPER_ADMIN | 상태 전이: `PENDING → APPROVED`

**Response** `200`

```json
{ "success": true }
```

**에러** `SELLER_001` 판매자 없음 · `ADMIN_018` 승인 대기 상태가 아님

---

### PATCH /api/admin/sellers/{sellerId}/reject — 판매자 반려

**권한** ADMIN | SUPER_ADMIN | 상태 전이: `PENDING → REJECTED`

**Request**

```json
{ "reason": "사업자번호 확인 불가" }
```

**Response** `200`

```json
{ "success": true }
```

**에러** `SELLER_001` 판매자 없음 · `ADMIN_018` 승인 대기 상태가 아님 · `reason` 누락 시 `400`

---

### PATCH /api/admin/sellers/{sellerId}/force-inactive — 판매자 강제 비활성화

**권한** ADMIN | SUPER_ADMIN | 상태 전이: `→ SUSPENDED`

**Request**

```json
{ "reason": "신고 누적으로 강제 비활성화" }
```

**Response** `200`

```json
{ "success": true }
```

> 판매자/연결 회원 정지, 해당 판매자 상품 전체 `FORCE_INACTIVE` 전환,
> `ORDERED`/`CONFIRMED` 주문 자동 취소 + 재고 복구(주문 전체 취소 시 포인트·쿠폰 복구),
> 토큰 무효화 + 전 기기 로그아웃을 함께 처리한다.

**에러** `SELLER_001` 판매자 없음 · `ADMIN_019` 이미 정지된 판매자 · `reason` 누락 시 `400`

---

### PATCH /api/admin/sellers/{sellerId}/reactivate — 판매자 정지 해제

**권한** ADMIN | SUPER_ADMIN | 상태 전이: `SUSPENDED → APPROVED`

**Response** `200`

```json
{ "success": true }
```

> 정지 해제 시 상품은 정책상 자동 복구하지 않는다.

**에러** `SELLER_001` 판매자 없음 · `ADMIN_010` 정지 상태인 판매자가 아님

---

### GET /api/admin/products — 승인 요청 상품 목록

**권한** ADMIN | SUPER_ADMIN

`APPROVE_REQUESTED` 상태 상품만 페이징 조회한다. (상태/판매자/카테고리 필터 없음)

**Query** `page`, `size` (기본 20, `id` DESC 정렬)

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "productId": 42,
        "sellerId": 3,
        "name": "맥북 프로 14인치 M3",
        "description": "M3 칩 탑재 14인치 노트북",
        "thumbnailUrl": "https://cdn.example.com/products/42/thumb.jpg",
        "status": "APPROVE_REQUESTED"
      }
    ],
    "page": 0, "size": 20, "totalElements": 12, "totalPages": 1
  }
}
```

---

### PATCH /api/admin/products/{productId}/approve — 상품 승인

**권한** ADMIN | SUPER_ADMIN | 상태 전이: `APPROVE_REQUESTED → APPROVED`

**Response** `200`

```json
{ "success": true }
```

**에러** `PRODUCT_001` 상품 없음 · `ADMIN_008` 이미 승인됨 · `ADMIN_017` 승인 요청 상태가 아님

---

### PATCH /api/admin/products/{productId}/reject — 상품 반려

**권한** ADMIN | SUPER_ADMIN | 상태 전이: `APPROVE_REQUESTED → REJECTED`

**Request**

```json
{ "reason": "상품 설명 부적절" }
```

**Response** `200`

```json
{ "success": true }
```

**에러** `PRODUCT_001` 상품 없음 · `ADMIN_009` 이미 반려됨 · `ADMIN_017` 승인 요청 상태가 아님 · `reason` 누락 시 `400`

---

### PATCH /api/admin/products/{productId}/force-inactive — 상품 강제 비활성화

**권한** ADMIN | SUPER_ADMIN | 상태 전이: `→ FORCE_INACTIVE`

**Request**

```json
{ "reason": "신고 누적으로 강제 비활성화" }
```

**Response** `200`

```json
{ "success": true }
```

**에러** `PRODUCT_001` 상품 없음 · `ADMIN_007` 이미 강제 비활성화됨 · `reason` 누락 시 `400`

---

### POST /api/admin/coupons — 쿠폰 생성

**권한** ADMIN | SUPER_ADMIN

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

> `maxDiscountPrice`는 `RATE` 타입에 사용하며 `FIXED` 타입은 `null`로 둔다.
>

**Response** `201`

```json
{
  "success": true,
  "data": {
    "couponId": 10,
    "name": "여름 세일 10% 할인",
    "discountType": "RATE",
    "discountValue": 10,
    "minOrderPrice": 50000,
    "maxDiscountPrice": 30000,
    "totalQuantity": 1000,
    "issuedQuantity": 0,
    "remainingQuantity": 1000,
    "status": "ACTIVE",
    "startAt": "2025-06-01T00:00:00",
    "expiredAt": "2025-06-30T23:59:59",
    "createdAt": "2025-05-12T09:00:00"
  }
}
```

---

### GET /api/admin/coupons — 쿠폰 목록

**권한** ADMIN | SUPER_ADMIN

**Query** `status` (`ACTIVE` / `INACTIVE`, 선택), `page`, `size` (기본 20, `createdAt` DESC)

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "couponId": 10,
        "name": "여름 세일 10% 할인",
        "discountType": "RATE",
        "discountValue": 10,
        "minOrderPrice": 50000,
        "maxDiscountPrice": 30000,
        "totalQuantity": 1000,
        "issuedQuantity": 245,
        "remainingQuantity": 755,
        "status": "ACTIVE",
        "startAt": "2025-06-01T00:00:00",
        "expiredAt": "2025-06-30T23:59:59",
        "createdAt": "2025-05-12T09:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 12, "totalPages": 1
  }
}
```

---

### PATCH /api/admin/coupons/{couponId}/deactivate — 쿠폰 비활성화

**권한** ADMIN | SUPER_ADMIN | 상태 전이: `ACTIVE → INACTIVE`

**Response** `200` (비활성화된 쿠폰 정보 반환, 필드는 쿠폰 목록과 동일)

```json
{
  "success": true,
  "data": { "couponId": 10, "status": "INACTIVE" }
}
```

**에러** `COUPON_004` 쿠폰 없음 · `COUPON_010` 이미 비활성화된 쿠폰

---

### GET /api/admin/search/popular — 인기 검색어

**권한** ADMIN | SUPER_ADMIN | Redis ZSET

**Query** `date` (기본 오늘, KST), `limit` (기본 20, 1~50)

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

**에러** `PRODUCT_014` `limit`이 허용 범위(1~50)를 벗어남

---

### GET /api/admin/orders — 전체 주문 현황

**권한** ADMIN | SUPER_ADMIN

**Query** `status` (OrderStatus, 선택), `from`, `to` (`yyyy-MM-dd`, 선택), `keyword` (구매자 이름/이메일, 선택), `page`, `size` (기본 20, `createdAt` DESC)

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "orderId": 1001,
        "userName": "홍길동",
        "userEmail": "hong@example.com",
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

**에러** `COMMON_001` `from`이 `to`보다 이후

---

### GET /api/admin/subscriptions/stats — 구독 상태별 통계

**권한** ADMIN | SUPER_ADMIN

**Response** `200`

```json
{
  "success": true,
  "data": {
    "activeCount": 320,
    "cancelledCount": 45,
    "expiredCount": 135,
    "totalCount": 500
  }
}
```

---

### GET /api/admin/subscriptions — 전체 구독 목록

**권한** ADMIN | SUPER_ADMIN

**Query** `page`, `size` (기본 20, `createdAt` DESC) — 상태 필터 없음

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "subscriptionId": 12,
        "userId": 7,
        "userEmail": "hong@example.com",
        "userName": "홍길동",
        "status": "ACTIVE",
        "startAt": "2025-05-12T00:00:00",
        "endAt": "2025-06-12T00:00:00",
        "nextPaymentDate": "2025-06-11T23:59:59",
        "cancelReason": null,
        "createdAt": "2025-05-12T00:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 500, "totalPages": 25
  }
}
```

---

### GET /api/admin/dashboard — 관리자 대시보드

**권한** ADMIN | SUPER_ADMIN | 파라미터 없음

주문 상태별 건수, 오늘 주문 수/매출, 배송 상태별 건수를 반환한다.

**Response** `200`

```json
{
  "success": true,
  "data": {
    "orders": {
      "statusCount": { "PAID": 1245, "ORDERED": 30, "CANCELLED": 12 },
      "todayOrderCount": 87,
      "todayRevenue": 45800000
    },
    "deliveries": {
      "statusCount": { "READY": 20, "SHIPPING": 14, "COMPLETED": 1100 }
    }
  }
}
```

> `statusCount`의 키는 각각 `OrderStatus`, `DeliveryStatus`의 전체 값이며, 건수가 0인 상태도 포함된다.
> `todayRevenue`는 당일 `PAID` 주문의 최종 결제 금액 합계이다.

---

### GET /api/admin/action-histories — 관리자 처리 이력

**권한** ADMIN | SUPER_ADMIN

`SUPER_ADMIN`은 전체 이력을, `ADMIN`은 본인 처리 이력만 조회한다.

**Query** `page`, `size` (기본 20, `createdAt` DESC)

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": 501,
        "actorId": 2,
        "targetType": "SELLER",
        "targetId": 3,
        "action": "APPROVE",
        "reason": null,
        "createdAt": "2025-05-12T16:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 340, "totalPages": 17
  }
}
```

> `targetType`: `SELLER` / `PRODUCT` / `ADMIN_USER`
> `action`: `APPROVE` / `REJECT` / `FORCE_INACTIVE` / `REACTIVATE` / `GRANT_ADMIN` / `REVOKE_ADMIN`

---

### GET /api/admin/users — 관리자 목록

**권한** SUPER_ADMIN 전용

`ADMIN`/`SUPER_ADMIN` 역할 회원을 페이징 조회한다.

**Query** `page`, `size` (기본 20, `createdAt` DESC)

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": 2,
        "email": "admin@onestop.com",
        "name": "관리자",
        "role": "ADMIN",
        "status": "ACTIVE",
        "createdAt": "2025-04-01T09:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 5, "totalPages": 1
  }
}
```

---

### PATCH /api/admin/users/{userId}/grant — 관리자 권한 부여

**권한** SUPER_ADMIN 전용 | 역할 전이: `BUYER → ADMIN`

**Response** `200`

```json
{ "success": true }
```

> 권한 변경 후 대상자의 모든 기기를 로그아웃 처리하여 기존 토큰의 역할 클레임을 즉시 무효화한다.

**에러** `MEMBER_001` 회원 없음 · `ADMIN_011` 판매자 계정 승격 불가 · `ADMIN_012` 이미 관리자 · `ADMIN_014` 비활성(정지/탈퇴) 회원 승격 불가

---

### PATCH /api/admin/users/{userId}/revoke — 관리자 권한 회수

**권한** SUPER_ADMIN 전용 | 역할 전이: `ADMIN → BUYER`

**Response** `200`

```json
{ "success": true }
```

> 권한 회수 후 대상자의 모든 기기를 로그아웃 처리한다.

**에러** `MEMBER_001` 회원 없음 · `ADMIN_014` 본인 계정 회수 불가 · `ADMIN_013` SUPER_ADMIN 회수 불가 · `ADMIN_015` 관리자 권한이 없는 계정

---

### POST /api/admin/security/users/{userId}/actions — 사용자 보안 조치

**권한** SUPER_ADMIN 전용

`BUYER`/`SELLER` 대상의 정지/정지해제/강제 로그아웃을 수행한다.

**Request**

```json
{
  "actionType": "SUSPEND",
  "reasonCode": "ABUSE_REPORTED",
  "reasonDetail": "반복 신고 누적",
  "durationMinutes": 1440
}
```

> `actionType`: `SUSPEND` / `UNSUSPEND` / `FORCE_LOGOUT`
> `durationMinutes`: 1~43200 (선택, `SUSPEND` 미지정 시 기본 1440분)
> `reasonDetail`은 민감정보 마스킹 후 저장된다.

**Response** `200`

```json
{
  "success": true,
  "data": {
    "targetUserId": 7,
    "actionType": "SUSPEND",
    "reasonCode": "ABUSE_REPORTED",
    "startedAt": "2025-05-12T16:00:00",
    "expiresAt": "2025-05-13T16:00:00"
  }
}
```

> `UNSUSPEND`/`FORCE_LOGOUT`은 `expiresAt`이 `null`이다.

**에러** `SECURITY_003` 권한 없음 · `SECURITY_004` 자기 자신 대상 · `SECURITY_002` 대상 없음 · `SECURITY_006` 제재 대상 역할 아님 · `SECURITY_007` 현재 상태에서 수행 불가

---

### GET /api/admin/security/users/{userId} — 보안 조치 대상 조회

**권한** SUPER_ADMIN 전용

**Response** `200`

```json
{
  "success": true,
  "data": {
    "id": 7,
    "email": "buyer@example.com",
    "name": "홍길동",
    "role": "BUYER",
    "status": "ACTIVE",
    "sanctionable": true
  }
}
```

> `sanctionable`은 역할이 `BUYER`/`SELLER`이고 상태가 `WITHDRAWN`이 아닐 때 `true`이다.

**에러** `SECURITY_002` 대상 없음

---
