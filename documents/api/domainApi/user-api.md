## 2. 회원 (User)

### GET /api/users/me — 내 정보 조회

**권한** BUYER / SELLER

**Response** `200`

```json
{
  "success": true,
  "data": {
    "userId": 1,
    "email": "hong@example.com",
    "name": "홍길동",
    "phone": "010-1234-5678",
    "address": "서울시 강남구 테헤란로 123",
    "detailAddress": "10층 1001호",
    "role": "BUYER",
    "status": "ACTIVE",
    "social": false,
    "provider": "kakao",
    "subscription": { "active": true, "endAt": "2026-05-12T00:00:00" },
    "createdAt": "2025-05-01T10:00:00"
  }
}
```

> `social`은 항상 포함(소셜 로그인 여부). `provider`·`subscription`은 값이 없으면 응답에서 생략(`@JsonInclude(NON_NULL)`).

---

### PATCH /api/users/me — 내 정보 수정

**권한** BUYER / SELLER

**Request** `UserUpdateRequest` (모든 필드 선택 — null이면 해당 필드 변경 안 함)

```json
{
  "name": "홍길순",
  "phone": "010-9999-8888",
  "address": "서울시 서초구 강남대로 100",
  "detailAddress": "20층"
}
```

**Response** `200` — `UserUpdateResponse`

```json
{
  "success": true,
  "data": {
    "userId": 1,
    "name": "홍길순",
    "phone": "010-9999-8888",
    "address": "서울시 서초구 강남대로 100",
    "detailAddress": "20층"
  }
}
```

---

### PATCH /api/users/me/password — 비밀번호 변경

**권한** BUYER / SELLER

**Request**

```json
{
  "currentPassword": "Password123!",
  "newPassword": "NewPassword456!"
}
```

**Response** `200`

```json
{ "success": true, "data": null }
```

**에러** `MEMBER_002` 현재 비밀번호 불일치 | `MEMBER_003` 동일한 비밀번호

---

### DELETE /api/users/me — 회원 탈퇴

**권한** BUYER / SELLER

**Request**

```json
{ "password": "Password123!" }
```

**Response** `200`

```json
{ "success": true, "data": null }
```

> `user.status = WITHDRAWN`, Refresh Token 삭제
>