# Auth/Security 도메인 설계문서

## 1. 도메인 목적

Auth/Security 도메인은 사용자의 인증, 토큰 발급/재발급, 로그아웃, OAuth2 로그인, 기기별 세션 관리, 보안 이벤트 기록을 담당한다.

단순히 로그인 성공 여부만 처리하는 것이 아니라, 실제 운영 중 발생할 수 있는 다음 문제를 함께 해결하는 것을 목표로 한다.

- Access Token 탈취 후 만료 전까지 사용되는 문제
- Refresh Token 탈취 및 재사용 문제
- 여러 기기에서의 세션 관리 문제
- 사용자 정지/권한 변경 후 기존 토큰이 계속 살아있는 문제
- 무차별 로그인 시도와 신규 기기 등록 남용
- OAuth2 로그인과 자체 로그인 간 보안 정책 불일치
- 보안 이벤트 추적과 감사로그 관리

## 2. 주요 책임

| 책임 | 설명 |
|---|---|
| 자체 로그인 | 이메일/비밀번호 기반 인증 |
| OAuth2 로그인 | Kakao OAuth2 인증 후 내부 JWT 체계로 통합 |
| Access Token 발급 | 보호 API 접근용 단기 토큰 발급 |
| Refresh Token 발급 | Access Token 재발급용 장기 토큰 발급 |
| Refresh Token Rotation | 재발급 시 기존 RT 폐기 후 새 RT 발급 |
| 기기 관리 | `device_id` 기반 기기별 세션 관리 |
| tokenVersion 검증 | 보안 이벤트 발생 후 기존 AT 무효화 |
| Rate Limit | 로그인/재발급/기기등록 남용 방지 |
| 보안감사로그 | 인증 성공/실패, 의심 이벤트, 관리자 조치 기록 |

## 3. 전체 인증 흐름

### 3.1 일반 로그인

```text
로그인 요청
↓
IP/계정 기준 Rate Limit 검사
↓
이메일/비밀번호 검증
↓
사용자 상태 검증
↓
device_id 확인 또는 발급
↓
신규 기기 등록 제한 검사
↓
AT/RT 발급
↓
RT hash Redis 저장
↓
device_id 및 refresh_token 쿠키 설정
↓
로그인 성공 감사로그 기록
```

### 3.2 OAuth2 로그인

```text
OAuth2 인증 요청
↓
Kakao 인증 완료
↓
OAuth2 Success Handler 진입
↓
기존 사용자 조회 또는 신규 사용자 생성
↓
사용자 상태 검증
↓
device_id 확인 또는 발급
↓
신규 기기 등록/갱신
↓
AT/RT 발급
↓
프론트 콜백으로 리다이렉트
```

### 3.3 보호 API 접근

```text
Authorization: Bearer Access Token
↓
JWT 서명/만료 검증
↓
JTI blacklist 확인
↓
user-level cutoff 확인
↓
tokenVersion 검증
↓
사용자 상태 cache 검증
↓
SecurityContext 등록
```

### 3.4 Refresh Token 재발급

```text
refresh_token 쿠키 확인
↓
device_id 쿠키 확인
↓
RT 파싱 및 내부 device_id 확인
↓
Redis 저장 RT hash 조회
↓
요청 RT hash와 비교
↓
CAS Lua Script로 기존 RT hash를 새 RT hash로 교체
↓
새 AT/RT 발급
↓
재발급 성공 감사로그 기록
```

## 4. 토큰 정책

| 토큰 | 역할 | 만료 | 저장 위치 | 서버 저장 |
|---|---|---:|---|---|
| Access Token | 보호 API 접근 | 15분 | 클라이언트 | 원칙적으로 저장하지 않음 |
| Refresh Token | AT 재발급 | 7일 | HttpOnly Cookie | Redis에 hash 저장 |

Access Token에는 `userId`, `role`, `jti`, `ver` claim을 포함한다.

```text
JWT ver claim = User.tokenVersion
```

Refresh Token은 Redis에 원문으로 저장하지 않고 hash로 저장한다.

```text
refresh:{userId}:{deviceId} = hash(refreshToken)
```

## 5. Refresh Token Rotation

Refresh Token Rotation은 재발급 요청마다 기존 RT를 폐기하고 새로운 RT를 발급하는 방식이다.

### 선택 이유

고정 Refresh Token 방식에서는 RT가 탈취되면 만료 전까지 계속 재사용될 수 있다. RTR을 적용하면 이미 교체된 RT가 다시 사용되는 순간 탈취 또는 재사용 시도로 판단할 수 있다.

### 동시성 제어

동일한 RT로 동시에 여러 재발급 요청이 들어올 수 있으므로 Redis Lua Script를 사용해 비교와 교체를 원자적으로 처리한다.

```text
if stored_hash == request_hash:
    set new_hash
    return success
else:
    return fail
```

CAS 실패 시 `REFRESH_TOKEN_REUSE_DETECTED` 이벤트로 기록하고, 사용자 단위 세션 무효화를 수행한다.

## 6. device_id 기반 기기 관리

사용자는 여러 기기에서 로그인할 수 있으므로 RT를 사용자 단위가 아니라 기기 단위로 관리한다.

```text
devices:{userId} = ZSET(deviceId, lastUsedAt)
refresh:{userId}:{deviceId} = hash(refreshToken)
```

정책:

- 사용자당 최대 5개 기기 허용
- 6번째 기기 로그인 시 가장 오래된 기기 제거
- 제거된 기기의 RT 삭제
- 로그아웃 시 현재 기기의 RT 삭제
- 강제 로그아웃 시 모든 기기의 RT 삭제

## 7. tokenVersion 기반 Access Token 무효화

JWT는 stateless 구조이므로 한 번 발급된 AT를 서버가 직접 삭제하기 어렵다. 이를 보완하기 위해 User 엔티티에 `tokenVersion`을 둔다.

```text
AT 발급 시: JWT.ver = User.tokenVersion
API 요청 시: JWT.ver == 현재 User.tokenVersion
```

다음 상황에서 tokenVersion을 증가시킨다.

| 상황 | 이유 |
|---|---|
| 비밀번호 변경 | 기존 토큰 무효화 |
| 권한 변경 | 기존 권한 토큰 차단 |
| 사용자 정지 | 정지 사용자 API 접근 차단 |
| 강제 로그아웃 | 전체 세션 무효화 |
| RT 재사용 감지 | 탈취 의심 세션 전체 차단 |

이 선택은 JWT의 완전한 stateless 장점을 일부 포기하지만, 보안 이벤트 발생 시 기존 AT를 즉시 무효화하기 위한 의도적인 trade-off이다.

## 8. Rate Limit 설계

Auth/Security의 Rate Limit은 일반 API 트래픽 제어가 아니라 보안 정책에 가깝다.

적용 대상:

- 회원가입
- 로그인
- 신규 기기 등록
- Refresh Token 재발급
- 로그아웃

Redis Counter + TTL 기반 Fixed Window 계열 방식을 사용한다.

선택 이유:

- 로그인 실패 횟수 누적에 적합
- TTL 기반 자동 해제 가능
- 성공 시 counter 초기화가 쉬움
- Lua Script로 증가와 TTL 설정을 원자 처리 가능

Token Bucket/Bucket4j는 일반 API 트래픽 shaping에는 적합하지만, 로그인 실패 누적과 계정 잠금 정책에는 Counter + TTL 방식이 더 명확하다고 판단했다.

## 9. 보안감사로그

기록 대상:

| 이벤트 | 설명 |
|---|---|
| LOGIN_SUCCESS | 로그인 성공 |
| LOGIN_FAILED | 로그인 실패 |
| RATE_LIMIT_BLOCKED | Rate Limit 차단 |
| TOKEN_REFRESH_SUCCESS | 토큰 재발급 성공 |
| REFRESH_TOKEN_REUSE_DETECTED | RT 재사용 의심 |
| TOKEN_DEVICE_MISMATCH | RT와 device_id 불일치 |
| LOGOUT | 로그아웃 |
| USER_SUSPENDED | 사용자 정지 |
| USER_FORCE_LOGOUT | 강제 로그아웃 |

PII 최소화 정책:

| 항목 | 저장 방식 |
|---|---|
| IP | AES 암호화 + HMAC hash + prefix |
| User-Agent | HMAC hash |
| device_id | HMAC hash |
| Token | 저장 금지 |
| Cookie | 저장 금지 |
| Request Body | 저장 금지 |

감사로그 저장은 인증 흐름과 분리하기 위해 ApplicationEvent + Async 방식으로 처리한다.

## 10. 예외 처리

| 상황 | 처리 |
|---|---|
| 로그인 실패 | 실패 응답 및 감사로그 기록 |
| Rate Limit 초과 | 요청 차단 및 감사로그 기록 |
| RT 만료 | 재로그인 요구 |
| RT hash 불일치 | 재발급 거부 |
| RT 재사용 의심 | 전체 세션 무효화 |
| device_id 불일치 | 재발급 거부 및 감사로그 기록 |
| tokenVersion 불일치 | 인증 거부 |
| 정지 사용자 접근 | 인증 거부 |

## 11. 한계 및 후속 개선

| 한계 | 후속 개선 |
|---|---|
| 감사로그 비동기 큐 유실 가능성 | Kafka/Redis Stream 기반 내구성 강화 |
| RateLimit 정책값 환경별 분리 부족 | profile/config 기반 정책 분리 |
| Nginx 1차 Rate Limit 제한적 적용 | 경로별 Nginx Rate Limit 고도화 |
| 일반 조회 API traffic shaping 부족 | Token Bucket/Bucket4j 후속 적용 검토 |
| OAuth2와 자체 로그인 후처리 중복 가능성 | 공통 AuthPostProcessor 분리 |
