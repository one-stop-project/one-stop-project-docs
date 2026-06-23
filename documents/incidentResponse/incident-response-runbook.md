# 장애 시나리오 및 대응 Runbook

> One-Stop 프로젝트 운영 중 발생 가능한 주요 장애 상황을 정의하고, 장애 감지 기준, 영향 범위, 즉시 대응 절차, 복구 후 점검 항목을 정리한 문서입니다.  
> 본 문서는 단순 기능 구현을 넘어, 장애 발생 시 서비스 안정성과 데이터 정합성을 유지하기 위한 운영 대응 기준을 제공합니다.

---

## 1. 문서 목적

One-Stop은 인증, 사용자, 판매자, 상품, 장바구니, 주문, 결제, 포인트, 쿠폰, 배송, 리뷰, 알림, 검색, 구독, 관리자, AI 기능 등 여러 도메인이 연결된 커머스 플랫폼입니다.

특정 인프라나 외부 연동 서비스에 장애가 발생할 경우, 단순 API 실패를 넘어 로그인 불가, 결제 정합성 훼손, 포인트 잔액 불일치, 쿠폰 초과 발급, 알림 누락, 사용자 경험 저하로 이어질 수 있습니다.

따라서 본 문서는 다음을 목적으로 합니다.

- 장애 발생 시 빠르게 원인을 분류한다.
- 서비스 영향 범위를 파악한다.
- 즉시 적용 가능한 임시 대응 절차를 제공한다.
- 복구 후 데이터 정합성과 재발 방지 항목을 점검한다.
- 운영 관점에서 프로젝트의 안정성 설계 근거를 남긴다.

---

## 2. 장애 대응 기본 원칙

장애 대응은 다음 우선순위를 기준으로 수행합니다.

1. 사용자 영향 최소화
2. 데이터 정합성 보존
3. 장애 범위 격리
4. 서비스 복구
5. 원인 분석 및 재발 방지

특히 인증, 결제, 포인트, 쿠폰, 주문, 재고와 같이 정합성이 중요한 도메인은 단순히 요청을 성공 처리하는 것보다 **중복 처리 방지, 상태 불일치 방지, 실패 이력 보존**을 우선합니다.

---

## 3. 공통 점검 명령어

### 3.1 애플리케이션 상태 확인

```bash
sudo systemctl status one-stop-main --no-pager
sudo systemctl status one-stop-dummy --no-pager
```

### 3.2 애플리케이션 로그 확인

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager
sudo journalctl -u one-stop-dummy -n 100 --no-pager
```

### 3.3 Nginx 상태 확인

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo journalctl -u nginx -n 100 --no-pager
```

### 3.4 Docker 인프라 확인

```bash
docker ps
docker logs one-stop-mysql --tail=100
docker logs one-stop-redis --tail=100
docker logs kafka --tail=100
docker logs zookeeper --tail=100
```

### 3.5 주요 API 헬스 체크

```bash
curl -Ik https://onestop1.duckdns.org
curl -Ik https://onestop1.duckdns.org/api/health
curl -Ik https://onestop1.duckdns.org/oauth2/authorization/kakao
```

### 3.6 라운드로빈 서버별 응답 확인

```bash
for i in {1..10}; do
  curl -skI https://onestop1.duckdns.org/oauth2/authorization/kakao | grep -i "redirect_uri"
done
```

---

# Part 1. 공통 인프라 및 운영 장애

---

## 4. 장애 시나리오 1: 애플리케이션 서버 장애

### 상황

Spring Boot 애플리케이션이 비정상 종료되거나, systemd 상에서 `failed`, `deactivating`, `inactive` 상태가 되는 경우입니다.

### 감지 기준

```bash
sudo systemctl status one-stop-main --no-pager
sudo systemctl status one-stop-dummy --no-pager
```

다음 상태가 확인되면 장애로 판단합니다.

```text
Active: failed
Active: inactive
Active: deactivating (stop-sigterm)
```

### 영향 범위

- API 요청 실패
- 로그인, 주문, 결제, 포인트 처리 불가
- Nginx upstream에 장애 서버가 포함된 경우 간헐적 5xx 발생
- 라운드로빈 구성 시 요청마다 성공/실패가 반복될 수 있음

### 즉시 대응

Graceful shutdown이 오래 걸리는 경우 다음 명령어로 강제 정리 후 재기동합니다.

```bash
sudo systemctl kill -s SIGKILL one-stop-main
sudo systemctl reset-failed one-stop-main
sudo systemctl start one-stop-main
sudo systemctl status one-stop-main --no-pager
```

Dummy 서버의 경우 서비스명을 변경합니다.

```bash
sudo systemctl kill -s SIGKILL one-stop-dummy
sudo systemctl reset-failed one-stop-dummy
sudo systemctl start one-stop-dummy
sudo systemctl status one-stop-dummy --no-pager
```

### 복구 확인

```bash
curl -Ik https://onestop1.duckdns.org
curl -Ik https://onestop1.duckdns.org/api/health
```

### 재발 방지

- Graceful shutdown timeout 조정
- Kafka consumer 종료 지연 여부 확인
- systemd restart policy 설정
- 애플리케이션 메모리 사용량 및 GC 로그 확인
- 배포 전 smoke test 추가

---

## 5. 장애 시나리오 2: Nginx 라우팅 장애

### 상황

Nginx location 설정 오류로 인해 프론트 요청이 백엔드로 전달되거나, 백엔드 요청이 프론트로 전달되는 경우입니다.

OAuth2 장애 사례에서는 `/oauth2/callback`이 프론트가 아닌 백엔드로 전달되어 500 오류가 발생했습니다.

### 감지 기준

```bash
curl -Ik "https://onestop1.duckdns.org/oauth2/callback?code=test" | egrep -i "HTTP/|content-type|location|server"
```

정상 응답:

```text
HTTP/2 200
content-type: text/html
```

비정상 응답:

```text
content-type: application/json
```

### 영향 범위

- React Router 기반 페이지 접근 실패
- OAuth2 로그인 후 콜백 처리 실패
- 특정 경로에서 404 또는 500 발생
- 프론트/백엔드 경계가 불명확해져 디버깅 난이도 증가

### 즉시 대응

Nginx 설정에서 `/oauth2` 전체를 백엔드로 보내지 않도록 수정합니다.

정상 라우팅 기준은 다음과 같습니다.

```nginx
location ^~ /oauth2/authorization/ {
    proxy_pass http://one-stop-backend;
}

location ~ ^/(api|v3|images|swagger-ui|login/oauth2)(/|$) {
    proxy_pass http://one-stop-backend;
}

location / {
    root /var/www/frontend;
    try_files $uri $uri/ /index.html;
}
```

### 적용

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 복구 확인

```bash
curl -Ik "https://onestop1.duckdns.org/oauth2/callback?code=test" | egrep -i "HTTP/|content-type"
curl -Ik "https://onestop1.duckdns.org/oauth2/authorization/kakao" | egrep -i "HTTP/|location"
```

### 재발 방지

- `/oauth2/authorization/**`와 `/oauth2/callback`의 책임 분리 문서화
- Nginx 설정 변경 후 필수 curl 체크리스트 운영
- 프론트 라우트와 백엔드 API 경로 네이밍 규칙 분리

---

## 6. 장애 시나리오 3: OAuth2 로그인 장애

### 상황

카카오 로그인 과정에서 redirect URI 불일치, Nginx forwarded header 누락, Spring profile 설정 오류 등으로 OAuth2 인증이 실패하는 경우입니다.

### 감지 기준

```bash
curl -Ik https://onestop1.duckdns.org/oauth2/authorization/kakao | grep -i location
```

정상 응답에는 다음 값이 포함되어야 합니다.

```text
redirect_uri=https://onestop1.duckdns.org/login/oauth2/code/kakao
```

비정상 예시는 다음과 같습니다.

```text
redirect_uri=http://onestop1.duckdns.org/login/oauth2/code/kakao
redirect_uri=http://127.0.0.1:8080/login/oauth2/code/kakao
redirect_uri=http://10.0.xx.xx:8080/login/oauth2/code/kakao
```

### 영향 범위

- 카카오 로그인 불가
- 신규 사용자 유입 차단
- 기존 사용자도 소셜 로그인 기반 접근 불가
- 프론트 콜백 페이지에서 400 또는 500 오류 발생

### 즉시 대응

운영 설정 파일의 redirect URI를 확인합니다.

```bash
grep -n "redirect-uri" /app/application-prod.yml
```

정상 설정:

```yaml
redirect-uri: "https://onestop1.duckdns.org/login/oauth2/code/kakao"
```

수정 후 애플리케이션을 재기동합니다.

```bash
sudo systemctl restart one-stop-main
sudo systemctl restart one-stop-dummy
```

Graceful shutdown이 지연되면 다음과 같이 강제 재기동합니다.

```bash
sudo systemctl kill -s SIGKILL one-stop-main
sudo systemctl reset-failed one-stop-main
sudo systemctl start one-stop-main
```

### 복구 확인

```bash
for i in {1..10}; do
  curl -skI https://onestop1.duckdns.org/oauth2/authorization/kakao | grep -i "redirect_uri"
done
```

모든 응답에서 HTTPS redirect URI가 나와야 합니다.

### 재발 방지

- 운영 환경에서는 `{baseUrl}` 대신 명시적 HTTPS redirect URI 사용
- Kakao Developers에 등록된 redirect URI와 운영 설정 일치 여부 확인
- main/dummy 서버 설정 파일 동기화
- Nginx forwarded header 설정 유지

---

## 7. 장애 시나리오 4: Redis 장애

### 상황

Redis가 중단되거나 연결 불가 상태가 되어 Refresh Token, Rate Limit, Device ID, 블랙리스트, 쿠폰 발급 등의 Redis 기반 기능이 실패하는 경우입니다.

### 영향 범위

- Refresh Token Rotation 실패
- 로그아웃 처리 실패 가능
- 로그인 Rate Limit 무력화 또는 과도한 차단
- Device ID 기반 다중 기기 관리 실패
- 쿠폰 발급 Lua Script 실패
- Redis 기반 캐시 조회 실패

### 감지 기준

```bash
docker ps | grep redis
docker logs one-stop-redis --tail=100
```

애플리케이션 로그에서 다음 유형의 오류를 확인합니다.

```text
RedisConnectionFailureException
Redis command timed out
Unable to connect to Redis
```

### 즉시 대응

Redis 컨테이너 상태를 확인합니다.

```bash
docker restart one-stop-redis
docker logs one-stop-redis --tail=100
```

애플리케이션이 Redis 연결을 회복했는지 확인합니다.

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i redis
```

### 서비스 정책

Redis 장애 시 기능별 정책을 구분합니다.

- 인증 토큰 검증: 보수적으로 실패 처리
- 로그인 Rate Limit: Fail-Closed 또는 제한적 Fail-Open 정책 검토
- 일반 API Rate Limit: 서비스 가용성을 위해 제한적 Fail-Open 가능
- 쿠폰 발급: 중복 발급 및 수량 정합성 보장을 위해 실패 처리
- 캐시 조회: DB fallback 가능 시 fallback 처리

### 복구 후 점검

- Refresh Token 재발급 성공 여부
- 로그인 Rate Limit 정상 동작 여부
- 쿠폰 중복 발급 여부
- Redis key TTL 정상 여부
- Redis memory 사용량 확인

### 재발 방지

- Redis 장애 알림 추가
- Redis connection timeout 및 retry 정책 점검
- Redis Lua Script 실패 로그 분리
- 인증/쿠폰처럼 정합성 중요한 기능은 Redis 장애 시 실패 처리 기준 명확화

---

## 8. 장애 시나리오 5: MySQL 장애

### 상황

MySQL 컨테이너 또는 DB 연결이 실패하여 주요 도메인 데이터 조회 및 저장이 불가능한 경우입니다.

### 영향 범위

- 로그인 사용자 조회 실패
- 주문 생성 실패
- 결제 상태 저장 실패
- 포인트 잔액 변경 실패
- 쿠폰 발급 이력 저장 실패
- 관리자 기능 사용 불가

### 감지 기준

```bash
docker ps | grep mysql
docker logs one-stop-mysql --tail=100
```

애플리케이션 로그에서 다음 오류를 확인합니다.

```text
Communications link failure
Connection refused
SQLTransientConnectionException
HikariPool
```

### 즉시 대응

MySQL 컨테이너 상태를 확인하고 재시작합니다.

```bash
docker restart one-stop-mysql
docker logs one-stop-mysql --tail=100
```

애플리케이션 DB 연결 회복 여부를 확인합니다.

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "hikari\|mysql\|sql"
```

### 복구 후 점검

- 주문 상태가 PENDING_PAYMENT에 머문 건 없는지 확인
- 결제 승인 후 내부 저장 실패 건 확인
- 포인트 차감/적립 이력 불일치 여부 확인
- 배치 작업 중단 여부 확인
- 관리자 감사 로그 누락 여부 확인

### 재발 방지

- HikariCP pool 설정 점검
- Slow Query 및 Lock wait 로그 확인
- DB 백업 정책 수립
- 결제/포인트/쿠폰 도메인 정합성 검증 쿼리 준비

---

## 9. 장애 시나리오 6: Kafka 장애

### 상황

Kafka 또는 Zookeeper 장애로 인해 결제 승인 후 알림 발행, 이벤트 기반 처리, 비동기 메시지 소비가 실패하는 경우입니다.

### 영향 범위

- 결제 승인 이벤트 후 알림 누락
- 비동기 후속 처리 지연
- Consumer lag 증가
- 애플리케이션 종료 시 Kafka consumer shutdown 지연 가능

### 감지 기준

```bash
docker ps | grep kafka
docker logs kafka --tail=100
docker logs zookeeper --tail=100
```

애플리케이션 로그에서 다음 오류를 확인합니다.

```text
KafkaException
TimeoutException
Consumer stopped
Broker may not be available
```

### 즉시 대응

Kafka 및 Zookeeper 상태를 확인하고 재시작합니다.

```bash
docker restart zookeeper
docker restart kafka
```

애플리케이션 consumer가 재연결되는지 확인합니다.

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "kafka\|consumer"
```

### 복구 후 점검

- 결제 승인 후 알림 누락 건 확인
- Consumer lag 확인
- 동일 이벤트 중복 소비 여부 확인
- 알림 이력 테이블과 이벤트 처리 상태 비교

### 재발 방지

- Kafka consumer idempotency 보장
- 실패 이벤트 재처리 전략 마련
- DLQ 또는 실패 이벤트 테이블 도입 검토
- 애플리케이션 종료 시 Kafka consumer graceful shutdown timeout 조정

---

## 10. 장애 시나리오 7: AI 기능 장애

### 상황

AI API 또는 AI 연동 기능 장애로 인해 리뷰 요약, 상품 추천, 판매자 보조 기능 등 AI 기능이 실패하는 경우입니다.

### 영향 범위

- AI 요약/추천 기능 사용 불가
- 응답 지연으로 API 전체 응답 시간 증가
- 외부 API 비용 증가 가능
- 핵심 구매/주문 기능과 장애가 전파될 수 있음

### 감지 기준

- AI API timeout 증가
- 429 또는 5xx 응답 증가
- AI 응답 파싱 실패
- AI 기능 호출 API의 P95/P99 지연 증가

### 즉시 대응

- AI 기능을 비핵심 기능으로 분리하여 graceful degradation 적용
- AI 호출 실패 시 기본 응답 또는 기존 캐시 응답 반환
- AI API timeout을 짧게 제한
- 핵심 주문/결제 흐름에서는 AI 호출을 동기 처리하지 않음

### 복구 후 점검

- 실패 요청 수
- 재시도 횟수
- API 비용 증가 여부
- 사용자 응답 지연 구간

### 재발 방지

- AI 호출 timeout, retry, fallback 정책 문서화
- 자주 요청되는 요약 결과 캐싱
- AI 기능과 핵심 거래 기능의 장애 격리
- 사용량 제한 및 비용 알림 설정

---

## 11. 장애 시나리오 8: 모니터링 및 로그 수집 장애

### 상황

Prometheus, Grafana, 애플리케이션 로그, 감사 로그가 정상 수집되지 않아 장애 원인 분석이 어려워지는 경우입니다.

### 영향 범위

- 장애 발생 시 원인 파악 지연
- 성능 저하 감지 실패
- 보안 이벤트 추적 불가
- 장애 대응 회고 작성 어려움

### 감지 기준

- Grafana 대시보드 데이터 공백
- Prometheus target down
- journalctl 로그 미출력
- 감사 로그 테이블 적재 중단

### 즉시 대응

```bash
docker ps | grep prometheus
docker ps | grep grafana
docker logs prometheus --tail=100
docker logs grafana --tail=100
```

애플리케이션 로그 확인:

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager
```

### 복구 후 점검

- Prometheus target 상태
- Grafana 대시보드 데이터 표시 여부
- 애플리케이션 로그 출력 여부
- 감사 로그 저장 여부

### 재발 방지

- 모니터링 컨테이너 health check 추가
- 로그 보존 기간 정책 수립
- 주요 에러 로그 알림 추가
- TraceId 기반 요청 추적 강화

---

# Part 2. 인증/보안/User 도메인 장애

---

## 12. 장애 시나리오 9: 보안 이벤트 및 비정상 로그인 시도

### 상황

특정 IP 또는 계정에 대해 로그인 실패가 반복되거나, Refresh Token 재사용, 비정상 device_id 접근, 권한 없는 API 접근이 반복되는 경우입니다.

### 영향 범위

- 계정 탈취 시도
- Refresh Token 탈취 가능성
- 관리자 API 접근 시도
- Rate Limit 우회 시도

### 감지 기준

- 로그인 실패 횟수 급증
- 동일 IP에서 다수 계정 로그인 시도
- 동일 계정에 대한 다중 IP 로그인 시도
- Refresh Token Rotation 실패 증가
- AUTH_010, AUTH_007 등 인증 예외 증가

### 즉시 대응

- 해당 IP 또는 계정 기준 Rate Limit 상태 확인
- 필요 시 계정 tokenVersion 증가로 전체 토큰 무효화
- 의심 device_id의 Refresh Token 삭제
- 보안 감사 로그 확인

### 복구 후 점검

- 비정상 로그인 성공 여부
- 의심 계정의 세션 무효화 여부
- 관리자 권한 변경 이력
- 보안 감사 로그 누락 여부

### 재발 방지

- 로그인 Rate Limit 정책 유지
- device_id 기반 Refresh Token 관리
- Token Versioning 기반 강제 로그아웃 지원
- 보안 감사 로그 적재 및 검색 가능성 개선

---

## 13. 장애 시나리오 10: 사용자 계정 상태 장애

### 상황

회원 탈퇴, 계정 비활성화, 권한 변경, 비밀번호 변경 후 인증 상태가 정상적으로 반영되지 않는 경우입니다.

### 영향 범위

- 탈퇴 또는 비활성 사용자가 계속 API 사용 가능
- 비밀번호 변경 후 기존 토큰이 계속 유효
- 권한 변경 후 이전 권한으로 접근 가능
- 사용자 정보 수정 결과가 인증 객체와 불일치

### 감지 기준

- inactive user의 API 접근 성공
- tokenVersion 불일치 로그 증가
- 권한 변경 후 403/401 오류 증가
- 사용자 정보 수정 후 JWT claim과 DB 값 불일치

### 즉시 대응

사용자 상태 확인:

```sql
SELECT id, email, role, active, token_version, updated_at
FROM users
WHERE id = {userId};
```

필요 시 tokenVersion 증가로 기존 토큰을 무효화합니다.

```sql
UPDATE users
SET token_version = token_version + 1
WHERE id = {userId};
```

### 복구 후 점검

- 기존 Access Token 무효화 여부
- Refresh Token 삭제 여부
- 사용자 active 상태 반영 여부
- 권한 변경 후 API 접근 제어 정상 여부

### 재발 방지

- 비밀번호 변경/권한 변경/탈퇴 시 tokenVersion 증가 보장
- Refresh Token 저장소 삭제 처리
- 인증 필터에서 active 상태 검증
- 사용자 상태 변경 통합 테스트 보강

---

## 14. 장애 시나리오 11: Refresh Token Rotation 장애

### 상황

Refresh Token Rotation 과정에서 Redis 저장 값과 요청 토큰이 일치하지 않거나, 동시 요청으로 CAS 교체에 실패하는 경우입니다.

### 영향 범위

- 사용자가 재로그인을 요구받을 수 있음
- 탈취된 Refresh Token 재사용 시도 차단
- 모바일/브라우저 중복 요청 시 간헐적 인증 실패
- 정상 사용자 UX 저하 가능성

### 감지 기준

- AUTH_010, AUTH_007 증가
- `/api/auth/refresh` 401 증가
- 동일 userId/deviceId에 대한 refresh 요청 동시 발생
- Redis refresh key TTL 비정상

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "refresh\|AUTH_010\|AUTH_007"
```

의심 기기 세션을 삭제하거나 전체 토큰 버전을 증가시킵니다.

```sql
UPDATE users
SET token_version = token_version + 1
WHERE id = {userId};
```

### 복구 후 점검

- 재로그인 후 Refresh Token 정상 저장 여부
- device_id 쿠키 존재 여부
- Redis refresh key TTL 정상 여부
- 동일 기기에서 재발 여부

### 재발 방지

- Refresh Token 해시 선검증 및 CAS 실패 원인 구분
- 프론트에서 refresh 중복 요청 방지
- refresh 실패 시 사용자 재로그인 플로우 명확화
- device_id 쿠키 정책 문서화

---

## 15. 장애 시나리오 12: Device ID 및 다중 기기 관리 장애

### 상황

device_id 쿠키가 누락되거나, Redis ZSET 기반 기기 등록/추방 로직이 실패하여 다중 기기 제한이 정상 동작하지 않는 경우입니다.

### 영향 범위

- Refresh Token 저장 key 불일치
- 동일 사용자 다중 기기 제한 무력화
- 신규 기기 로그인 실패
- 기존 기기 세션이 의도치 않게 만료될 수 있음

### 감지 기준

- device_id 쿠키 누락
- Redis `devices:{userId}` ZSET 누락 또는 과다
- 5대 초과 로그인 가능
- 신규 기기 등록 API 또는 로그인 성공 후 예외 발생

### 즉시 대응

- 사용자 Redis device key 확인
- 의심 device_id의 refresh key 삭제
- 필요 시 사용자 tokenVersion 증가로 전체 세션 무효화

### 복구 후 점검

- 신규 로그인 시 device_id 쿠키 발급 여부
- 5대 초과 시 LRU 추방 여부
- OAuth2 로그인에서도 device context 등록 여부
- 로그아웃 시 해당 device refresh token 삭제 여부

### 재발 방지

- Redis Lua Script 문법 검증 테스트 유지
- 신규 기기 등록과 Refresh Token 저장 순서 점검
- OAuth2 성공 핸들러의 device context 처리 테스트 보강
- device_id 쿠키 path, maxAge 정책 명확화

---

## 16. 장애 시나리오 13: 관리자 기능 장애

### 상황

관리자 권한으로 사용자 제재, 상품 관리, 쿠폰 관리, 감사 로그 조회 등의 기능을 수행하는 과정에서 오류가 발생하는 경우입니다.

### 영향 범위

- 운영자가 장애 상황을 직접 조치할 수 없음
- 비정상 사용자 또는 상품 제재 지연
- 쿠폰/포인트 수동 보정 불가
- 관리자 감사 이력 누락 가능성

### 감지 기준

- 관리자 API 403 또는 500 증가
- 관리자 권한 사용자의 tokenVersion 불일치
- 관리자 감사 로그 저장 실패
- 권한 변경 후 기존 토큰이 계속 유효함

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "admin"
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "audit"
```

관리자 권한 및 상태 확인:

```sql
SELECT id, email, role, active, token_version
FROM users
WHERE role IN ('ADMIN', 'SUPER_ADMIN');
```

### 복구 후 점검

- 관리자 로그인 가능 여부
- 관리자 API 권한 검증 정상 여부
- 감사 로그 저장 여부
- 권한 변경 시 tokenVersion 증가 여부

### 재발 방지

- 관리자 API 권한 테스트 강화
- 권한 변경 시 사용자 토큰 일괄 무효화
- 관리자 감사 로그는 비동기 유실 가능성 최소화
- 관리자 기능에 대한 별도 모니터링 추가

---

# Part 3. 주문/결제/배송 도메인 장애

---

## 17. 장애 시나리오 14: 주문 생성 장애

### 상황

사용자가 상품을 주문하려고 할 때 주문 생성이 실패하거나, 주문은 생성되었지만 주문 상세, 결제 대기 상태, 재고 차감 상태가 불일치하는 경우입니다.

### 영향 범위

- 사용자가 상품을 구매할 수 없음
- 주문은 생성되었지만 결제 페이지로 이동하지 못함
- 주문 상태가 `PENDING_PAYMENT`에 머무름
- 재고 차감 또는 쿠폰/포인트 선점 상태와 주문 상태가 불일치할 수 있음

### 감지 기준

- 주문 생성 API 4xx/5xx 증가
- 주문 상태가 장시간 `PENDING_PAYMENT`에 머무는 데이터 증가
- 동일 사용자의 중복 주문 생성
- 주문 생성 후 결제 요청이 이어지지 않는 케이스 증가

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "order"
sudo journalctl -u one-stop-dummy -n 100 --no-pager | grep -i "order"
```

DB에서 주문 상태를 확인합니다.

```sql
SELECT id, user_id, status, created_at
FROM orders
WHERE status = 'PENDING_PAYMENT'
ORDER BY created_at DESC;
```

주문 생성 실패가 특정 상품에 집중되어 있다면 상품 재고, 판매 상태, 상품 노출 상태를 함께 확인합니다.

### 복구 후 점검

- 주문 상태와 결제 상태 일치 여부
- 동일 사용자 중복 주문 여부
- 재고 차감 여부
- 쿠폰/포인트 사용 예약 상태 여부
- 실패 주문에 대한 사용자 안내 여부

### 재발 방지

- 주문 생성과 결제 요청 사이의 상태 전이 명확화
- 주문 생성 API 멱등성 검토
- 장시간 `PENDING_PAYMENT` 주문 정리 배치 도입
- 주문 생성 실패 로그에 userId, itemId, orderId 포함

---

## 18. 장애 시나리오 15: 결제 승인 후 내부 처리 실패

### 상황

외부 PG 결제는 성공했지만, 내부 주문 상태 변경, 결제 이력 저장, 포인트 차감, 쿠폰 사용 처리 중 실패가 발생하는 경우입니다.

### 영향 범위

- 사용자는 결제 완료로 인식하지만 내부 주문은 PENDING_PAYMENT 상태
- 결제 금액과 주문 상태 불일치
- 포인트/쿠폰 사용 이력 불일치
- 환불 또는 보상 트랜잭션 필요

### 감지 기준

- PG 승인 성공 로그는 있으나 내부 payment 상태가 PAID가 아님
- 주문 상태가 PENDING_PAYMENT에 장시간 머무름
- 결제 승인 시간과 내부 저장 시간이 불일치
- 결제 실패 예외 로그 발생

### 즉시 대응

1. PG 결제 승인 여부 확인
2. 내부 payment/order 상태 확인
3. 내부 저장 실패라면 보상 트랜잭션 또는 수동 환불 여부 판단
4. 중복 처리 방지를 위해 idempotency key 확인

### 복구 후 점검

- 주문 상태와 결제 상태 일치 여부
- 포인트 사용/복구 이력 일치 여부
- 쿠폰 사용 여부 일치
- 환불 이력 저장 여부
- 사용자에게 노출된 주문 상태 확인

### 재발 방지

- 결제 승인과 내부 상태 변경의 멱등성 강화
- 결제 상태 검증 API 운영
- 보상 트랜잭션 로그 테이블 관리
- 결제 실패 케이스 통합 테스트 강화

---

## 19. 장애 시나리오 16: 배송 상태 변경 장애

### 상황

주문 후 배송 상태가 정상적으로 변경되지 않거나, 배송 이력 저장이 누락되는 경우입니다.

### 영향 범위

- 사용자가 배송 상태를 확인할 수 없음
- 판매자 또는 관리자가 배송 처리를 추적하기 어려움
- 주문 상태와 배송 상태 불일치
- 환불/취소 가능 상태 판단 오류

### 감지 기준

- 배송 상태 변경 API 5xx 증가
- 주문 상태는 배송중인데 배송 이력이 없음
- 배송 완료 후 주문 상태가 변경되지 않음
- delivery_history 누락

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "delivery"
```

배송 상태와 이력을 확인합니다.

```sql
SELECT id, order_id, status, updated_at
FROM deliveries
ORDER BY updated_at DESC
LIMIT 50;
```

```sql
SELECT delivery_id, status, created_at
FROM delivery_history
ORDER BY created_at DESC
LIMIT 50;
```

### 복구 후 점검

- 주문 상태와 배송 상태 일치 여부
- 배송 이력 생성 여부
- 배송 완료 후 구매 확정 가능 여부
- 취소/환불 가능 상태 정상 여부

### 재발 방지

- 배송 상태 머신 정의
- 상태 변경 시 이력 저장 트랜잭션 보장
- 잘못된 상태 전이 차단
- 배송 상태 변경 테스트 보강

---

# Part 4. 상품/재고/장바구니/판매자 도메인 장애

---

## 20. 장애 시나리오 17: 재고 차감 장애

### 상황

동시에 다수 사용자가 같은 상품을 주문하면서 재고가 음수가 되거나, 결제 실패 후 재고 복구가 누락되는 경우입니다.

### 영향 범위

- 초과 판매 발생
- 주문은 성공했지만 실제 재고 부족
- 결제 실패 또는 취소 후 재고 복구 누락
- 판매자 재고 관리 화면과 실제 주문 가능 수량 불일치

### 감지 기준

- 상품 재고가 0 미만으로 감소
- 주문 수량 합계가 상품 재고보다 큼
- 결제 실패 주문인데 재고가 복구되지 않음
- 재고 관련 optimistic lock 예외 증가

### 즉시 대응

특정 상품의 재고 상태를 확인합니다.

```sql
SELECT id, name, stock_quantity, updated_at
FROM items
WHERE id = {itemId};
```

주문 수량과 비교합니다.

```sql
SELECT item_id, SUM(quantity) AS ordered_quantity
FROM order_items
WHERE item_id = {itemId}
GROUP BY item_id;
```

### 복구 후 점검

- 상품 재고 수량 정상 여부
- 주문 상태별 재고 반영 여부
- 결제 실패/취소 주문의 재고 복구 여부
- 초과 판매 발생 시 사용자 안내 또는 주문 취소 처리 여부

### 재발 방지

- 재고 차감 경로에 비관적 락 또는 Redis 기반 원자 처리 검토
- 주문/결제 실패 시 재고 복구 이벤트 보장
- 재고 변경 이력 테이블 도입 검토
- 동시 주문 부하 테스트 정기 수행

---

## 21. 장애 시나리오 18: 장바구니 병합 장애

### 상황

비회원 장바구니와 로그인 사용자 장바구니를 병합하는 과정에서 상품이 중복되거나, 기존 장바구니 항목이 사라지는 경우입니다.

### 영향 범위

- 로그인 후 장바구니 상품 누락
- 동일 상품 중복 표시
- 수량 합산 오류
- 사용자가 의도하지 않은 상품을 주문할 가능성 발생

### 감지 기준

- 로그인 직후 장바구니 API 오류 증가
- 동일 userId + itemId 장바구니 중복 데이터 발생
- guest_cart_id 쿠키는 존재하지만 병합 후 데이터가 사라짐
- 장바구니 수량이 비정상적으로 증가

### 즉시 대응

사용자 장바구니와 비회원 장바구니 식별자를 확인합니다.

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "cart"
```

중복 데이터 확인 예시:

```sql
SELECT user_id, item_id, COUNT(*)
FROM cart_items
GROUP BY user_id, item_id
HAVING COUNT(*) > 1;
```

### 복구 후 점검

- userId 기준 장바구니 항목 정상 여부
- 동일 상품 수량 합산 여부
- guest_cart_id 쿠키 삭제 또는 유지 정책 확인
- 병합 후 장바구니 조회 API 정상 여부

### 재발 방지

- userId + itemId 유니크 제약 검토
- 장바구니 병합 로직 멱등성 보장
- 로그인 성공 후 CartMergeService 테스트 보강
- guest_cart_id 쿠키 만료 및 삭제 정책 명확화

---

## 22. 장애 시나리오 19: 판매자 상품 관리 장애

### 상황

판매자가 상품을 등록, 수정, 삭제하거나 판매 상태를 변경하는 과정에서 오류가 발생하는 경우입니다.

### 영향 범위

- 판매자 상품 등록 불가
- 상품 정보 수정 반영 지연
- 품절/판매중지 상품이 계속 노출
- 구매 가능한 상품과 실제 판매자 관리 상태 불일치

### 감지 기준

- 판매자 상품 등록 API 5xx 증가
- 이미지 업로드 실패 증가
- 상품 상태 변경 후 조회 결과 불일치
- 판매자 권한 검증 실패 로그 증가

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "seller"
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "item"
```

상품 상태를 확인합니다.

```sql
SELECT id, seller_id, name, status, stock_quantity, updated_at
FROM items
WHERE seller_id = {sellerId}
ORDER BY updated_at DESC;
```

### 복구 후 점검

- 상품 등록/수정 API 정상 여부
- 판매 상태 변경 반영 여부
- 상품 검색/목록 노출 여부
- 판매자 권한 검증 정상 여부

### 재발 방지

- 판매자 권한 검증 로직 테스트 보강
- 상품 상태 변경 이력 관리
- 이미지 업로드 실패 시 상품 등록 롤백 기준 명확화
- 상품 목록 캐시 사용 시 캐시 무효화 정책 추가

---

## 23. 장애 시나리오 20: 판매자 대시보드 장애

### 상황

판매자가 주문 현황, 매출, 리뷰, 상품 통계를 조회하는 대시보드 API에서 오류 또는 성능 저하가 발생하는 경우입니다.

### 영향 범위

- 판매자가 주문/매출 현황을 확인할 수 없음
- 판매자 운영 판단 지연
- 특정 기간 조회 시 DB 부하 증가
- Page 응답 구조 또는 집계 쿼리 오류로 프론트 표시 실패

### 감지 기준

- 판매자 대시보드 API 응답 지연 증가
- 특정 sellerId 조회 시 500 발생
- 집계 쿼리 timeout 발생
- 프론트에서 매출/주문 통계가 빈 값으로 표시됨

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "dashboard"
```

최근 주문/매출 집계 대상 데이터를 확인합니다.

```sql
SELECT seller_id, COUNT(*) AS order_count
FROM order_items
GROUP BY seller_id
ORDER BY order_count DESC;
```

### 복구 후 점검

- 판매자별 주문 수 조회 정상 여부
- 기간별 매출 집계 정상 여부
- 리뷰 통계 조회 정상 여부
- 페이지네이션 응답 구조 정상 여부

### 재발 방지

- 대시보드 집계 쿼리 인덱스 점검
- 기간 조건 필수화
- 대량 집계는 캐시 또는 배치 집계 테이블 검토
- Spring Data Page 직접 노출 대신 응답 DTO 표준화

---

# Part 5. 포인트/쿠폰/구독 도메인 장애

---

## 24. 장애 시나리오 21: 포인트 정합성 장애

### 상황

포인트 차감, 적립, 만료, 환불 처리 중 동시성 충돌이나 배치 누락으로 사용자 포인트 잔액과 이력이 불일치하는 경우입니다.

### 영향 범위

- 사용자 포인트 잔액 불일치
- 중복 차감 또는 중복 적립
- 만료 포인트 누락
- 환불 시 포인트 복구 실패

### 감지 기준

- Point 잔액과 PointHistory 합산 결과 불일치
- 포인트 만료 배치 재실행 시 결과가 달라짐
- OptimisticLockingFailureException 반복 발생
- 동일 주문에 대한 포인트 이력이 중복 생성됨

### 즉시 대응

- affected userId 기준으로 Point와 PointHistory 조회
- 동일 orderId 또는 referenceId 기준 중복 이력 확인
- 포인트 차감/복구 로직 재실행 전 멱등성 키 확인
- 필요 시 수동 보정 이력 생성

### 복구 후 점검

- 사용자별 잔액과 이력 합계 일치
- 만료 대상 포인트 처리 여부
- 재실행 시 중복 만료 이력 미생성
- 환불 시 복구/회수 이력 정상 생성

### 재발 방지

- 포인트 변경 경로에 낙관적 락 적용
- 포인트 이력 referenceId 유니크 제약 검토
- 만료 배치 멱등성 테스트 유지
- 잔액 검증용 관리 API 또는 점검 쿼리 마련

---

## 25. 장애 시나리오 22: 쿠폰 발급 동시성 장애

### 상황

다수 사용자가 동시에 쿠폰 발급을 요청하면서 쿠폰 수량 초과 발급, 중복 발급, Redis와 DB 상태 불일치가 발생하는 경우입니다.

### 영향 범위

- 재고보다 많은 쿠폰 발급
- 동일 사용자 중복 발급
- Redis 수량과 DB 발급 이력 불일치
- 이벤트성 쿠폰 발급 신뢰도 저하

### 감지 기준

- 쿠폰 발급 수량이 총 수량 초과
- 동일 userId + couponId 중복 이력 발생
- Redis remaining count와 DB count 불일치
- 쿠폰 발급 API에서 동시성 예외 증가

### 즉시 대응

- 해당 couponId의 발급 이력 count 확인
- 중복 발급 사용자 확인
- Redis 쿠폰 수량과 DB 발급 수량 비교
- 초과 발급 건은 정책에 따라 회수 또는 보상 처리

### 복구 후 점검

- 쿠폰 총 발급 수량
- 사용자 중복 발급 여부
- Redis key TTL 및 remaining count
- DB unique constraint 적용 여부

### 재발 방지

- Redis Lua Script 기반 원자적 발급 유지
- DB unique constraint로 최종 중복 방어
- 쿠폰 발급 부하 테스트 정기 수행
- Redis 실패 시 쿠폰 발급은 실패 처리

---

## 26. 장애 시나리오 23: 구독 결제 및 상태 변경 장애

### 상황

정기 결제 배치 또는 구독 상태 변경 로직에서 실패가 발생하여 구독 상태가 정상적으로 갱신되지 않는 경우입니다.

### 영향 범위

- 구독 사용자의 혜택 적용 실패
- 결제 실패 사용자가 계속 ACTIVE 상태로 남음
- 정상 결제 사용자가 PAST_DUE 또는 CANCELED 상태로 잘못 변경됨
- 중복 정기 결제 발생 가능성

### 감지 기준

- 구독 결제 배치 실패
- billing history 누락
- 동일 subscriptionId 중복 결제 시도
- 결제 실패 후 상태 변경 누락
- 구독 상태와 결제 이력 불일치

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 200 --no-pager | grep -i "subscription"
sudo journalctl -u one-stop-main -n 200 --no-pager | grep -i "billing"
```

구독 상태 확인:

```sql
SELECT id, user_id, status, next_billing_date, updated_at
FROM subscriptions
ORDER BY updated_at DESC;
```

### 복구 후 점검

- 결제 성공/실패 이력 확인
- 구독 상태 전이 정상 여부
- 다음 결제일 정상 갱신 여부
- 중복 결제 여부
- 혜택 적용 여부

### 재발 방지

- 정기 결제 배치 멱등성 보장
- 동일 billing cycle 중복 결제 방지
- 실패 결제 재시도 정책 명확화
- ShedLock 또는 분산락으로 중복 배치 실행 방지
- 구독 상태 머신 테스트 보강

---

# Part 6. 리뷰/검색/알림/파일 도메인 장애

---

## 27. 장애 시나리오 24: 리뷰 등록 및 조회 장애

### 상황

사용자가 주문 완료 후 리뷰를 등록하거나, 상품 리뷰 목록을 조회하는 과정에서 오류가 발생하는 경우입니다.

### 영향 범위

- 리뷰 등록 불가
- 동일 주문에 대한 중복 리뷰 등록
- 상품 상세 페이지 리뷰 영역 표시 실패
- 판매자 평점 또는 리뷰 통계 불일치

### 감지 기준

- 리뷰 등록 API 4xx/5xx 증가
- 동일 orderItemId에 대한 중복 리뷰 발생
- 리뷰 목록 API 응답 지연 증가
- 리뷰 평점 평균과 실제 리뷰 데이터 불일치

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "review"
```

중복 리뷰 확인:

```sql
SELECT order_item_id, COUNT(*)
FROM reviews
GROUP BY order_item_id
HAVING COUNT(*) > 1;
```

### 복구 후 점검

- 주문 완료 상태에서만 리뷰 등록 가능한지 확인
- 동일 주문 상품에 대한 중복 리뷰 여부
- 리뷰 평점 평균 재계산 필요 여부
- 리뷰 이미지 또는 첨부 데이터 정상 여부

### 재발 방지

- orderItemId 기준 유니크 제약 검토
- 리뷰 등록 권한 검증 강화
- 리뷰 통계 캐시 무효화 정책 추가
- 리뷰 목록 조회 인덱스 점검

---

## 28. 장애 시나리오 25: 검색 및 상품 목록 장애

### 상황

상품 검색, 카테고리 조회, 정렬, 필터링 기능에서 오류 또는 성능 저하가 발생하는 경우입니다.

### 영향 범위

- 사용자가 상품을 찾을 수 없음
- 검색 결과 누락 또는 중복
- 인기 상품/최신 상품 정렬 오류
- DB Full Scan으로 전체 서비스 성능 저하

### 감지 기준

- 검색 API P95/P99 응답 시간 증가
- 특정 키워드 검색 시 500 발생
- 검색 결과 수가 비정상적으로 적거나 많음
- DB CPU 사용량 급증

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "search"
```

최근 검색 조건과 쿼리 파라미터를 확인합니다.

```text
keyword
categoryId
sort
page
size
```

### 복구 후 점검

- 검색 결과 정상 노출 여부
- 카테고리 필터 정상 여부
- 정렬 조건 정상 여부
- 페이지네이션 누락 여부

### 재발 방지

- 검색 조건별 인덱스 점검
- 최소 검색어 길이 제한
- 과도한 page size 제한
- 자주 사용되는 검색 결과 캐싱 검토
- 검색 부하 테스트 수행

---

## 29. 장애 시나리오 26: 알림 발송 장애

### 상황

주문 상태 변경, 결제 승인, 배송 상태 변경, 쿠폰 발급 등의 이벤트가 발생했지만 사용자 알림이 발송되지 않는 경우입니다.

### 영향 범위

- 사용자가 주문/결제/배송 상태 변화를 인지하지 못함
- 판매자 또는 관리자의 후속 대응 지연
- Kafka 이벤트 발행 또는 소비 실패 시 알림 누락
- 중복 소비 시 동일 알림이 여러 번 발송될 수 있음

### 감지 기준

- Kafka consumer 오류
- notification 테이블 저장 실패
- 동일 이벤트에 대한 중복 알림 발생
- 이벤트는 존재하지만 알림 이력이 없음

### 즉시 대응

```bash
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "notification"
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "kafka"
```

알림 이력 확인:

```sql
SELECT id, user_id, type, created_at
FROM notifications
ORDER BY created_at DESC
LIMIT 50;
```

### 복구 후 점검

- 이벤트 발행 여부
- 이벤트 소비 여부
- 알림 저장 여부
- 동일 이벤트 중복 알림 여부
- 사용자 알림 조회 API 정상 여부

### 재발 방지

- notification event idempotency 보장
- 실패 이벤트 재처리 전략 마련
- Kafka 장애 시 DB 기반 outbox 패턴 검토
- 알림 발송 실패 모니터링 추가

---

## 30. 장애 시나리오 27: 파일 및 이미지 업로드 장애

### 상황

상품 이미지, 리뷰 이미지, 사용자 첨부 파일 업로드 과정에서 저장 실패, 경로 오류, 파일 노출 실패가 발생하는 경우입니다.

### 영향 범위

- 상품 등록 실패
- 상품 이미지만 누락
- 리뷰 이미지 표시 실패
- 잘못된 파일 접근 권한으로 보안 위험 발생

### 감지 기준

- 이미지 업로드 API 5xx 증가
- `/images/**` 경로 404 증가
- 서버 디스크 사용량 초과
- 파일 저장 경로 권한 오류

### 즉시 대응

```bash
df -h
ls -al /app/uploads
sudo journalctl -u one-stop-main -n 100 --no-pager | grep -i "upload\|image\|file"
```

Nginx 이미지 라우팅 확인:

```bash
sudo grep -n "images\|uploads" /etc/nginx/conf.d/one-stop.conf
```

### 복구 후 점검

- 파일 저장 경로 존재 여부
- 파일 권한 정상 여부
- 이미지 URL 접근 가능 여부
- DB에 저장된 이미지 경로와 실제 파일 경로 일치 여부

### 재발 방지

- 업로드 경로 환경변수화
- 파일 크기 제한
- 허용 확장자 및 MIME 타입 검증
- 디스크 사용량 모니터링
- 이미지 저장 실패 시 도메인 데이터 롤백 기준 명확화

---

# Part 7. 배치/프론트/배포 장애

---

## 31. 장애 시나리오 28: 배치 작업 장애

### 상황

포인트 만료, 구독 결제, 오래된 주문 정리, 통계 집계 등 배치 작업이 실패하거나 중복 실행되는 경우입니다.

### 영향 범위

- 포인트 만료 누락
- 구독 결제 누락 또는 중복 결제
- 오래된 PENDING_PAYMENT 주문 누적
- 통계 데이터 불일치
- 대량 데이터 처리 중 DB 부하 증가

### 감지 기준

- Spring Batch job status FAILED
- 동일 JobParameter로 중복 실행
- 배치 실행 시간이 평소보다 크게 증가
- 배치 이후 대상 데이터가 남아 있음

### 즉시 대응

Spring Batch 메타 테이블을 확인합니다.

```sql
SELECT JOB_INSTANCE_ID, JOB_NAME
FROM BATCH_JOB_INSTANCE
ORDER BY JOB_INSTANCE_ID DESC;
```

```sql
SELECT JOB_EXECUTION_ID, STATUS, START_TIME, END_TIME, EXIT_CODE
FROM BATCH_JOB_EXECUTION
ORDER BY JOB_EXECUTION_ID DESC;
```

애플리케이션 로그 확인:

```bash
sudo journalctl -u one-stop-main -n 200 --no-pager | grep -i "batch\|job\|step"
```

### 복구 후 점검

- 실패 Job 재실행 가능 여부
- 재실행 시 중복 처리 발생 여부
- 대상 데이터 처리 완료 여부
- 배치 실행 중 생성된 이력 데이터 정합성 여부

### 재발 방지

- JobParameter 설계 명확화
- 배치 멱등성 보장
- Cursor/Paging 처리 방식 검증
- 대량 처리 시 chunk size 조정
- 실패 Step 재시작 전략 문서화

---

## 32. 장애 시나리오 29: 프론트엔드 정적 파일 배포 장애

### 상황

React 빌드 산출물이 Nginx 정적 파일 경로에 정상 배포되지 않아 화면이 깨지거나, 새로고침 시 404가 발생하는 경우입니다.

### 영향 범위

- 메인 페이지 접근 실패
- React Router 경로 새로고침 시 404
- 최신 JS/CSS 파일 로드 실패
- OAuth2 callback 페이지 접근 실패

### 감지 기준

```bash
curl -Ik https://onestop1.duckdns.org
curl -Ik "https://onestop1.duckdns.org/oauth2/callback?code=test"
```

정상:

```text
content-type: text/html
```

비정상:

```text
HTTP/2 404
HTTP/2 500
content-type: application/json
```

### 즉시 대응

정적 파일 경로를 확인합니다.

```bash
ls -al /var/www/frontend
sudo grep -n "root\|try_files" /etc/nginx/conf.d/one-stop.conf
```

정상 Nginx 설정:

```nginx
location / {
    root /var/www/frontend;
    try_files $uri $uri/ /index.html;
}
```

### 복구 후 점검

- 메인 페이지 접근 가능 여부
- 새로고침 시 React Router 경로 유지 여부
- OAuth2 callback 페이지 접근 가능 여부
- 최신 JS/CSS 파일 로드 여부

### 재발 방지

- 프론트 빌드 후 smoke test 추가
- 배포 전 `/oauth2/callback?code=test` 확인
- 정적 파일 경로와 Nginx root 경로 문서화
- 이전 빌드 백업 유지

---

## 33. 장애 시나리오 30: CI/CD 배포 장애

### 상황

GitHub Actions, Docker build, 서버 배포, systemd 재기동 과정에서 실패가 발생하는 경우입니다.

### 영향 범위

- 최신 코드 배포 실패
- 일부 서버만 최신 버전으로 동작
- main/dummy 버전 불일치
- Nginx 라운드로빈 환경에서 요청마다 응답이 달라짐

### 감지 기준

- GitHub Actions 실패
- 서버의 app.jar 수정 시간이 다름
- main/dummy 응답 결과가 다름
- 한쪽 서버만 5xx 발생

### 즉시 대응

각 서버에서 배포 파일과 실행 상태를 확인합니다.

```bash
ls -lh /app/app.jar
sudo systemctl status one-stop-main --no-pager
sudo systemctl status one-stop-dummy --no-pager
```

직접 서버별 API를 확인합니다.

```bash
curl -Ik http://127.0.0.1:8080/oauth2/authorization/kakao
```

외부 라운드로빈 확인:

```bash
for i in {1..10}; do
  curl -skI https://onestop1.duckdns.org/oauth2/authorization/kakao | grep -i "redirect_uri"
done
```

### 복구 후 점검

- main/dummy 동일 버전 여부
- application-prod.yml 동일 여부
- systemd 서비스 정상 실행 여부
- Nginx upstream 정상 여부
- smoke test 통과 여부

### 재발 방지

- 배포 후 main/dummy 설정 diff 확인
- 배포 자동화 스크립트에 smoke test 포함
- 실패 시 자동 롤백 검토
- 운영 설정 파일 변경 이력 관리

---

## 34. 장애 대응 후 회고 템플릿

장애 복구 후 다음 양식으로 회고를 작성합니다.

```text
## 장애 개요
- 발생 일시:
- 종료 일시:
- 영향 범위:
- 사용자 영향:
- 담당자:

## 원인
- 직접 원인:
- 근본 원인:
- 관련 설정/코드:

## 대응 과정
1.
2.
3.

## 복구 확인
- 확인 API:
- 확인 로그:
- 데이터 정합성 점검 결과:

## 재발 방지
- 단기 조치:
- 장기 조치:
- 테스트 보강:
- 모니터링 보강:
```

---

## 35. README 링크 추가 예시

메인 README에는 다음 링크를 추가합니다.

```markdown
- [장애 시나리오 및 대응 Runbook](documents/operations/incident-response-runbook.md)
```

---

## 36. 최종 정리

One-Stop 프로젝트의 장애 대응 전략은 단순히 서버를 재시작하는 수준이 아니라, 장애 유형별로 사용자 영향과 데이터 정합성을 구분하여 대응하는 것을 목표로 합니다.

특히 인증, 결제, 포인트, 쿠폰, 주문, 재고 도메인은 장애 상황에서도 중복 처리와 상태 불일치를 방지해야 하므로, Redis Lua Script, Token Versioning, Refresh Token Rotation, Device ID 기반 세션 관리, 낙관적 락, 멱등성 키, 보상 트랜잭션, 상태 머신과 같은 방어 전략을 함께 사용합니다.

이를 통해 프로젝트는 기능 구현뿐 아니라 운영 환경에서 발생 가능한 장애를 고려한 백엔드 시스템으로 발전할 수 있습니다.
