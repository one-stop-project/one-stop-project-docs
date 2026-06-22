## 1. 인증 (Auth)

### POST /api/auth/signup — 회원가입

**권한** 전체

**Request**

```json
{
  "email": "hong@example.com",
  "password": "Password123!",
  "name": "홍길동",
  "phone": "010-1234-5678",
  "address": "서울시 강남구 테헤란로 123",
  "role": "BUYER"
}
```

| 필드 | 타입 | 필수 | 제약 |
| --- | --- | --- | --- |
| email | String | ✅ | 이메일 형식, 중복 불가 |
| password | String | ✅ | 8자 이상, 영문+숫자+특수문자 |
| name | String | ✅ | 2~20자 |
| phone | String |  | 010-XXXX-XXXX |
| address | String |  | 기본 주소 |
| role | String | ✅ | BUYER / SELLER |

**Response** `201`

```json
{
  "success": true,
  "data": {
    "userId": 1,
    "email": "hong@example.com",
    "name": "홍길동",
    "role": "BUYER"
  }
}
```

**에러** `AUTH_001` 이메일 형식 오류 | `AUTH_002` 이메일 중복 | `AUTH_003` 비밀번호 규칙 위반

---

### POST /api/auth/login — 로그인

**권한** 전체

**Request**

```json
{
  "email": "hong@example.com",
  "password": "Password123!"
}
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGc...",
    "refreshToken": "eyJhbGc...",
    "expiresIn": 900,
    "user": {
      "userId": 1,
      "name": "홍길동",
      "role": "BUYER"
    }
  }
}
```

**에러** `AUTH_004` 이메일/비밀번호 불일치 | `AUTH_005` 정지된 계정

---

### POST /api/auth/refresh — 토큰 재발급

**권한** 전체

**Request**

```json
{ "refreshToken": "eyJhbGc..." }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGc...(new)",
    "expiresIn": 900
  }
}
```

**에러** `AUTH_010` Refresh Token 만료

---

### POST /api/auth/logout — 로그아웃

**권한** 로그인 사용자

**Response** `200`

```json
{ "success": true, "data": null }
```

> Redis에서 Refresh Token 삭제
>