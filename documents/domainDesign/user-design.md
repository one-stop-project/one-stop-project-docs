# User/Admin 도메인 설계문서

## 1. 도메인 목적

User/Admin 도메인은 회원의 기본 정보, 권한, 상태, 주소/계좌 정보, 관리자 보안 조치 흐름을 담당한다.

Auth 도메인이 인증을 담당한다면, User 도메인은 인증 이후 서비스에서 사용할 사용자 상태와 권한의 기준 데이터를 관리한다.

## 2. 주요 책임

| 책임 | 설명 |
|---|---|
| 회원 정보 관리 | 이메일, 닉네임, 주소, 상세주소, 은행명, 계좌번호 등 관리 |
| 회원 상태 관리 | ACTIVE, SUSPENDED, WITHDRAWN 상태 관리 |
| 권한 관리 | BUYER, SELLER, ADMIN, SUPER_ADMIN 등 역할 관리 |
| 사용자 조회 | 내 정보 조회, 관리자 사용자 조회 |
| 사용자 정보 수정 | 프로필/주소/계좌 정보 변경 |
| 관리자 보안 조치 | 정지, 정지 해제, 강제 로그아웃 |
| tokenVersion 변경 | 비밀번호 변경, 권한 변경, 정지 시 기존 토큰 무효화 |

## 3. 사용자 상태 모델

| 상태 | 의미 |
|---|---|
| ACTIVE | 정상 이용 가능 |
| SUSPENDED | 관리자 또는 정책에 의해 이용 제한 |
| WITHDRAWN | 탈퇴 사용자 |

보호 API 요청 시 Auth 필터에서 사용자 상태를 검증한다. 사용자가 SUSPENDED 또는 WITHDRAWN이면 기존 Access Token이 있더라도 API 접근을 차단한다.

## 4. 권한 모델

| 권한 | 설명 |
|---|---|
| BUYER | 일반 구매자 |
| SELLER | 판매자 |
| ADMIN | 운영 관리자 |
| SUPER_ADMIN | 보안 조치 등 상위 관리자 |

일반 회원가입으로 ADMIN/SUPER_ADMIN 권한을 획득할 수 없도록 제한한다.

## 5. 주요 흐름

### 5.1 회원 정보 수정

```text
인증 사용자 요청
↓
현재 사용자 조회
↓
수정 가능 필드 검증
↓
주소/상세주소/계좌 정보 업데이트
↓
응답 반환
```

### 5.2 사용자 정지

```text
SUPER_ADMIN 요청
↓
대상 사용자 조회
↓
대상 role 검증
↓
User.status = SUSPENDED
↓
User.tokenVersion 증가
↓
userStatus cache evict
↓
전체 Refresh Token 삭제
↓
보안감사로그 기록
```

### 5.3 정지 해제

```text
SUPER_ADMIN 요청
↓
대상 사용자 조회
↓
User.status = ACTIVE
↓
userStatus cache evict
↓
정지 action 비활성화
↓
보안감사로그 기록
```

### 5.4 강제 로그아웃

```text
SUPER_ADMIN 요청
↓
대상 사용자 조회
↓
User.tokenVersion 증가
↓
tokenVersion cache evict
↓
user-level cutoff 등록
↓
전체 Refresh Token 삭제
↓
보안감사로그 기록
```

## 6. 데이터 모델

### User

| 필드 | 설명 |
|---|---|
| id | 사용자 식별자 |
| email | 로그인 이메일 |
| password | 암호화된 비밀번호 |
| role | 사용자 권한 |
| status | 사용자 상태 |
| tokenVersion | JWT 무효화용 버전 |
| address | 주소 |
| detailAddress | 상세주소 |
| bankName | 은행명 |
| accountNumber | 계좌번호 |
| createdAt/updatedAt | 생성/수정 시각 |

### UserSecurityAction

| 필드 | 설명 |
|---|---|
| id | 조치 식별자 |
| targetUserId | 대상 사용자 |
| actorUserId | 조치 관리자 |
| actionType | SUSPEND, UNSUSPEND, FORCE_LOGOUT |
| reasonCode | 조치 사유 코드 |
| reasonDetail | 상세 사유 |
| expiresAt | 정지 만료 시각 |
| active | 조치 활성 여부 |

## 7. 기술 선택 근거

### tokenVersion

사용자 상태나 권한이 변경되면 기존 JWT Access Token도 더 이상 신뢰할 수 없다. JWT는 stateless 구조이므로 직접 삭제할 수 없기 때문에 `tokenVersion`을 사용해 기존 토큰을 무효화한다.

### userStatus cache

매 요청마다 DB에서 사용자 상태를 조회하면 비용이 크다. 따라서 Redis 캐시를 사용하되, 상태 변경 시 반드시 evict하여 stale cache 문제를 방지한다.

### UserSecurityAction 분리

사용자 상태만 변경하면 “누가, 왜, 언제, 얼마 동안” 조치했는지 추적하기 어렵다. 관리자 조치 이력을 별도 모델로 분리해 감사 가능성을 확보한다.

## 8. 정합성/보안 고려사항

- 사용자 정지/해제 시 userStatus cache를 즉시 무효화한다.
- 정지/강제 로그아웃 시 Refresh Token을 전체 삭제한다.
- 정지/강제 로그아웃 시 tokenVersion을 증가시켜 기존 AT를 차단한다.
- SUPER_ADMIN 조치 API는 권한 검증을 Controller와 Service 양쪽에서 수행한다.
- 관리자 자기 자신에 대한 정지/권한 회수는 제한한다.
- ADMIN/SUPER_ADMIN 대상 조치는 별도 정책으로 제한한다.

## 9. 예외 처리

| 상황 | 처리 |
|---|---|
| 존재하지 않는 사용자 | USER_NOT_FOUND |
| 탈퇴 사용자 조치 | 요청 거부 |
| 권한 없는 관리자 요청 | FORBIDDEN |
| 자기 자신 정지 시도 | 요청 거부 |
| 보호된 관리자 계정 제재 시도 | 요청 거부 |
| 정지 사용자 로그인/접근 | AUTH_SUSPENDED |

## 10. 한계 및 후속 개선

| 한계 | 후속 개선 |
|---|---|
| 관리자 조치 정책이 코드에 고정될 수 있음 | 정책 테이블 또는 Rule 기반 분리 |
| 정지 만료 처리가 Lazy Evaluation 중심 | 배치 기반 만료 정리 추가 |
| 관리자 조치 감사 조회 기능 제한 | 보안감사로그 검색/필터 고도화 |
| 개인정보 변경 이력 추적 제한 | 개인정보 변경 감사로그 확장 |
