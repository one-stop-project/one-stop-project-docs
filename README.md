# 🛒 OneStop Docs

> OneStop 프로젝트의 설계 문서, 정책 문서, API 명세, ERD, 테스트 및 기술 검증 문서를 관리하는 문서 저장소입니다.
>
> OneStop은 쿠팡형 커머스와 판매자 입점형 마켓플레이스를 결합한 하이브리드 이커머스 플랫폼으로, 대규모 트래픽 환경에서의 정합성 보장과 확장 가능한 아키텍처 설계를 목표로 개발되었습니다.

---

# 📄 핵심 문서

| 문서 | 설명 |
|--------|--------|
| [서비스 정책](./documents/policy) | 도메인별 서비스 정책 및 비즈니스 규칙 |
| [팀 협업 정책](./documents/policy/TeamPolicy.md) | Git 전략, 코드 리뷰, 브랜치 운영 정책 |
| [ERD 설계](./documents/erd/ERD.md) | 데이터베이스 구조 및 관계 설계 |
| [API 명세](./documents/api/API.md) | REST API 명세 |
| [도메인 설계](./documents/domainDesign) | 도메인별 설계 문서 |
| [비즈니스 플로우](./documents/business-flow/BusinessFlow.md) | 주요 업무 흐름 및 상태 전이 |
| [시퀀스 다이어그램](./documents/sequence/domain-sequence-diagrams.md) | 도메인별 시퀀스 다이어그램 |
| [ADR](./documents/architecture/ArchitectureDecisions.md) | 기술 의사결정 기록 |
| [성능 검증](./documents/performanceVerfication/performance-reliability-verification.md) | 부하 테스트 및 성능 검증 |
| [테스트 문서](./documents/test/TestCases.md) | 테스트 케이스 및 테스트 전략 |
---

# 🚀 핵심 구현 포인트

## 1. Kafka + Transactional Outbox

* Outbox 패턴 기반 이벤트 발행
* Kafka 비동기 메시징 적용
* 이벤트 유실 방지
* 주문 · 포인트 · 쿠폰 · 알림 이벤트 처리

## 2. 동시성 제어

* Redis Distributed Lock
* 비관적 락(Pessimistic Lock)
* 낙관적 락(Optimistic Lock)

비즈니스 특성에 따라 적절한 동시성 제어 전략 적용

## 3. AI Shopping Assistant

* Spring AI Tool Calling
* 상품 검색 API 연동
* 재고 조회 API 연동
* 구조화 출력(JSON Response)

## 4. AI 리뷰 요약

* 상품 리뷰 자동 요약
* 장점 / 단점 분석
* 구매 추천 정보 생성

## 5. SSE 실시간 알림

* 주문 상태 변경
* 배송 상태 변경
* 쿠폰 지급
* 포인트 적립

## 6. Scheduler 기반 자동 처리

* 구독 자동 갱신
* 구독 만료 처리
* 포인트 만료 처리
* 쿠폰 만료 처리

---

# 🗂️ 문서 구조

```text
documents
├── api
├── architecture
├── business-flow
├── domainDesign
├── erd
├── finOps
├── incidentResponse
├── meetings
├── milestone
├── performanceVerfication
├── policy
├── sequence
├── test
└── troubleshooting
```

---

# 🚨 트러블슈팅

| 구분 | 설명 |
|--------|--------|
| [팀 트러블슈팅](./documents/troubleshooting/teamTroubleshooting) | 프로젝트 진행 중 발생한 기술 문제 해결 기록 |
| [개인 트러블슈팅](./documents/troubleshooting/personalTroubleshooting) | 팀원별 문제 해결 및 학습 기록 |
---

# 📝 회의록

프로젝트 진행 과정의 의사결정과 논의 내용을 기록합니다.

- [회의록 바로가기](./documents/meetings)

---

# 📊 운영 및 품질 관리 문서

| 문서 | 설명 |
|--------|--------|
| [FinOps](./documents/finOps) | 비용 최적화 전략 |
| [Incident Response](./documents/incidentResponse) | 장애 대응 및 운영 절차 |
| [Milestone](./documents/milestone/Milestone.md) | 프로젝트 일정 및 마일스톤 |
| [Test](./documents/test) | 테스트 케이스 및 커버리지 |
---

# 🧠 주요 기술 키워드

* Java 21
* Spring Boot 3
* Spring Security
* JWT
* QueryDSL
* MySQL
* Redis
* Apache Kafka
* Transactional Outbox
* SSE
* Spring AI
* Resilience4j
* AWS ECS Fargate
* Docker
* GitHub Actions
* Prometheus
* Grafana
* K6
* Toss Payments

---

# 📅 프로젝트 기간

```text
2026.05.11 ~ 2026.06.24
```
- [Milestone](./documents/milestone/Milestone.md)
---

# 👥 팀원

| 이름 | 역할 | 핵심 기술 |
| --- | ------------------------ |------------------------------------------------------------------------------------------------------------------------|
| 정은지 | 팀장 / Infra / AI | 관리자 기능, GitHub Actions 기반 CI/CD, Prometheus·Grafana 모니터링, Spring AI(Gemini) Tool Calling, 연관상품 추천(카테고리·판매량 기반)·리뷰 요약 |
| 임호진 | Auth / Seller / Member | 인증·인가·보안 아키텍처, JWT + Refresh Token Rotation(RTR), OAuth2(Kakao), Redis Fixed Window Rate Limit, 보안 감사 로그, 회원·판매자 라이프사이클 |
| 정지훈 | Cart / Order / Payment / Coupon / Notification | 장바구니 → 주문 → 결제 구매 플로우, 비회원 장바구니 Redis Hash + ZSet, 쿠폰(Lua·DECR·Redisson Lock)·포인트(낙관적 락·재시도) 정합성, Outbox-Kafka 이벤트 처리, Redis Pub/Sub + SSE 실시간 알림 |
| 이중현 | Product / Search | 상품·카테고리, QueryDSL 기반 상품 검색, Redis 캐싱, 인기 랭킹·검색어·조회수 집계, MySQL FULLTEXT 인덱스, AI 기반 더미 데이터 |
| 김예은 | Delivery / Review / Subscription | 배송 상태 관리, 리뷰 정합성, 정기결제 자동화, Outbox-Kafka 이벤트 처리(배송 완료 → 포인트 적립) |
---

# 📌 문서 관리 기준

* 코드와 문서를 분리하여 관리합니다.
* 정책, API, ERD 변경 시 관련 문서를 함께 업데이트합니다.
* ADR을 통해 주요 기술 의사결정 이력을 관리합니다.
* 트러블슈팅은 문제 상황, 원인, 해결 방법, 결과를 기준으로 작성합니다.
* 회의록은 프로젝트 의사결정 이력을 기록하는 용도로 관리합니다.

---

# ⚖️ License

This documentation is licensed under the MIT License.

See the LICENSE file for details.
