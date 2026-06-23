## 1. 인증 (Auth)

> Access Token은 JSON 응답으로 반환하고, `refresh_token`·`device_id`는 HttpOnly 보안 쿠키로 설정한다. 클라이언트는 refresh/logout 시 쿠키를 자동 전송한다(요청 바디로 토큰을 받지 않는다).

### POST /api/auth/signup — 회원가입

**권한** 전체

**Request** `SignUpRequest`

```json
{
  "email": "hong@example.com",
  "password": "Password123!",
  "name": "홍길동",
  "phone": "010-1234-5678",
  "address": "서울시 강남구 테헤란로 123",
  "detailAddress": "10층 1001호",
  "role": "BUYER"
}
```

| 필드 | 타입 | 필수 | 제약 |
| --- | --- | --- | --- |
| email | String | ✅ | 이메일 형식, 중복 불가 |
| password | String | ✅ | 8자 이상, 영문+숫자+특수문자 |
| name | String | ✅ | 2~20자 |
| phone | String | ✅ | 010-XXXX-XXXX |
| address | String |  | 기본 주소 |
| detailAddress | String |  | 상세 주소 |
| role | String | ✅ | BUYER / SELLER |
| shopName | String | SELLER 필수 | 상호명 |
| businessNumber | String | SELLER 필수 | 사업자 등록번호 |
| bankName | String | SELLER 필수 | 은행명 |
| bankAccount | String | SELLER 필수 | 계좌번호 |

> `role=SELLER`이면 shopName·businessNumber·bankName·bankAccount 4개가 모두 필수다.

**Response** `201` — `SignUpResponse`

```json
{
  "success": true,
  "data": { "userId": 1, "email": "hong@example.com", "name": "홍길동", "role": "BUYER" }
}
```

**에러** `AUTH_002` 이메일 중복 | `SELLER_010`·`SELLER_011`·`SELLER_012`·`SELLER_013` 판매자 필드 누락(상호명/사업자번호/은행명/계좌번호) (이메일 형식·비밀번호 규칙·전화번호 형식 위반은 400 검증 에러)

---

### POST /api/auth/login — 로그인

**권한** 전체

**Request** `LoginRequest`

```json
{ "email": "hong@example.com", "password": "Password123!" }
```

**Response** `200` — `LoginResponse` (평면 구조. refresh_token·device_id는 응답 바디가 아닌 HttpOnly 쿠키로 내려간다)

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGc...",
    "expiresIn": 900,
    "userId": 1,
    "email": "hong@example.com",
    "name": "홍길동",
    "role": "BUYER"
  }
}
```

> `guest_cart_id` 쿠키가 있으면 비로그인 장바구니를 로그인 사용자 DB 장바구니로 병합한다(병합 실패는 로그인 실패로 이어지지 않음).

**에러** `AUTH_004` 이메일/비밀번호 불일치 | `AUTH_005` 정지된 계정 | `AUTH_006` 탈퇴한 계정 | `AUTH_014` 최대 기기 수 초과

---

### POST /api/auth/refresh — 토큰 재발급

**권한** 전체 | RTR(Refresh Token Rotation)

**Request** — 요청 바디 없음. 쿠키 `refresh_token`·`device_id`를 검증한다.

**Response** `200` — `TokenRefreshResponse` (새 refresh_token은 쿠키로 재설정)

```json
{
  "success": true,
  "data": { "accessToken": "eyJhbGc...(new)", "expiresIn": 900 }
}
```

**에러** `AUTH_010` 갱신 토큰 만료/누락 | `AUTH_020` 기기 정보 불일치 | `AUTH_012` 다른 기기 로그인으로 세션 만료

---

### POST /api/auth/oauth2/exchange — 소셜 로그인 코드 교환

**권한** 전체 | OAuth2 성공 후 발급된 일회용 code를 Access Token으로 교환 (`refresh_token`·`device_id`는 이미 쿠키에 설정됨)

**Request** `OAuth2ExchangeRequest`

```json
{ "code": "one-time-code" }
```

**Response** `200` — `TokenRefreshResponse`

```json
{
  "success": true,
  "data": { "accessToken": "eyJhbGc...", "expiresIn": 900 }
}
```

**에러** `AUTH_018` OAuth2 인증 처리 실패

---

### POST /api/auth/logout — 로그아웃

**권한** 전체(permitAll) — 만료된 Access Token으로도 호출 가능(쿠키 정리 목적)

**Request** — Authorization 헤더(Bearer, 선택) + `device_id` 쿠키

**Response** `200`

```json
{ "success": true, "data": null }
```

> Access Token 블랙리스트 등록 + 서버측 Refresh Token 제거 + `refresh_token`·`device_id` 쿠키 강제 만료.
