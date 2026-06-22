# 성능 및 안정성 검증 결과

> 본 문서는 One-Stop 프로젝트에서 수행한 k6 기반 부하 테스트, 동시성 테스트, 보안 정책 검증, 이벤트 흐름 검증 결과를 정리한다.  
> 목적은 과장된 성능 수치를 나열하는 것이 아니라, 실제 테스트 결과를 기반으로 각 도메인의 안정 구간, 한계 구간, 정합성 보장 여부, 기술 선택 근거를 기록하는 것이다.

---

# 1. 문서 목적

One-Stop 프로젝트는 커머스 플랫폼의 핵심 요구사항인 인증 보안, 주문/결제 정합성, 쿠폰 선착순 발급, 포인트 차감, 상품 조회, AI 보조 기능, 이벤트 기반 알림을 검증하기 위해 여러 부하 테스트와 동시성 테스트를 수행했다.

본 문서는 다음 질문에 답하기 위해 작성되었다.

- 인증 API는 어느 구간까지 안정적으로 처리되는가?
- Refresh Token Rotation은 동시 요청 상황에서도 안전하게 동작하는가?
- Rate Limit 정책은 실제로 요청을 차단하는가?
- 쿠폰 발급에서 Redis Lua 전략은 정합성과 성능 측면에서 유효한가?
- 주문 재고 차감에서 DB 비관적 락과 Redis 분산 락 중 어떤 전략이 더 적합한가?
- 포인트 차감에서 낙관적 락과 재시도 정책은 잔액 정합성을 보장하는가?
- Redis 캐시가 적용된 조회 API는 부하 증가 상황에서 안정적인가?
- AI, Kafka, Nginx 등 부가 기능은 배포 환경에서 정상 동작하는가?

---

# 2. 테스트 환경

테스트는 로컬 환경과 배포 서버 환경을 나누어 수행했다.

| 구분 | 내용 |
|---|---|
| Load Test Tool | k6 |
| Application | Spring Boot |
| Reverse Proxy | Nginx |
| Database | Docker MySQL / 로컬 MySQL |
| Cache & Token Store | Docker Redis / 로컬 Redis |
| Messaging | Docker Kafka / Zookeeper |
| 주요 테스트 대상 | Auth, Refresh, Coupon, Order, Point, Product, AI, Kafka, Nginx |
| 테스트 방식 | constant-arrival-rate, shared-iterations, VU 기반 동시성 테스트 |

주의할 점은, 일부 테스트는 운영급 인프라의 최대 TPS를 측정하기 위한 것이 아니라 **동일 환경 내에서의 안정성, 정합성, 상대 성능 비교**를 목적으로 수행했다는 점이다.

---

# 3. 종합 검증 결과 요약

| 영역 | 검증 대상 | 주요 결과 | 판정 |
|---|---|---|---|
| Auth/Login | 로그인 API 처리 한계 | 20 RPS까지 성공률 100%, p95 185.62ms. 30 RPS부터 p95 2.14s로 경계 구간 진입 | 통과 |
| Auth/RateLimit | 동일 계정 반복 로그인 차단 | 8회 중 4회 허용, 4회 429 차단, 예상 밖 응답 0건 | 통과 |
| Refresh Token | 동일 RT 동시 재발급 | 10건 중 1건 성공, 9건 거부. 20/100건 동시 요청에서도 1건만 성공 | 통과 |
| Device Limit | 기기 수 제한 및 LRU Eviction | 신규 기기 로그인 10건 성공 후 Redis ZSET 최종 5개 유지 | 통과 |
| Coupon | 1000 VU 선착순 발급 | DECR/LUA/LOCK 모두 500장 정확 발급, 초과 발급 0건 | 통과 |
| Coupon 전략 비교 | Lua vs Redisson Lock | Lua avg 6.22s, Lock avg 9.8s. Lua가 Lock 대비 평균 응답시간 약 36.5% 낮음 | Lua 우위 |
| Order | DB락 vs Redis락 | 두 방식 모두 재고 정합성 100%. DB락은 타임아웃 0건, Redis락은 941건 waitTime 초과 | DB락 채택 근거 확보 |
| Point | 1000 VU 포인트 차감 | 910건 성공, 최종 잔액 기대값과 실제값 일치, 음수 잔액 0건 | 통과 |
| Product Cache | 상품 목록 조회 | 100 req/s에서 5,995건 완료, 성공률 100%, p95 6ms | 통과 |
| AI | AI 엔드포인트 부하 | 8 VU 5분, AI summary p95 25ms, related p95 34ms, assistant p95 893ms | 통과 |
| AI Review | 리뷰 동시 등록 | 10명 동시 리뷰 등록 10/10 성공, error_rate 0%, 요약 상태 READY 전환 | 통과 |
| Kafka | payment.approved 이벤트 | 메시지 발행 → Consumer 소비 → notification 저장 확인 | 통과 |
| Nginx | 라운드로빈 로드밸런싱 | upstream 로그에서 양쪽 서버로 요청 분산 확인 | 통과 |

---

# 4. 도메인별 성능 및 안정성 검증

---

## 4.1 Auth/Login 부하 테스트

## 테스트 목적

로그인 API가 실제 인증 처리 경로에서 어느 수준까지 안정적으로 처리 가능한지 확인했다.

로그인 API는 단순 조회 API와 달리 다음 비용을 포함한다.

- BCrypt 비밀번호 검증
- JWT Access Token 발급
- Refresh Token 발급
- Redis Refresh Token 저장
- device_id 쿠키 발급
- 사용자 상태 검증
- Redis 기반 인증 부가 처리

## 테스트 조건

| 항목 | 값 |
|---|---|
| 대상 API | `POST /api/auth/login` |
| 테스트 도구 | k6 |
| 테스트 방식 | constant-arrival-rate |
| 측정 구간 | 5, 10, 20, 30, 40, 50 RPS |
| 테스트 시간 | 각 구간 30초 |
| 서버 구성 | EC2 + Nginx + Spring Boot |
| DB/Cache | Docker MySQL + Docker Redis |

로그인 처리량 측정을 위해 일부 Rate Limit 정책은 load-test 환경에서 일시적으로 완화했다.  
단, BCrypt 검증, JWT 발급, Refresh Token 저장, device_id 발급 등 실제 인증 처리 경로는 유지했다.

## 결과

| 구간 | 실제 처리량 | 성공률 | p95 Latency | 판정 |
|---|---:|---:|---:|---|
| 20 RPS | 19.96 RPS | 100% | 185.62ms | 안정 구간 |
| 30 RPS | 28.33 RPS | 100% | 2.14s | 경계 구간 |
| 40 RPS | 28.69 RPS | 100% | 9.52s | 포화 구간 |
| 50 RPS | 27.30 RPS | 100% | 13.36s | 포화 구간 |

## 분석

20 RPS까지는 성공률 100%, 서버 오류율 0%, p95 185.62ms로 안정적으로 처리되었다.

30 RPS부터는 요청 성공률은 유지되었지만 p95가 2초 이상으로 증가했다.  
40 RPS 이상에서는 실제 처리량이 27~29 RPS 수준에서 정체되고 dropped iterations가 발생했다.

이는 서버가 오류를 반환한 것이 아니라, 로그인 처리 비용이 누적되어 목표 도착률을 유지하지 못한 상황으로 해석된다.

## 결론

현재 서버 구성에서 로그인 API의 안정 처리 구간은 **20 RPS**, 실질 처리 한계는 **약 28~30 RPS**로 판단했다.

---

## 4.2 Auth Rate Limit 정책 검증

## 테스트 목적

동일 계정의 반복 로그인 요청이 Rate Limit 정책에 의해 정상적으로 차단되는지 검증했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 대상 API | `POST /api/auth/login` |
| 반복 요청 | 8회 |
| 조건 | 동일 이메일, 비밀번호, IP, device_id 사용 |
| 저장소 | Redis Rate Limit Counter |

## 결과

| 항목 | 결과 |
|---|---:|
| 총 요청 | 8건 |
| 정상 로그인 허용 | 4건 |
| Rate Limit 차단 | 4건 |
| 예상 밖 응답 | 0건 |
| 체크 성공률 | 100% |
| avg 응답시간 | 65ms |
| p95 응답시간 | 132.68ms |

## 분석

8건 중 4건은 정상 처리되고, 이후 4건은 429로 차단되었다.

k6에서는 429 응답을 `http_req_failed`로 집계하지만, 본 테스트에서 429는 의도된 보안 정책 응답이므로 기능 실패가 아니다.

## 결론

동일 조건의 반복 로그인 요청에 대해 Redis 기반 Rate Limit 정책이 정상적으로 동작함을 확인했다.

---

## 4.3 Refresh Token Rotation 동시성 검증

## 테스트 목적

동일 Refresh Token이 동시에 여러 재발급 요청에 사용될 때, Redis Lua CAS 기반 Refresh Token Rotation이 원자적으로 동작하는지 검증했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 대상 API | `POST /api/auth/refresh` |
| 인증 방식 | `refresh_token` + `device_id` Cookie |
| 검증 대상 | Redis Lua CAS 기반 RTR |
| 성공 조건 | 동일 RT 동시 요청 중 1개만 성공, 나머지 거부 |

## 결과 1: 10건 동시 요청

| 항목 | 결과 |
|---|---:|
| Setup 로그인 요청 | 1건 |
| 동시 Refresh 요청 | 10건 |
| Refresh 성공 | 1건 |
| Refresh 거부 | 9건 |
| 예상 밖 응답 | 0건 |
| 500 오류 | 0건 |
| 체크 성공률 | 100% |

## 결과 2: 20건 / 100건 동시 요청

| 동시 요청 수 | 성공 | 거부 | 서버 오류율 |
|---:|---:|---:|---:|
| 20건 | 1건 | 19건 | 0% |
| 100건 | 1건 | 99건 | 0% |

## 분석

Refresh Token Rotation 동시성 테스트에서는 모든 요청이 성공하면 오히려 보안 결함이다.  
동일 RT로 여러 요청이 동시에 성공한다면 하나의 Refresh Token이 중복 사용될 수 있기 때문이다.

이번 테스트에서는 10건, 20건, 100건 동시 요청 모두에서 정확히 1건만 성공했고, 나머지는 이미 사용된 Refresh Token 재사용으로 거부되었다.

## 결론

Redis Lua CAS 기반 Refresh Token Rotation이 동시 재발급 상황에서도 원자적으로 동작하며, RT 재사용 공격 및 race condition을 정상적으로 차단함을 확인했다.

---

## 4.4 Device Limit 및 LRU Eviction 검증

## 테스트 목적

동일 사용자가 여러 기기에서 로그인할 때, 사용자별 최대 기기 수 제한과 LRU Eviction이 정상적으로 동작하는지 검증했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 대상 API | `POST /api/auth/login` |
| 테스트 방식 | 동일 계정 신규 기기 로그인 10회 |
| 검증 대상 | Redis ZSET `devices:{userId}` |
| 기대 결과 | 최종 활성 기기 수 5개 |

## 결과

| 항목 | 결과 |
|---|---:|
| 전체 로그인 요청 | 10건 |
| 로그인 성공 | 10건 |
| HTTP 실패율 | 0% |
| device_id 쿠키 발급 | 100% |
| refresh_token 쿠키 발급 | 100% |
| p95 latency | 268.92ms |
| 최종 Redis ZSET 기기 수 | 5개 |

## 분석

동일 계정으로 10개의 신규 기기 로그인을 수행했음에도 Redis의 `devices:{userId}` ZSET에는 최종적으로 5개의 device_id만 유지되었다.

이를 통해 신규 기기 등록, Refresh Token의 device 단위 발급, 최대 기기 수 초과 시 오래된 기기 제거 정책이 정상 동작함을 확인했다.

## 결론

device_id 기반 다중 기기 세션 관리와 LRU Eviction 정책이 정상적으로 동작했다.

---

## 4.5 쿠폰 선착순 발급 동시성 검증

## 테스트 목적

1000 VU 동시 요청 상황에서 쿠폰 선착순 발급 전략별 성능과 재고 정합성을 비교했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 테스트 환경 | AWS EC2 배포 서버 |
| VU | 1000명 동시 |
| 쿠폰 수량 | 각 전략 500개 |
| 비교 전략 | DECR, LUA, LOCK |

## 결과

| 전략 | 발급 성공 | avg 응답시간 | p95 응답시간 | max 응답시간 |
|---|---:|---:|---:|---:|
| DECR | 500 / 500 | 6.43s | 10.03s | 19.2s |
| LUA | 500 / 500 | 6.22s | 10.03s | 13.0s |
| LOCK | 500 / 500 | 9.8s | 15.86s | 19.25s |

## 재고 정합성

| 전략 | total_quantity | issued_quantity | user_coupon 수 | 정합성 |
|---|---:|---:|---:|---|
| DECR | 500 | 500 | 500 | 일치 |
| LUA | 500 | 500 | 500 | 일치 |
| LOCK | 500 | 500 | 500 | 일치 |

## 분석

세 전략 모두 초과 발급 0건으로 정합성은 동일하게 유지되었다.  
성능 측면에서는 Lua 전략이 평균 6.22s로 가장 낮았고, Redisson Lock은 평균 9.8s, p95 15.86s로 가장 높았다.

Lua 전략은 Redisson Lock 대비 평균 응답시간 기준 약 36.5% 낮았다.

```text
(9.8s - 6.22s) / 9.8s = 약 36.5%
```

## 결론

쿠폰 발급은 정합성 측면에서는 세 전략 모두 통과했지만, 성능과 단순성을 고려할 때 Redis Lua 원자 스크립트 전략이 가장 적합하다고 판단했다.

---

## 4.6 주문 동시성 검증: DB 비관적 락 vs Redis 분산 락

## 테스트 목적

동일 상품에 1000 VU가 동시에 주문을 생성하는 상황에서 DB 비관적 락과 Redis 분산 락의 성능 및 재고 정합성을 비교했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 테스트 환경 | AWS EC2 배포 서버 |
| VU | 1000명 동시 |
| 테스트 상품 | item_id = 1 |
| 초기 재고 | 50,000 |
| 비교 방식 | DB 비관적 락 vs Redis 분산 락 |

## 결과

| 구분 | DB 비관적 락 | Redis 분산 락 |
|---|---:|---:|
| 주문 성공 | 9,455건 | 8,001건 |
| 락 타임아웃 | 0건 | 941건 |
| avg 응답시간 | 12.57s | 13.61s |
| p95 응답시간 | 15.15s | 16.65s |
| max 응답시간 | 17.46s | 19.32s |

## 재고 정합성 검증

| 항목 | 값 |
|---|---:|
| 초기 재고 | 50,000 |
| DB락 성공 + Redis락 성공 | 17,456건 |
| 기대 잔여 재고 | 32,544 |
| 실제 DB 재고 | 32,544 |

## 분석

두 방식 모두 재고 정합성은 100% 유지했다.  
그러나 Redis 분산 락은 waitTime 초과로 941건이 거부되었고, DB 비관적 락은 타임아웃 없이 더 많은 주문을 처리했다.

극단적으로 동일 item_id에 요청이 집중되는 구조에서는 Redis 분산 락보다 DB 비관적 락의 대기 큐 방식이 더 안정적으로 동작했다.

## 결론

현재 구조에서는 Redis 분산 락을 무조건 선택하기보다, 단일 상품 재고 차감 시나리오에서는 DB 비관적 락이 더 적합하다고 판단했다.

이 테스트는 “유행하는 기술”보다 실제 부하 결과와 도메인 특성에 맞춰 기술을 선택했다는 근거가 된다.

---

## 4.7 포인트 차감 동시성 검증

## 테스트 목적

동일 계정의 포인트를 1000 VU가 동시에 차감할 때, 낙관적 락과 재시도 정책이 잔액 정합성을 보장하는지 검증했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 테스트 환경 | AWS EC2 배포 서버 |
| VU | 1000 |
| 대상 계정 | testbuyer1 |
| 차감 금액 | 요청당 100pt |
| 초기 잔액 | 399,900pt |
| 적용 기술 | `@Version` + `@Retryable(maxAttempts=5)` |

## 결과

| 항목 | 결과 |
|---|---:|
| 총 시도 | 1000회 |
| 차감 성공 | 910건 |
| 성공 차감 총액 | 91,000pt |
| 잔액 부족 실패 | 0건 |
| 낙관적 락 재시도 5회 소진 | 0건 |
| 기타 실패 | 90건 |
| 테스트 소요 시간 | 20.0s |

## 잔액 정합성 검증

| 항목 | 값 |
|---|---:|
| 초기 잔액 | 399,900pt |
| 성공 차감 총액 | 910 × 100 = 91,000pt |
| 기대 최종 잔액 | 308,900pt |
| 실제 DB 잔액 | 308,900pt |

## 분석

1000 VU 중 910건이 성공했고, 성공한 차감 건수 기준으로 기대 잔액과 실제 잔액이 정확히 일치했다.  
음수 잔액과 중복 차감은 발생하지 않았다.

90건의 기타 실패는 동일 IP/동일 계정 집중 요청에 따른 Rate Limit 등 부가 제약의 영향으로 추정했다.

## 결론

포인트 차감은 낙관적 락과 재시도 정책을 통해 동시 요청 상황에서도 잔액 정합성을 유지했다.

---

## 4.8 Redis 캐시 적용 상품 목록 조회 검증

## 테스트 목적

Redis 캐시가 적용된 상품 목록 조회 API가 로컬 환경에서 부하 증가에도 안정적으로 처리되는지 검증했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 대상 API | `GET /api/products?page=0&size=20&sort=createdAt,desc` |
| 테스트 환경 | Windows 로컬 PC |
| 테스트 도구 | k6 |
| 부하 단계 | 2 req/s, 30 req/s, 100 req/s |

본 테스트는 운영 서버 최대 처리량 산정이 아니라, 동일 로컬 환경에서 캐시 적용 조회 경로의 안정성을 확인하기 위해 수행했다.

## 결과

| 구분 | 최초 Smoke | Warm Smoke | 30 req/s | 100 req/s |
|---|---:|---:|---:|---:|
| 완료 요청 | 21건 | 21건 | 1,801건 | 5,995건 |
| 실제 처리량 | 2.10 req/s | 2.10 req/s | 30.01 req/s | 99.91 req/s |
| 요청 성공률 | 100% | 100% | 100% | 100% |
| HTTP 실패 | 0건 | 0건 | 0건 | 0건 |
| avg 응답시간 | 60ms | 10.11ms | 5.8ms | 5.92ms |
| p95 응답시간 | 305.65ms | 16.79ms | 8.43ms | 6ms |

## 분석

최초 Smoke 이후 Warm Smoke에서 p95가 305.65ms에서 16.79ms로 감소했다.  
100 req/s 부하에서도 5,995건의 요청이 모두 성공했고, p95는 6ms로 안정적이었다.

30 req/s에서 100 req/s로 요청량을 약 3.3배 높였지만 응답시간은 증가하지 않았다.

## 결론

Redis 캐시가 적용된 상품 목록 조회 경로는 로컬 100 req/s 부하에서 성공률 100%, p95 6ms를 기록했다.  
다만 운영 환경의 최대 처리량이나 RDS IOPS 절감률을 직접 측정한 것은 아니므로, 비용 절감 효과는 후속 운영 지표로 검증해야 한다.

---

## 4.9 AI 더미데이터 부하 테스트

## 테스트 목적

배포 서버에서 AI 관련 엔드포인트가 복합 시나리오 부하에서도 안정적으로 응답하는지 검증했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 대상 서버 | 배포 서버 |
| VU | 8 |
| 테스트 시간 | 5분 |
| 주요 시나리오 | 상품 목록, 주문 생성, AI 리뷰 요약, AI Assistant, AI 연관 상품 |

## 결과

| 지표 | 결과 | 임계값 | 판정 |
|---|---:|---:|---|
| ai_summary_duration p95 | 25ms | 3000ms | 통과 |
| ai_related_duration p95 | 34ms | 3000ms | 통과 |
| ai_assist_duration p95 | 893ms | 5000ms | 통과 |
| order_duration p95 | 60ms | 500ms | 통과 |
| products_duration p95 | 107ms | 500ms | 통과 |
| login_duration p95 | 221ms | 500ms | 통과 |
| 전체 error_rate | 4.22% | <1% | 초과 |

## 실패 원인 분석

전체 실패 91건은 AI 엔드포인트가 아니라 주문 생성 실패였다.

주문 실패 원인은 `ORDER_CREATE_PER_USER` Rate Limit 정책에 의한 의도된 429 응답이었다.  
따라서 AI 엔드포인트 자체의 장애로 해석하지 않았다.

## AI 엔드포인트 성능 요약

| 엔드포인트 | 결과 |
|---|---|
| AI 리뷰 요약 | avg 16ms, p95 25ms |
| AI 연관 상품 | avg 22ms, p95 34ms |
| AI Assistant | avg 113ms, p95 893ms, max 1.17s |

## 결론

AI 관련 엔드포인트는 설정한 임계값 내에서 안정적으로 동작했다.  
전체 error_rate 초과는 주문 생성 Rate Limit으로 인한 의도된 실패였으며, AI 기능 자체의 장애는 확인되지 않았다.

---

## 4.10 AI 리뷰 요약 동시성 테스트

## 테스트 목적

리뷰 요약이 없는 상태에서 10명이 동시에 리뷰를 작성할 때, 최초 요약 생성 충돌이나 낙관적 락 충돌이 서버 오류로 전파되지 않는지 검증했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 대상 서버 | 배포 서버 |
| 대상 상품 | product_id = 90001 |
| VU / iterations | 10 VU × 10 iterations |
| 초기 요약 상태 | INSUFFICIENT, reviewCount = 0 |

## 결과

| 항목 | 값 |
|---|---:|
| 리뷰 등록 성공 | 10 / 10 |
| error_rate | 0% |
| 요약 상태 before | INSUFFICIENT, reviewCount = 0 |
| 요약 상태 after | READY, reviewCount = 9 |

## 분석

동시 최초 요약 생성 상황에서 `DataIntegrityViolationException`은 핸들러가 정상적으로 처리했고, 에러가 사용자 요청으로 전파되지 않았다.

## 결론

AI 리뷰 요약 생성 흐름은 동시 리뷰 등록 상황에서도 서버 오류 없이 처리되었고, 요약 상태가 READY로 전환됨을 확인했다.

---

## 4.11 Kafka 이벤트 흐름 검증

## 테스트 목적

Kafka가 배포 서버에서 정상 실행 중이며, `payment.approved` 이벤트를 Consumer가 소비해 알림으로 저장하는지 엔드투엔드로 검증했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| Kafka | confluentinc/cp-kafka:7.5.0 |
| Zookeeper | confluentinc/cp-zookeeper:7.5.0 |
| Topic | `payment.approved` |
| Consumer | `PaymentApprovedConsumer` |
| 결과 저장 | `notification` 테이블 |

## 검증 흐름

```text
payment.approved 토픽에 테스트 메시지 발행
↓
PaymentApprovedConsumer 소비 확인
↓
notification 테이블 저장 확인
```

## 결론

Kafka 기반 결제 승인 이벤트 흐름이 실제 배포 환경에서 정상 동작함을 확인했다.  
본 테스트는 처리량 성능 검증이 아니라 이벤트 플로우의 기능 검증이다.

---

## 4.12 Nginx 로드밸런싱 검증

## 테스트 목적

Nginx upstream 설정이 두 개의 Spring Boot 서버로 요청을 분산하는지 확인했다.

## 테스트 조건

| 항목 | 값 |
|---|---|
| 서버 1 | main EC2 |
| 서버 2 | dummy EC2 |
| 방식 | Nginx upstream round-robin |
| 확인 방법 | access log의 `$upstream_addr` 확인 |

## 결과

Nginx access log에서 다음과 같이 서로 다른 upstream으로 요청이 전달되는 것을 확인했다.

```text
upstream=10.0.22.194:8080
upstream=127.0.0.1:8080
```

## 추가 확인 사항

테스트 중 다음 이슈도 함께 확인되었다.

- `/images/default-product.png` 500 오류 반복
- `ProductSummaryResponse` 캐시 역직렬화 실패
- `@NoArgsConstructor` 누락 가능성

## 결론

Nginx 라운드로빈 로드밸런싱은 정상 동작했다.  
다만 일부 정적 리소스 및 캐시 역직렬화 오류는 별도 수정 대상으로 분리했다.

---

# 5. 테스트 결과가 기술 선택에 반영된 지점

## 5.1 쿠폰: Redis Lua Script 선택 근거 확보

쿠폰 발급 테스트에서 DECR, Lua, Redisson Lock 모두 정합성은 유지했다.  
하지만 Lua는 Lock 대비 평균 응답시간이 낮고 max 응답시간도 안정적이었다.

따라서 선착순 쿠폰 발급에는 Redis Lua Script 기반 원자 처리가 적합하다고 판단했다.

---

## 5.2 주문: DB 비관적 락 선택 근거 확보

주문 재고 차감에서는 Redis 분산 락보다 DB 비관적 락이 더 안정적으로 동작했다.

두 방식 모두 정합성은 유지했지만, Redis 분산 락은 waitTime 초과로 941건이 거부되었다.  
반면 DB 비관적 락은 타임아웃 없이 더 많은 주문을 처리했다.

따라서 현재 구조에서는 DB 비관적 락을 선택하는 것이 더 타당하다고 판단했다.

---

## 5.3 포인트: 낙관적 락과 재시도 정책의 정합성 검증

포인트 차감은 충돌이 발생할 수 있지만, 성공한 요청 기준으로 잔액 정합성이 정확히 유지되었다.

이를 통해 포인트처럼 충돌 가능성은 있지만 전체 요청을 직렬화할 필요가 없는 영역에서는 `@Version` 기반 낙관적 락과 재시도 정책이 적합하다고 판단했다.

---

## 5.4 Auth: tokenVersion, RTR, device_id 정책 검증

Refresh Token Rotation 테스트와 Device Limit 테스트를 통해 다음 정책의 유효성을 확인했다.

- 동일 RT 재사용 시 1건만 성공
- 나머지 요청 차단
- 서버 오류율 0%
- 신규 기기 10건 로그인 후 최종 활성 기기 5개 유지

이를 통해 Redis Lua CAS, device_id 기반 기기 관리, LRU Eviction 정책이 실제 동시성 상황에서도 정상 동작함을 확인했다.

---

## 5.5 AI: 핵심 거래 흐름과 분리된 보조 기능 검증

AI 엔드포인트는 복합 부하 상황에서도 임계값 내 응답시간을 유지했다.  
또한 주문 Rate Limit으로 인한 실패가 AI 자체 장애로 오인되지 않도록 시나리오별 지표를 분리했다.

이를 통해 AI는 핵심 거래 흐름과 분리된 보조 기능으로 운영하는 방향이 적절하다고 판단했다.

---

# 6. 한계 및 후속 개선

## 6.1 운영 환경 최대 처리량 검증 필요

일부 테스트는 로컬 또는 단일 배포 서버 기준으로 수행되었다.  
따라서 운영 환경 전체의 최대 TPS나 RDS/Redis 비용 절감률을 직접 증명한 것은 아니다.

후속 개선:

- 운영 환경 기준 장시간 부하 테스트
- Prometheus/Grafana 기반 CPU, Memory, DB Connection, Redis latency 수집
- RDS IOPS, DB CPU, Network 지표 기반 FinOps 검증

---

## 6.2 Rate Limit과 성능 측정 분리 필요

Auth, Order, AI 일부 테스트에서 Rate Limit이 의도된 실패로 집계되었다.

후속 개선:

- 성능 측정용 profile과 운영 보안 profile 분리
- 429 응답을 기능 실패와 정책 차단으로 분리 집계
- 도메인별 Rate Limit 정책값 문서화

---

## 6.3 장시간 Soak Test 필요

대부분의 테스트는 짧은 시간 동안 동시성 또는 부하 구간을 확인했다.

후속 개선:

- 30분~2시간 이상 장시간 테스트
- 메모리 누수 확인
- Redis key 누적 확인
- DB Connection pool 안정성 확인
- Kafka Consumer lag 확인

---

## 6.4 관측성 기반 성과 검증 필요

현재 문서는 k6 결과와 DB/Redis 상태 확인 중심이다.  
CPU, GC, HikariCP, Redis latency, Kafka lag 등 시스템 지표와 결합하면 더 설득력 있는 성과 문서가 된다.

후속 개선:

- Spring Boot Actuator metric 수집
- Prometheus/Grafana 대시보드 구성
- HikariCP active/idle connection 추적
- Redis command latency 추적
- Kafka consumer lag 모니터링

---

# 7. 기존 성과 문서 대비 정리한 내용

기존 성과 문서에서 근거가 부족한 과장 수치는 제거하고, 실제 테스트 결과로 확인된 수치만 유지했다.

## 제거 또는 후속 검증으로 이동한 항목

- TPS 3,200+
- 쿠폰 16배 향상
- P95 0.08s
- RDS Write IOPS 95.4% 절감
- DB CPU 82% → 14%
- 상품 상세 조회 API 가용성 400% 증가
- DB 커넥션 점유 시간 98% 단축
- 결제 응답 1.2s → 0.05s
- Kafka 리밸런싱 92% 단축
- Minor GC 80% 감소
- Max GC Pause 50ms
- JMH/JMeter/ElastiCache/Prometheus 기반 측정 주장

## 유지한 항목

- k6 기반 부하 테스트
- Redis Lua 기반 쿠폰 정합성 검증
- DB락 vs Redis락 비교 결과
- Refresh Token Rotation 원자성 검증
- Rate Limit 차단 검증
- Device Limit LRU 검증
- 포인트 잔액 정합성 검증
- Redis 캐시 조회 성능 검증
- AI 엔드포인트 부하 검증
- Kafka 이벤트 흐름 검증
- Nginx 로드밸런싱 확인

---

# 8. 포트폴리오 요약

One-Stop 프로젝트는 단순히 기능 구현에 그치지 않고, 주요 병목과 정합성 위험이 있는 구간을 k6 기반 테스트로 검증했다.

검증 결과, 쿠폰 발급에서는 Redis Lua Script가 Redisson Lock 대비 더 낮은 응답시간으로 동일한 정합성을 보장했으며, 주문 재고 차감에서는 Redis 분산 락보다 DB 비관적 락이 더 안정적으로 동작함을 확인했다.

Auth/Security 영역에서는 Refresh Token Rotation의 Redis Lua CAS가 동일 RT 동시 재사용 상황에서 1건만 성공시키고 나머지를 차단함을 검증했고, device_id 기반 기기 제한 정책도 Redis ZSET 기준 최종 5개 기기 유지로 확인했다.

포인트 차감은 낙관적 락과 재시도 정책으로 성공 차감 건수 기준 잔액 정합성을 보장했으며, 상품 목록 조회는 Redis 캐시 적용 상태에서 로컬 100 req/s 부하를 성공률 100%, p95 6ms로 처리했다.

이러한 결과를 바탕으로 One-Stop은 “무조건 고급 기술을 도입하는 것”이 아니라, 실제 테스트 결과를 기준으로 도메인별로 적합한 동시성 제어와 안정성 전략을 선택했다.
