# Team Policy

> OneStop 프로젝트 협업 규칙 및 개발 프로세스

---

# 1. 브랜치 전략

Git Flow 기반 브랜치 전략을 사용한다.

## 브랜치 구조

```text
main
 └── develop
      └── feat/{feature-name}-{issue-number}
```

### 예시

```text
feat/order-create-12
feat/payment-approve-24
feat/coupon-issue-31
```

## 규칙

- `main` 브랜치 직접 Push 금지
- 모든 개발은 `develop` 기준으로 진행
- 기능 단위 브랜치 생성 후 작업
- Pull Request 승인 후 Merge
- Merge 완료 후 Feature 브랜치 삭제

---

# 2. 커밋 컨벤션

## 형식

```text
type: subject
```

### 예시

```text
feat: 주문 생성 API 구현
fix: 재고 차감 동시성 문제 수정
refactor: OrderService 책임 분리
test: 주문 생성 테스트 추가
docs: 정책 문서 수정
chore: 의존성 버전 업데이트
infra: GitHub Actions 설정
```

## 타입

| Type | 설명 |
|--------|--------|
| feat | 기능 개발 |
| fix | 버그 수정 |
| refactor | 리팩토링 |
| test | 테스트 |
| docs | 문서화 |
| chore | 기타 설정 |
| infra | 인프라 |
| ai | AI 기능 |

---

# 3. Pull Request 규칙

## 제목 형식

```text
[feat] 주문 생성 API 구현
[fix] 쿠폰 중복 발급 오류 수정
[refactor] PaymentService 분리
```

## PR 템플릿

```markdown
## 작업 내용

- 구현 내용 작성

## 관련 이슈

- closes #이슈번호

## 체크리스트

- [ ] 테스트 코드 작성
- [ ] 로컬 테스트 완료
- [ ] API 문서 확인
```

## Merge 규칙

- 최소 1명 이상 리뷰 승인
- CI 통과 필수
- 리뷰 의견 반영 후 Merge

---

# 4. 코드 리뷰 규칙

## 리뷰 체크 항목

### 1. 정책 검증

- 요구사항이 정책대로 구현되었는가
- 예외 케이스가 고려되었는가

### 2. 비즈니스 플로우 검증

- 성공 흐름이 정상 동작하는가
- 실패 흐름이 처리되는가
- 상태 전이가 올바른가

### 3. 코드 품질 검증

- 단일 책임 원칙 준수
- 중복 코드 존재 여부
- 네이밍 적절성

### 4. 테스트 검증

- 정상 케이스 존재
- 실패 케이스 존재
- 경계값 검증 존재

---

# 5. 이슈 관리 규칙

## 이슈 라벨

| Label | 설명 |
|--------|--------|
| feat | 기능 개발 |
| fix | 버그 수정 |
| refactor | 리팩토링 |
| test | 테스트 |
| docs | 문서화 |
| infra | 인프라 |
| ai | AI 기능 |
| chore | 기타 작업 |

## 보드 상태

| 상태 | 설명 |
|--------|--------|
| Backlog | 작업 대기 |
| Todo | 이번 주 작업 |
| In Progress | 진행 중 |
| In Review | 코드 리뷰 |
| Done | 완료 |

---

# 6. 문서 관리 규칙

프로젝트 문서는 다음 구조로 관리한다.

```text
documents
├── api
├── policy
├── erd
├── meetings
├── troubleshooting
├── milestone
└── business-flow
```

## 문서 작성 원칙

- 정책 변경 시 Policy 문서 우선 수정
- API 변경 시 API 문서 즉시 반영
- 주요 장애 발생 시 Troubleshooting 작성
- 회의 내용은 Daily Meeting 문서로 기록
- 비즈니스 플로우 변경 시 Flow 문서 갱신

---

# 7. 개발 원칙

## 정합성 우선

- 데이터 정합성을 최우선으로 고려한다.
- 동시성 문제 발생 가능성을 검토한다.

## 정책 기반 개발

- 정책 문서 작성 후 구현한다.
- 정책과 구현이 불일치하지 않도록 관리한다.

## 테스트 우선 검증

- 핵심 비즈니스 로직은 테스트 코드 작성
- 정상 흐름과 실패 흐름 모두 검증

## 문서화

- 구현 완료 후 문서 갱신
- 변경된 정책은 즉시 반영