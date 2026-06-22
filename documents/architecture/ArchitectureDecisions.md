# Architecture & Technical Decisions

> 기술은 목적이 아니라 비즈니스 문제를 해결하기 위한 수단입니다.
> One-Stop 플랫폼은 성능, 데이터 정합성, 보안성, 운영 단순성, 비용 효율성을 기준으로 기술을 선택했습니다.
> 최신 기술 도입 자체보다 문제 해결 적합성과 운영 가능한 복잡도를 우선했습니다.

---

# 1. 문서 목적

이 문서는 One-Stop 프로젝트의 아키텍처 진화 과정과 주요 기술 의사결정 기록을 정리한다.

프로젝트를 진행하면서 단순 기능 구현을 넘어 다음과 같은 문제를 해결해야 했다.

* 대량 요청 상황에서의 동시성 제어
* 결제/주문/쿠폰/포인트 처리의 데이터 정합성
* 구독 정기결제의 상태 전이와 실패 복구
* JWT 기반 인증 구조의 보안 한계 보완
* Refresh Token 탈취 및 재사용 대응
* 다중 인스턴스 환경에서의 알림 전달
* 검색/랭킹 기능의 운영 비용 최소화
* 외부 API 장애 전파 차단
* 대용량 배치 처리 중 데이터 누락 방지

따라서 본 문서는 “어떤 기술을 사용했는가”보다 “왜 그 기술을 선택했는가”에 초점을 둔다.

---

# 2. Architecture Principles

## 2.1 문제 해결 적합성을 우선한다

기술은 도입 자체가 목적이 아니다.
One-Stop 프로젝트에서는 현재 프로젝트 규모와 요구사항에 비해 과도한 기술은 의도적으로 배제했다.

예시:

* ElasticSearch 대신 MySQL Full-Text Index 사용
* Kafka 기반 이벤트 파이프라인 대신 Transactional Outbox 우선 적용
* 복잡한 MSA 분리 대신 모듈러 모놀리스 구조 유지
* 외부 API 실패 시 Retry 남발 대신 Fast-Fail 적용
* Refresh Token은 Redis에 저장하되 원문이 아닌 hash 저장

---

## 2.2 정합성이 필요한 구간과 비동기 처리 가능한 구간을 분리한다

모든 작업을 하나의 트랜잭션으로 묶으면 구현은 단순해 보일 수 있지만, 외부 장애가 핵심 비즈니스 흐름까지 전파될 수 있다.

따라서 One-Stop은 다음 기준으로 트랜잭션 경계를 분리했다.

| 구분                          | 처리 방식                                 |
| --------------------------- | ------------------------------------- |
| 주문 생성, 결제 승인, 쿠폰 발급, 포인트 차감 | 정합성 우선                                |
| 구독 정기결제, 상태 전이, 결제 이력 저장    | 멱등성과 추적성 우선                           |
| 알림 발송, 장바구니 정리, 감사로그 저장     | 비동기 처리 가능                             |
| 외부 API 호출                   | Timeout / Circuit Breaker / Fast-Fail |
| 보안 이벤트 기록                   | 인증 흐름과 분리된 비동기 감사로그                   |

---

## 2.3 운영 가능한 복잡도를 유지한다

프로젝트 기간과 팀 규모를 고려하여, 운영 비용을 설명할 수 없는 기술은 도입하지 않았다.

예시:

* 검색은 MySQL Full-Text Index로 시작
* 랭킹은 Redis ZSET으로 처리
* 알림 실시간 전달은 SSE + Redis Pub/Sub 사용
* 보안감사로그는 ApplicationEvent + Async 기반으로 시작
* Kafka/Redis Stream 기반 감사로그 파이프라인은 후속 개선으로 분리

---

## 2.4 Trade-off를 기록한다

모든 기술 선택에는 장점과 단점이 존재한다.
본 프로젝트는 선택한 이유뿐 아니라 선택하지 않은 이유도 함께 기록하여, 향후 개선 시 의사결정 근거로 활용할 수 있도록 했다.

---

# 3. 전체 아키텍처 개요

One-Stop은 단일 쇼핑몰과 오픈 마켓플레이스 성격이 결합된 커머스 플랫폼이다.

주요 도메인은 다음과 같다.

| 도메인            | 주요 책임                                     |
| -------------- | ----------------------------------------- |
| Auth/Security  | 로그인, JWT, RT Rotation, Rate Limit, 보안감사로그 |
| User/Admin     | 회원 상태 관리, 권한 관리, 관리자 조치                   |
| Seller         | 판매자 등록, 상품/리뷰/대시보드 관리                     |
| Order/Payment  | 주문 생성, 결제 승인, 결제 후속 처리                    |
| Subscription   | 구독 상품 가입, 정기결제, 구독 상태 전이, 결제 실패 복구        |
| Coupon         | 선착순 쿠폰 발급, 중복 발급 방지                       |
| Point          | 포인트 적립/사용/만료, 낙관적 락 기반 정합성                |
| Cart           | 회원/비회원 장바구니 관리                            |
| Notification   | 실시간 알림, 알림 이력 저장                          |
| Search/Ranking | 상품 검색, 인기 상품 랭킹                           |
| AI             | 리뷰 요약 등 비핵심 편의 기능                         |

---

# 4. Architecture Evolution

---

## Evolution 1. 선착순 쿠폰 발급 동시성 제어

## As-Is

초기 쿠폰 발급 구조는 Redis 명령을 여러 번 호출하는 방식이었다.

```text
중복 발급 확인
→ 재고 확인
→ 수량 차감
→ 발급 기록 저장
```

각 단계가 분리되어 있어 동시에 여러 요청이 들어오면 다음 문제가 발생할 수 있었다.

* 중복 발급 가능성
* 잔여 수량 음수 처리 가능성
* Redis 왕복 호출 증가
* Race Condition 발생 가능성

## To-Be

Redis Lua Script를 사용하여 쿠폰 발급 검증과 수량 차감을 하나의 원자적 작업으로 처리했다.

```text
Lua Script 내부 처리
1. 중복 발급 여부 확인
2. 쿠폰 재고 확인
3. 재고 차감
4. 발급 기록 저장
5. 결과 반환
```

## Why

선착순 쿠폰 발급은 짧은 시간에 많은 요청이 몰리는 대표적인 동시성 시나리오다.
DB Lock만으로 처리하면 DB 부하가 커질 수 있고, 애플리케이션 레벨에서 여러 Redis 명령을 순차 호출하면 중간 상태가 노출될 수 있다.

Redis Lua Script는 Redis 내부에서 여러 명령을 단일 작업처럼 실행할 수 있으므로, 쿠폰 발급처럼 빠른 원자성이 필요한 작업에 적합하다고 판단했다.

## Result

* 중복 발급 방지
* Race Condition 제거
* Redis 네트워크 왕복 횟수 감소
* 대량 쿠폰 발급 시나리오에서 정합성 확보

---

## Evolution 2. 결제 처리와 후속 이벤트 분리

## As-Is

초기 결제 구조에서는 결제 성공 이후의 후속 작업이 하나의 흐름에 강하게 결합되어 있었다.

결제 성공 이후 수행되는 작업은 다음과 같았다.

* 포인트 적립
* 알림 발송
* 장바구니 정리
* 주문 상태 변경
* 결제 이력 저장

이 구조에서는 알림 발송이나 외부 시스템 장애가 결제 처리 흐름에 영향을 줄 수 있었다.

## To-Be

결제 승인 자체는 정합성을 우선하고, 결제 이후 후속 작업은 이벤트 기반으로 분리했다.

적용한 방식:

* Redisson Distributed Lock
* Transactional Outbox Pattern
* 결제 성공 이벤트 발행
* 후속 작업 비동기 처리

## Why

결제는 금전이 오가는 핵심 트랜잭션이다.
동일 주문에 대한 중복 결제는 반드시 막아야 하지만, 알림 발송이나 장바구니 정리는 결제 승인 트랜잭션과 강하게 결합될 필요가 없다.

따라서 결제 트랜잭션은 짧게 유지하고, 후속 작업은 이벤트로 분리했다.

## Result

* 동일 주문 중복 결제 방지
* 결제 트랜잭션 범위 축소
* 외부 시스템 장애 전파 감소
* 후속 기능 확장성 확보

---

## Evolution 3. 구독 정기결제 상태 전이와 배치 처리

## As-Is

구독 기능은 단순히 사용자가 한 번 결제하고 끝나는 일반 주문/결제와 다르다.

구독은 다음과 같은 특징을 가진다.

* 일정 주기마다 자동 결제가 발생한다.
* 결제 성공 여부에 따라 구독 상태가 변경된다.
* 결제 실패 시 즉시 해지하지 않고 재시도 또는 유예 상태가 필요하다.
* 구독 상태에 따라 사용자 혜택 제공 여부가 달라진다.
* 매일 정해진 시점에 다수의 구독 건을 처리해야 한다.

초기 구조에서 구독 상태 변경과 결제 처리를 단순 서비스 로직으로만 처리하면 다음 문제가 발생할 수 있었다.

* 결제 성공 후 상태 변경 실패
* 상태 변경 후 결제 실패
* 동일 구독 건 중복 결제
* 배치 재실행 시 중복 청구
* 실패 구독 건 추적 어려움
* 구독 상태별 혜택 적용 기준 불명확

## To-Be

구독 도메인은 상태 머신 기반으로 설계하고, 정기결제 처리는 배치 또는 스케줄러 기반으로 분리했다.

핵심 구성은 다음과 같다.

```text
Subscription
- 구독 기본 정보
- 현재 상태
- 다음 결제 예정일
- 구독 시작일/종료일

SubscriptionBilling
- 결제 시도 이력
- 결제 성공/실패 상태
- PG 거래 식별자
- 실패 사유
- 재시도 횟수
```

구독 상태는 명시적인 상태 값으로 관리한다.

```text
TRIALING
→ ACTIVE
→ PAST_DUE
→ CANCELED
→ ENDED
```

정기결제 흐름은 다음과 같이 구성한다.

```text
정기결제 대상 구독 조회
↓
동일 구독 중복 처리 방지
↓
PG 빌링키 기반 결제 요청
↓
결제 성공 시 billing 성공 기록
↓
구독 상태 ACTIVE 유지 또는 갱신
↓
결제 실패 시 billing 실패 기록
↓
구독 상태 PAST_DUE 또는 CANCELED 전이
↓
후속 이벤트 발행
```

## Why

구독은 일반 결제보다 상태 전이가 중요하다.

일반 주문 결제는 하나의 주문에 대해 결제 성공/실패를 판단하면 되지만, 구독은 시간의 흐름에 따라 상태가 계속 변화한다.

따라서 구독 도메인은 다음 기준으로 설계했다.

* 구독 상태를 enum 기반 상태 머신으로 명확히 관리한다.
* 결제 시도 이력은 별도 Billing 테이블에 누적한다.
* 동일 구독 건에 대한 중복 결제를 방지한다.
* 배치 재실행 시에도 멱등성을 유지한다.
* 결제 실패는 즉시 삭제하지 않고 추적 가능한 상태로 남긴다.
* 구독 상태와 혜택 적용 기준을 분리한다.

## Result

* 구독 상태 전이 기준 명확화
* 정기결제 성공/실패 이력 추적 가능
* 배치 재실행 시 중복 결제 위험 감소
* 결제 실패 구독에 대한 후속 대응 가능
* 구독 상태에 따른 혜택 적용 기준 일관화
* 일반 결제와 구독 결제의 책임 분리

---

## Evolution 4. 비회원 장바구니 저장소 전환

## As-Is

초기 비회원 장바구니는 Cookie에 직접 데이터를 저장하는 방식이었다.

하지만 다음 한계가 있었다.

* Cookie 용량 제한
* 요청마다 장바구니 데이터가 전송됨
* 네트워크 Payload 증가
* 클라이언트 데이터 위변조 가능성
* 서버 측 TTL 관리 어려움

## To-Be

비회원 장바구니 데이터를 Redis Hash에 저장하고, Cookie에는 식별자만 유지하는 구조로 변경했다.

```text
Cookie
└ guest_cart_id

Redis
└ cart:guest:{guestCartId}
```

## Why

장바구니는 사용자가 로그인하지 않아도 일정 기간 유지되어야 한다.
하지만 장바구니 상세 데이터를 Cookie에 직접 저장하면 용량과 보안 측면에서 한계가 있다.

Redis는 TTL 기반 만료 처리가 가능하고, key-value 기반으로 빠르게 조회할 수 있으므로 비회원 장바구니 저장소로 적합하다고 판단했다.

## Result

* Cookie Payload 감소
* 장바구니 데이터 위변조 위험 감소
* TTL 기반 자동 만료 지원
* 로그인 후 장바구니 병합 구조 확장 가능

---

## Evolution 5. 대용량 배치 처리 전략 개선

## As-Is

포인트 만료 또는 구독 상태 변경과 같은 배치 작업에서 Offset Paging을 사용할 경우, 처리 중 상태가 변경되면 데이터 누락이 발생할 수 있다.

예를 들어 다음과 같은 흐름이 가능하다.

```text
1페이지 조회
→ 대상 데이터 처리
→ 처리된 데이터는 더 이상 조회 조건에 포함되지 않음
→ 2페이지 조회
→ 일부 데이터가 앞 페이지로 당겨지며 누락 발생
```

## To-Be

Repeatable Page-0 전략을 적용했다.

```text
Page 0 조회
→ 처리
→ 다시 Page 0 조회
→ 처리 대상이 없을 때까지 반복
```

## Why

처리된 데이터가 조회 조건에서 제외되는 배치에서는 항상 첫 페이지를 반복 조회하면 전체 대상 데이터를 순차적으로 소진할 수 있다.

Cursor Reader도 검토했지만 다음 이유로 채택하지 않았다.

* 장시간 DB Connection 유지
* 장애 복구 복잡성 증가
* 대량 처리 중 커넥션 점유 위험
* 운영 관점에서 단순 Paging 방식이 더 적합

## Result

* 상태 변경 배치에서 데이터 누락 방지
* 일정한 메모리 사용량 유지
* Chunk 기반 처리와 궁합이 좋음
* 포인트 만료/구독 상태 변경 작업 안정성 개선

---

## Evolution 6. JWT 인증 구조의 보안성 강화

## As-Is

JWT Access Token은 stateless 구조이므로 서명과 만료 시간이 유효하면 서버는 토큰을 정상으로 판단한다.

이 구조에서는 다음 문제가 발생할 수 있다.

* 비밀번호 변경 후 기존 Access Token이 계속 유효
* 권한 변경 후 기존 권한이 담긴 Access Token이 계속 유효
* 사용자 정지 후 기존 Access Token이 계속 유효
* 강제 로그아웃 후에도 Access Token이 만료 전까지 사용 가능
* Refresh Token 재사용 감지 후에도 기존 Access Token이 남아 있음

## To-Be

JWT 구조에 다음 보안 장치를 추가했다.

* Refresh Token Rotation
* Redis 기반 Refresh Token 저장
* device_id 기반 기기 식별
* User.tokenVersion
* JWT `ver` claim
* user-level token cutoff
* JTI blacklist
* userStatus cache 검증

## Why

순수 JWT의 stateless 장점은 크지만, 보안 이벤트 발생 시 기존 Access Token을 즉시 무효화하기 어렵다.

One-Stop은 단순 인증보다 다음 요구사항을 더 중요하게 판단했다.

* 사용자 정지 즉시 반영
* 비밀번호 변경 시 기존 토큰 무효화
* 권한 변경 시 기존 토큰 무효화
* Refresh Token 재사용 탐지 시 전체 세션 차단
* 관리자 강제 로그아웃 지원

따라서 완전한 stateless JWT 구조를 일부 포기하고, Redis와 tokenVersion을 결합한 하이브리드 인증 구조를 선택했다.

## Result

* 기존 Access Token 무효화 가능
* Refresh Token 재사용 탐지 가능
* 기기별 세션 관리 가능
* 강제 로그아웃 지원
* 사용자 상태 변경 즉시 반영 가능

> 상세 인증/보안 설계는 `documents/domainDesign/auth-security-design.md`를 참고한다.

---

# 5. Architecture Decision Records

---

## ADR-001. JWT Versioning 기반 인증 아키텍처

## Status

Accepted

## Context

JWT는 REST API와 프론트엔드/백엔드 분리 구조에 적합하다.
하지만 순수 JWT는 서버가 발급 이후의 Access Token을 직접 제어하기 어렵다.

다음 요구사항을 만족해야 했다.

* 비밀번호 변경 시 기존 토큰 무효화
* 권한 변경 시 기존 토큰 무효화
* 사용자 정지 시 기존 토큰 차단
* 강제 로그아웃 지원
* Refresh Token 재사용 감지 시 전체 세션 무효화
* 다중 기기 세션 관리

## Options

| 대안               | 장점                  | 한계                 |
| ---------------- | ------------------- | ------------------ |
| Stateful Session | 서버에서 세션 제어 쉬움       | 서버 확장 시 세션 공유 필요   |
| 순수 JWT           | stateless, 확장성 우수   | 기존 AT 즉시 무효화 어려움   |
| JWT Blacklist    | 특정 토큰 차단 가능         | 토큰별 저장소 증가         |
| JWT Versioning   | 사용자 단위 전체 토큰 무효화 가능 | 매 요청 version 조회 필요 |

## Decision

JWT Versioning + Refresh Token Rotation 구조를 선택했다.

구성 요소:

* Access Token: 짧은 수명
* Refresh Token: Redis 저장
* Refresh Token Rotation
* User.tokenVersion
* JWT `ver` claim
* Redis tokenVersion cache
* user-level cutoff
* 기기별 Refresh Token 관리

## Consequences

장점:

* 강제 로그아웃 가능
* 사용자 정지 즉시 반영 가능
* 기존 Access Token 무효화 가능
* Refresh Token 재사용 감지 가능
* 사용자 단위 전체 세션 무효화 가능

단점:

* 완전한 stateless JWT 장점 일부 포기
* Redis/cache 의존성 증가
* tokenVersion cache 무효화 정책 필요

## Trade-off

이 선택은 JWT의 완전한 stateless 장점을 일부 포기하는 대신, 보안 이벤트 발생 시 기존 Access Token을 즉시 무효화할 수 있는 구조를 얻기 위한 결정이다.

One-Stop은 인증 보안성과 운영 대응력을 더 중요하게 판단했다.

---

## ADR-002. Refresh Token Rotation과 Redis CAS Lua Script

## Status

Accepted

## Context

Refresh Token은 Access Token을 재발급할 수 있는 민감한 토큰이다.
고정 Refresh Token 방식에서는 RT가 탈취될 경우 만료 전까지 계속 재사용될 수 있다.

또한 같은 Refresh Token으로 동시에 여러 재발급 요청이 들어오면 중복 재발급 문제가 발생할 수 있다.

## Options

| 대안               | 장점                | 한계             |
| ---------------- | ----------------- | -------------- |
| 고정 Refresh Token | 구현 단순             | 탈취 시 재사용 가능    |
| DB 기반 RT 저장      | 영속성 높음            | 재발급마다 DB 부하 증가 |
| Redis RT 저장      | TTL/삭제 빠름         | Redis 장애 대응 필요 |
| RTR + Redis Lua  | 재사용 탐지, 동시성 제어 가능 | 구현 복잡도 증가      |

## Decision

Refresh Token Rotation을 적용하고, Redis Lua Script로 RT 비교와 교체를 원자적으로 처리한다.

Redis 저장 구조:

```text
refresh:{userId}:{deviceId} = hash(refreshToken)
```

재발급 시 처리:

```text
1. 요청 RT hash 계산
2. Redis 저장 RT hash 조회
3. Lua Script에서 기존 hash와 요청 hash 비교
4. 일치하면 새 RT hash로 교체
5. 불일치하면 재사용 의심으로 차단
```

## Consequences

장점:

* Refresh Token 원문 저장 방지
* RT 재사용 탐지 가능
* 동시 재발급 충돌 방지
* 기기별 RT 삭제 가능

단점:

* Redis 의존성 증가
* Redis 장애 시 fail-open/fail-closed 정책 필요
* 사용자 경험과 보안 사이의 정책 결정 필요

---

## ADR-003. device_id 기반 다중 기기 세션 관리

## Status

Accepted

## Context

사용자는 여러 기기에서 로그인할 수 있다.
Refresh Token을 사용자 단위로만 관리하면 특정 기기 로그아웃, 기기별 세션 제한, 오래된 기기 제거가 어렵다.

## Options

| 대안                      | 장점                  | 한계            |
| ----------------------- | ------------------- | ------------- |
| 사용자 단위 RT 1개            | 구현 단순               | 다중 기기 지원 어려움  |
| DB 기반 기기 세션             | 영속성 높음              | 인증 요청마다 DB 부하 |
| Redis ZSET 기반 device 관리 | 빠른 조회/갱신, LRU 구현 쉬움 | Redis 의존성     |

## Decision

device_id Cookie와 Redis ZSET을 사용해 기기별 세션을 관리한다.

구조:

```text
devices:{userId} = ZSET(deviceId, lastUsedAt)
refresh:{userId}:{deviceId} = hash(refreshToken)
```

정책:

* 사용자당 최대 5개 기기 허용
* 6번째 기기 로그인 시 가장 오래된 기기 제거
* 제거된 기기의 Refresh Token 삭제
* 로그아웃 시 해당 기기의 Refresh Token 삭제
* 강제 로그아웃 시 전체 기기 RT 삭제

## Consequences

장점:

* 기기별 세션 관리 가능
* 오래된 기기 LRU 제거 가능
* 전체 로그아웃/개별 로그아웃 구분 가능
* RT와 device_id binding 검증 가능

단점:

* Cookie 기반 device_id 관리 필요
* device_id 원문 로그 노출 방지 필요
* Redis 데이터 정합성 관리 필요

---

## ADR-004. Redis Counter + TTL 기반 Rate Limit

## Status

Accepted

## Context

Auth/Security 영역에서는 단순 API 트래픽 제어보다 보안 정책으로서의 Rate Limit이 필요했다.

필요한 제한은 다음과 같다.

* 동일 계정 반복 로그인 제한
* 동일 IP 반복 로그인 제한
* 신규 기기 등록 제한
* Refresh Token 재발급 남용 제한
* 로그아웃 요청 남용 제한
* Rate Limit 차단 감사로그 기록

## Options

| 대안                         | 장점                | 한계                          |
| -------------------------- | ----------------- | --------------------------- |
| Token Bucket               | 일반 API 트래픽 제어에 적합 | 로그인 실패 누적/성공 초기화 정책에 덜 직관적  |
| Sliding Window             | 정밀한 제한 가능         | Redis 상태 관리 복잡              |
| Fixed Window / Counter TTL | 구현 단순, TTL 관리 쉬움  | 시간 경계 burst 가능              |
| Nginx only                 | 빠른 1차 차단          | 계정/userId/deviceId 기반 판단 불가 |

## Decision

Spring/Redis Rate Limit은 Redis Counter + TTL 기반 Fixed Window 계열로 구현한다.
Redis Lua Script를 사용하여 counter 증가, TTL 설정, 초과 여부 판단을 원자적으로 처리한다.

Nginx는 IP/경로 기반 1차 방어, Spring/Redis는 계정/기기/토큰 기반 2차 방어로 역할을 분리한다.

## Consequences

장점:

* 로그인 실패 누적에 적합
* TTL 기반 자동 해제 가능
* 성공 시 카운터 초기화 가능
* Redis Lua로 원자 처리 가능
* 계정/userId/deviceId 기반 정책 가능

단점:

* 시간 경계 burst 가능
* 일반 API 트래픽 shaping에는 Token Bucket보다 부드럽지 않음
* 운영/부하테스트 설정값 분리 필요

---

## ADR-005. 보안감사로그 비동기 저장과 PII 최소화

## Status

Accepted

## Context

Auth/Security 영역에서는 보안 이벤트 추적이 필요하다.

기록 대상:

* 로그인 성공/실패
* Rate Limit 차단
* Refresh Token 재사용 의심
* RT-device mismatch
* 관리자 보안 조치
* 사용자 정지/해제
* 강제 로그아웃

하지만 감사로그를 동기적으로 DB에 저장하면 공격 상황에서 인증 요청이 감사로그 DB I/O에 묶일 수 있다.

또한 감사로그는 보안 분석을 위한 자료이지만, 잘못 설계하면 민감정보 저장소가 될 수 있다.

## Options

| 대안                       | 장점         | 한계                      |
| ------------------------ | ---------- | ----------------------- |
| 동기 DB 저장                 | 유실 적음      | 인증 흐름 지연, 공격 시 DB 부하 증가 |
| ApplicationEvent + Async | 인증 흐름과 분리  | 큐 포화 시 유실 가능            |
| Kafka/Redis Stream       | 내구성/확장성 우수 | 인프라 복잡도 증가              |
| 로그 파일 기반                 | 구현 단순      | 검색/분석/관리 어려움            |

## Decision

ApplicationEvent + Async 기반으로 보안감사로그를 저장한다.

PII 최소화를 위해 다음 정책을 적용한다.

| 항목           | 저장 방식                        |
| ------------ | ---------------------------- |
| IP           | AES 암호화 + HMAC hash + prefix |
| User-Agent   | HMAC hash                    |
| device_id    | HMAC hash                    |
| Token        | 저장 금지                        |
| Cookie       | 저장 금지                        |
| Email        | 원문 저장 금지                     |
| Request Body | 저장 금지                        |

큐 포화 시 Tomcat 요청 스레드가 감사로그 저장을 대신 수행하지 않도록 `CallerRunsPolicy`는 사용하지 않는다.
대신 drop counter를 통해 유실 수를 관측한다.

## Consequences

장점:

* 인증 흐름과 감사로그 저장 분리
* 공격 상황에서 요청 스레드 보호
* PII 원문 저장 위험 감소
* 사후 보안 분석 가능

단점:

* 비동기 큐 포화 시 감사로그 유실 가능
* 유실 관측 및 알림 체계 필요
* Kafka/Redis Stream 대비 내구성은 낮음

---

## ADR-006. 구독 정기결제 상태 머신 아키텍처

## Status

Accepted

## Context

구독 도메인은 일반 주문/결제와 다르게 시간 기반으로 상태가 변경된다.

일반 결제는 사용자의 요청 시점에 한 번 결제가 발생하지만, 구독은 다음과 같은 요구사항을 가진다.

* 매월 또는 정해진 주기마다 자동 결제 발생
* 결제 성공/실패에 따른 구독 상태 전이
* 결제 실패 시 유예 상태 관리
* 구독 상태에 따른 혜택 제공 여부 판단
* 정기결제 배치 재실행 시 멱등성 보장
* 동일 구독에 대한 중복 결제 방지
* PG 빌링키 기반 결제 요청 관리

따라서 단순 결제 로직과 동일하게 처리하면 상태 정합성과 재처리 안정성을 보장하기 어렵다.

## Options

| 대안                       | 장점                 | 한계                  |
| ------------------------ | ------------------ | ------------------- |
| 주문/결제 로직에 구독 처리 포함       | 구현이 단순해 보임         | 일반 결제와 정기결제 책임 혼재   |
| 구독 상태 없이 결제 이력만 관리       | 테이블 단순             | 혜택 제공 기준과 실패 복구 어려움 |
| 구독 상태 머신 + Billing 이력 분리 | 상태 전이 명확, 실패 추적 가능 | 설계 복잡도 증가           |
| 외부 구독 SaaS 의존            | 구현 부담 감소           | 정책 커스터마이징과 비용 부담    |

## Decision

구독 도메인은 상태 머신 기반으로 관리하고, 정기결제 시도 이력은 별도 Billing 모델로 분리한다.

구독 상태는 다음과 같이 관리한다.

```text
TRIALING
ACTIVE
PAST_DUE
CANCELED
ENDED
```

각 상태의 의미는 다음과 같다.

| 상태       | 의미                    |
| -------- | --------------------- |
| TRIALING | 체험 또는 초기 구독 상태        |
| ACTIVE   | 정상 구독 상태              |
| PAST_DUE | 결제 실패로 유예 중인 상태       |
| CANCELED | 사용자가 취소했거나 정책상 취소된 상태 |
| ENDED    | 구독 기간이 종료된 상태         |

정기결제는 다음 기준으로 처리한다.

* 결제 대상 구독을 일정 주기로 조회한다.
* 동일 구독 건에 대해 중복 결제가 발생하지 않도록 제어한다.
* 결제 시도마다 Billing 이력을 생성한다.
* PG 응답을 기반으로 성공/실패 상태를 기록한다.
* 성공 시 다음 결제 예정일을 갱신한다.
* 실패 시 재시도 가능 상태 또는 PAST_DUE 상태로 전이한다.
* 최종 실패 시 CANCELED 또는 ENDED 상태로 전이한다.

## Implementation

구독 도메인의 핵심 모델은 다음과 같이 분리한다.

```text
Subscription
- 사용자 ID
- 구독 상품 ID
- 현재 상태
- 시작일
- 종료일
- 다음 결제 예정일
- 취소일

SubscriptionBilling
- 구독 ID
- 결제 시도 금액
- 결제 상태
- PG 거래 ID
- 실패 사유
- 재시도 횟수
- 요청 시각
- 응답 시각
```

정기결제 배치는 다음 흐름으로 동작한다.

```text
결제 예정일이 도래한 ACTIVE/PAST_DUE 구독 조회
↓
구독 단위 중복 처리 방지
↓
Billing 결제 시도 이력 생성
↓
PG 빌링키 결제 요청
↓
결제 성공/실패 결과 저장
↓
Subscription 상태 및 다음 결제일 갱신
↓
후속 이벤트 발행
```

## Consequences

장점:

* 구독 상태 전이가 명확해진다.
* 결제 성공/실패 이력을 추적할 수 있다.
* 결제 실패 후 재시도 또는 유예 정책을 적용할 수 있다.
* 구독 상태에 따른 혜택 제공 기준이 명확해진다.
* 정기결제 배치 재실행 시 멱등성 검증이 쉬워진다.
* 일반 결제와 구독 결제의 책임이 분리된다.

단점:

* 일반 결제보다 도메인 모델이 복잡하다.
* 상태 전이 테스트가 필요하다.
* PG 장애 시 재시도/보상 정책이 필요하다.
* 배치 중복 실행 방지 전략이 필요하다.

## Trade-off

구독은 단순 결제가 아니라 시간 기반 상태 관리 도메인이다.

따라서 구현 단순성을 위해 일반 결제 흐름에 포함시키기보다, 상태 머신과 Billing 이력을 분리해 정합성과 추적 가능성을 우선했다.

이 선택은 초기 설계 복잡도는 증가시키지만, 결제 실패 복구, 구독 혜택 적용, 재처리 안정성 측면에서 더 적합하다고 판단했다.

---

## ADR-007. 실시간 알림 아키텍처

## Status

Accepted

## Context

사용자에게 주문, 결제, 쿠폰, 리뷰 등 이벤트 기반 알림을 전달해야 한다.
또한 멀티 인스턴스 환경에서는 사용자가 어떤 서버에 연결되어 있는지 알 수 없으므로 인스턴스 간 이벤트 전달 수단이 필요하다.

## Options

| 대안            | 장점                | 한계                   |
| ------------- | ----------------- | -------------------- |
| WebSocket     | 양방향 통신 가능         | 연결 관리 복잡             |
| SSE           | 단방향 알림에 단순하고 적합   | 클라이언트 → 서버 양방향에는 부적합 |
| Redis Pub/Sub | 멀티 인스턴스 이벤트 전달 쉬움 | 메시지 영속성 없음           |
| Kafka         | 영속성/확장성 우수        | 운영 복잡도 증가            |

## Decision

SSE + Redis Pub/Sub + Notification RDB 구조를 선택했다.

역할 분리:

```text
SSE
→ 클라이언트 실시간 전달

Redis Pub/Sub
→ 멀티 인스턴스 이벤트 브로드캐스트

Notification RDB
→ 알림 이력 영속 저장
```

## Consequences

장점:

* 단방향 실시간 알림에 적합
* 멀티 인스턴스 환경 지원
* 알림 이력 조회 가능
* Kafka 없이 구현 가능

단점:

* Redis Pub/Sub 자체는 메시지 영속성 없음
* 오프라인 사용자는 실시간 이벤트를 직접 수신하지 못함
* 재접속 시 RDB 조회 기반 보완 필요

---

## ADR-008. 검색 및 랭킹 아키텍처

## Status

Accepted

## Context

상품 검색과 인기 상품 랭킹 기능이 필요했다.

ElasticSearch 도입도 검토했지만, 프로젝트 규모와 운영 비용을 고려해야 했다.

## Options

| 대안                    | 장점             | 한계                |
| --------------------- | -------------- | ----------------- |
| LIKE 검색               | 구현 단순          | 성능/정확도 한계         |
| MySQL Full-Text Index | 별도 인프라 불필요     | 검색 품질 한계          |
| ElasticSearch         | 검색 품질과 확장성 우수  | 인프라/동기화/운영 복잡도 증가 |
| Redis ZSET 랭킹         | 점수 기반 랭킹 처리 쉬움 | 영속성/동기화 고려 필요     |

## Decision

검색은 MySQL Full-Text Index로 처리하고, 랭킹은 Redis ZSET을 사용한다.

랭킹 데이터는 사용자 요청 경로에서 직접 DB Write를 증가시키지 않고, Redis에 집계 후 배치로 RDB에 동기화한다.

## Consequences

장점:

* 별도 검색 인프라 없이 검색 기능 제공
* 운영 복잡도 감소
* 인기 상품 랭킹 집계에 Redis ZSET 활용 가능
* 사용자 요청 경로의 DB Write 감소

단점:

* ElasticSearch 대비 검색 고도화 한계
* 형태소 분석, 오타 보정, 복잡한 relevance tuning 어려움
* 랭킹 데이터 동기화 정책 필요

---

## ADR-009. 외부 API 장애 격리

## Status

Accepted

## Context

AI 요약, 외부 결제, 소셜 로그인 등 외부 시스템과의 연동이 존재한다.
외부 API 응답 지연이나 장애가 내부 서비스의 스레드/커넥션 자원을 점유하면 전체 장애로 확산될 수 있다.

## Options

| 대안              | 장점          | 한계               |
| --------------- | ----------- | ---------------- |
| 무제한 대기          | 구현 단순       | 장애 전파 위험         |
| Retry           | 일시 장애 복구 가능 | 과도한 Retry는 장애 증폭 |
| Timeout         | 자원 점유 시간 제한 | 실패율 증가 가능        |
| Circuit Breaker | 장애 전파 차단    | 설정 및 상태 관리 필요    |
| Fast-Fail       | 빠른 실패 처리    | 사용자에게 즉시 실패 응답   |

## Decision

외부 API 연동에는 Timeout + Circuit Breaker + Fast-Fail 전략을 적용한다.

정책:

* Connection Timeout 설정
* Read Timeout 설정
* 오류율 임계치 초과 시 Circuit Open
* 핵심 비즈니스가 아닌 기능은 Retry보다 Fast-Fail 우선
* 실패 시 fallback 메시지 또는 기능 제한 응답

## Consequences

장점:

* 외부 장애 전파 범위 감소
* 스레드/커넥션 자원 보호
* 장애 상황에서 빠른 실패 처리 가능
* 운영 안정성 향상

단점:

* 임계치 설정 필요
* 일시적 장애에도 실패 응답 가능
* 사용자 경험과 안정성 사이의 trade-off 존재

---

# 6. 기술 선택 요약

| 문제          | 선택 기술                     | 선택 이유                      |
| ----------- | ------------------------- | -------------------------- |
| 쿠폰 중복 발급    | Redis Lua Script          | 재고 확인/차감/발급 기록을 원자 처리      |
| 중복 결제       | Redisson Lock             | 동일 주문 결제 중복 처리 방지          |
| 결제 후속 처리    | Transactional Outbox      | 핵심 트랜잭션과 후속 작업 분리          |
| 구독 상태 관리    | 상태 머신                     | 정기결제 성공/실패에 따른 상태 전이 명확화   |
| 구독 결제 이력    | SubscriptionBilling 분리    | 결제 시도, 실패 사유, 재시도 이력 추적    |
| 정기결제 처리     | Scheduler/Batch           | 결제 예정 구독을 주기적으로 처리         |
| 구독 중복 결제 방지 | 구독 단위 멱등성 제어              | 배치 재실행 또는 중복 실행 시 중복 청구 방지 |
| 비회원 장바구니    | Redis Hash                | Cookie Payload 감소, TTL 관리  |
| 배치 데이터 누락   | Repeatable Page-0         | 상태 변경 배치에서 Offset drift 방지 |
| 인증 구조       | JWT + RT                  | REST API와 프론트/백 분리 구조에 적합  |
| 기존 AT 무효화   | tokenVersion              | 사용자 단위 토큰 무효화 가능           |
| RT 재사용 방지   | Refresh Token Rotation    | 탈취/재사용 탐지                  |
| RT 동시성 제어   | Redis CAS Lua             | RT 비교/교체 원자 처리             |
| 다중 기기 관리    | device_id + Redis ZSET    | 기기별 세션 관리                  |
| 로그인 남용 방지   | Redis Counter + TTL       | 실패 누적/TTL 관리에 적합           |
| 1차 요청 차단    | Nginx Rate Limit          | Spring 진입 전 대량 요청 차단       |
| 보안 이벤트 추적   | SecurityAuditLog          | 인증 이벤트 사후 분석               |
| 감사로그 저장     | Async Event               | 인증 흐름과 로그 저장 분리            |
| 알림          | SSE + Redis Pub/Sub       | 단방향 실시간 알림과 멀티 인스턴스 지원     |
| 검색          | MySQL Full-Text Index     | 별도 검색 인프라 없이 구현            |
| 랭킹          | Redis ZSET                | 점수 기반 랭킹 집계                |
| 외부 장애 격리    | Timeout + Circuit Breaker | 장애 전파 차단                   |

---

# 7. 후속 개선 사항

현재 구조는 프로젝트 기간과 운영 복잡도를 고려한 MVP 기반 아키텍처이다.
향후 서비스 규모가 커질 경우 다음 개선을 고려할 수 있다.

| 영역             | 후속 개선                                                               |
| -------------- | ------------------------------------------------------------------- |
| Auth/Security  | Nginx Rate Limit 고도화, WAF 연동                                        |
| Security Audit | Kafka 또는 Redis Stream 기반 내구성 있는 이벤트 파이프라인                           |
| Rate Limit     | 일반 API에 Token Bucket/Bucket4j 적용                                    |
| Subscription   | 결제 실패 재시도 정책 고도화, PAST_DUE 유예 기간 세분화, 정기결제 배치 모니터링, 구독 상태 전이 테스트 강화 |
| Search         | ElasticSearch/OpenSearch 도입                                         |
| Notification   | Kafka 기반 알림 이벤트 처리                                                  |
| Batch          | Job 모니터링 및 재처리 관리 고도화                                               |
| Payment        | PG Webhook 멱등성 및 보상 트랜잭션 강화                                         |
| Observability  | Prometheus/Grafana 기반 지표 대시보드 확장                                    |
| Deployment     | Blue-Green 또는 Rolling 배포 전략 고도화                                     |

---

# 8. 관련 문서

* `documents/domainDesign/auth-security-design.md`
* `documents/domainDesign/user-design.md`
* `documents/domainDesign/seller-design.md`
* `documents/domainDesign/subscription-design.md`
* `documents/domainDesign/order-payment-design.md`
* `documents/architecture/technology-decisions.md`
* `documents/troubleshooting/personalTroubleshooting/hojin/auth-security-troubleshooting.md`
* `documents/policy/ServicePolicy/auth-policy.md`
* `documents/policy/ServicePolicy/subscription-policy.md`
* `documents/api/domainApi/auth-api.md`
* `documents/api/domainApi/subscription-api.md`

