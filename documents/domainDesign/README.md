# Domain Design Documents

이 폴더는 One-Stop 프로젝트의 도메인별 상세 설계를 정리한다.

기존 `documents/policy/ServicePolicy`가 “서비스 규칙과 정책값”을 정의한다면, 본 폴더의 문서는 “왜 그렇게 설계했는가”와 “각 도메인이 어떤 책임을 가지는가”를 설명한다.

## 문서 목록

| 문서 | 설명 |
|---|---|
| `auth-security-design.md` | JWT, RT Rotation, device_id, tokenVersion, Rate Limit, 보안감사로그 |
| `user-design.md` | 회원 상태, 권한, 주소/계좌 정보, 관리자 보안 조치 |
| `seller-design.md` | 판매자 등록, 승인, 판매자 대시보드, 리뷰 조회, 소유권 검증 |
| `order-payment-design.md` | 주문 생성, 결제 승인, 중복 결제 방지, Outbox 기반 후속 처리 |
| `subscription-design.md` | 구독 상태 머신, 정기결제, Billing 이력, 실패 복구 |
| `coupon-point-design.md` | 쿠폰 발급 동시성, 포인트 사용/적립/만료, 배치 처리 |
| `cart-design.md` | 회원/비회원 장바구니, Redis 기반 비회원 장바구니, 로그인 병합 |
| `notification-search-ai-design.md` | SSE 알림, 검색/랭킹, AI 리뷰 요약 및 외부 장애 격리 |

## 공통 작성 기준

각 도메인 문서는 다음 관점으로 작성한다.

1. 도메인 목적
2. 주요 책임
3. 핵심 기능
4. 주요 흐름
5. 데이터 모델
6. 기술 선택 근거
7. 정합성/보안 고려사항
8. 예외 처리
9. 한계 및 후속 개선

## 관련 문서

- `documents/architecture/architecture-adr.md`
- `documents/architecture/technology-decisions.md`
- `documents/policy/ServicePolicy/*.md`
- `documents/api/domainApi/*.md`
- `documents/troubleshooting/`
