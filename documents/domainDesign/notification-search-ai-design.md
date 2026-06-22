# Notification/Search/AI 도메인 설계문서

## 1. 도메인 목적

이 문서는 Notification, Search/Ranking, AI 기능의 설계를 함께 정리한다.

세 도메인은 핵심 주문/결제 트랜잭션과 직접 결합되기보다 사용자 경험을 개선하는 보조 기능에 가깝다. 따라서 정합성보다 장애 격리와 운영 단순성을 우선한다.

---

# Part 1. Notification

## 2. Notification 목적

Notification 도메인은 주문, 결제, 쿠폰, 리뷰 등 서비스 이벤트를 사용자에게 실시간 또는 이력 형태로 전달한다.

## 3. 주요 책임

| 책임 | 설명 |
|---|---|
| 실시간 알림 | SSE 기반 클라이언트 알림 전송 |
| 멀티 인스턴스 전달 | Redis Pub/Sub 기반 이벤트 전달 |
| 알림 이력 저장 | Notification RDB 저장 |
| 재접속 보완 | 사용자가 놓친 알림 조회 가능 |

## 4. 알림 흐름

```text
도메인 이벤트 발생
↓
Notification 생성 및 DB 저장
↓
Redis Pub/Sub 발행
↓
해당 사용자가 연결된 서버가 이벤트 수신
↓
SSE로 클라이언트 전달
```

## 5. 기술 선택 근거

### SSE

알림은 서버에서 클라이언트로 전달하는 단방향 이벤트가 대부분이다. WebSocket보다 연결 관리가 단순하고 HTTP 기반으로 동작하므로 SSE를 선택했다.

### Redis Pub/Sub

멀티 인스턴스 환경에서는 사용자가 어떤 서버에 연결되어 있는지 알 수 없다. Redis Pub/Sub을 사용해 모든 인스턴스에 알림 이벤트를 브로드캐스트한다.

### Notification RDB

Redis Pub/Sub은 메시지 영속성을 제공하지 않는다. 따라서 알림 이력은 RDB에 저장하고, 사용자가 재접속하면 놓친 알림을 조회할 수 있도록 한다.

---

# Part 2. Search/Ranking

## 6. Search/Ranking 목적

Search/Ranking 도메인은 상품 검색과 인기 상품 랭킹 조회를 담당한다.

## 7. 주요 책임

| 책임 | 설명 |
|---|---|
| 상품 검색 | 키워드 기반 상품 조회 |
| 랭킹 집계 | 조회/구매/리뷰 등 점수 기반 랭킹 산정 |
| 랭킹 조회 | 인기 상품 목록 제공 |
| 집계 동기화 | Redis 랭킹 데이터를 RDB와 동기화 |

## 8. 기술 선택 근거

### MySQL Full-Text Index

ElasticSearch/OpenSearch는 검색 품질과 확장성 측면에서 강력하지만, 별도 인프라 운영과 데이터 동기화 파이프라인이 필요하다.

프로젝트 규모에서는 MySQL Full-Text Index로 기본 검색 요구사항을 충족하고 운영 복잡도를 줄이는 것이 더 적합하다고 판단했다.

### Redis ZSET

랭킹은 점수 기반 정렬이 핵심이다. Redis ZSET은 score 기준 정렬과 top-N 조회에 적합하므로 인기 상품 랭킹 집계에 사용한다.

```text
ranking:products
  productId -> score
```

---

# Part 3. AI

## 9. AI 목적

AI 도메인은 리뷰 요약 등 사용자 편의 기능을 제공한다.

핵심 주문/결제 흐름이 아니므로, AI 기능 장애가 서비스 전체 장애로 확산되지 않도록 격리한다.

## 10. 주요 책임

| 책임 | 설명 |
|---|---|
| 리뷰 요약 | 상품 리뷰를 기반으로 요약 생성 |
| 외부 API 호출 | AI Provider API 연동 |
| 비용 추적 | AI 토큰 사용량 또는 호출량 추적 |
| 장애 격리 | Timeout, Circuit Breaker, Fast-Fail 적용 |

## 11. 기술 선택 근거

### Fast-Fail

AI 요약은 핵심 결제/주문 기능이 아니므로 외부 API 장애 시 Retry를 반복하기보다 빠르게 실패시키는 것이 더 안전하다.

### Timeout/Circuit Breaker

외부 AI API 응답 지연이 서버 스레드를 장시간 점유하면 전체 서비스에 영향을 줄 수 있다. Timeout과 Circuit Breaker를 적용해 장애 전파를 차단한다.

### 비용 추적

AI API는 호출량에 따라 비용이 발생하므로 요청 수, 토큰 사용량, 실패율 등을 추적할 필요가 있다.

---

# 12. 공통 정합성/운영 고려사항

| 영역 | 고려사항 |
|---|---|
| Notification | Pub/Sub 유실 가능성을 RDB 이력으로 보완 |
| Search | 검색 품질보다 운영 단순성을 우선 |
| Ranking | Redis 집계와 RDB 동기화 정책 필요 |
| AI | 외부 장애가 내부 핵심 기능으로 전파되지 않도록 격리 |

# 13. 예외 처리

| 상황 | 처리 |
|---|---|
| SSE 연결 종료 | 재연결 유도 |
| Pub/Sub 유실 | RDB 알림 이력 조회로 보완 |
| 검색 결과 없음 | 빈 목록 반환 |
| Redis 랭킹 장애 | 기본 상품 목록 fallback |
| AI API Timeout | 요약 실패 응답 또는 fallback 메시지 |
| Circuit Open | 즉시 실패 처리 |

# 14. 한계 및 후속 개선

| 한계 | 후속 개선 |
|---|---|
| Redis Pub/Sub 영속성 없음 | Kafka 또는 Redis Stream 도입 검토 |
| MySQL Full-Text 검색 품질 한계 | OpenSearch/ElasticSearch 도입 검토 |
| AI 기능 장애 관측 제한 | AI 호출량/비용/실패율 대시보드 추가 |
| 랭킹 동기화 정책 단순 | 배치 주기 및 점수 산식 고도화 |
| SSE 연결 관리 고도화 필요 | heartbeat, timeout, 재연결 정책 강화 |
