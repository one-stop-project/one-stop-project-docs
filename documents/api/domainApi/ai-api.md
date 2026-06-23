## 14. AI

> AI 기능은 핵심 거래 흐름과 분리된 보조 기능이다.
> 외부 AI API 장애는 Resilience4j Circuit Breaker로 격리하며, 실패 시 fallback 응답을 반환한다.

### POST /api/ai/assistant — AI 쇼핑 어시스턴트

**권한** 인증 필요 | Spring AI Tool Calling | Resilience4j Circuit Breaker

자연어 메시지를 받아 상품 검색·재고 확인 도구(Tool)를 호출하고 자연어 답변을 반환한다.

**Request**

```json
{
  "message": "3만원 이하 반팔티 재고 있는 거 추천해줘",
  "categoryId": 5
}
```

> `message`: 필수, 최대 500자
> `categoryId`: 선택(양수). 현재 보고 있는 카테고리 힌트이며, `null`이면 전체 검색

**Response** `200`

```json
{
  "success": true,
  "data": {
    "answer": "조건에 맞는 반팔티 3개를 찾았어요. 재고가 있는 상품은 'A반팔티(M·L)'입니다."
  }
}
```

> 응답은 자연어 답변(`answer`) 단일 필드이며, 상품 배열은 포함하지 않는다.
> Circuit Open / LLM 장애 시: `answer`에 fallback 안내 메시지를 담아 `200`으로 반환한다.
> (예: `"현재 AI 어시스턴트를 이용할 수 없습니다. 잠시 후 다시 시도해주세요."`)

---

### GET /api/products/{productId}/reviews/ai-summary — AI 리뷰 요약 조회

**권한** 공개

DB에 저장된 요약이 있으면 반환하고, 없으면 AI를 호출하지 않고 상태값만 반환한다.

**Response** `200`

```json
{
  "success": true,
  "data": {
    "reviewCount": 42,
    "averageRating": 4.3,
    "summary": {
      "pros": ["배송이 빠름", "품질 우수"],
      "cons": ["사이즈가 작게 나옴"],
      "keywords": ["배송", "품질", "사이즈"],
      "sentiment": "POSITIVE"
    },
    "status": "READY"
  }
}
```

> `status`:
> - `READY` — 요약 정상 제공 (`summary` 포함)
> - `PENDING` — 리뷰는 충분(5건 이상)하나 요약 생성 전 (`summary`는 `null`)
> - `INSUFFICIENT` — 리뷰 수 부족(5건 미만) (`summary`는 `null`)
>
> `sentiment`: `POSITIVE` / `NEUTRAL` / `NEGATIVE`

---

### POST /api/products/{productId}/reviews/ai-summary/refresh — AI 리뷰 요약 강제 갱신

**권한** ADMIN | SUPER_ADMIN

기존 요약을 덮어쓰는 동기 갱신이다. 최신순 최대 50건 리뷰로 재요약한다.

**Response** `200` (응답 구조는 요약 조회와 동일)

```json
{
  "success": true,
  "data": {
    "reviewCount": 42,
    "averageRating": 4.3,
    "summary": {
      "pros": ["배송이 빠름", "품질 우수"],
      "cons": ["사이즈가 작게 나옴"],
      "keywords": ["배송", "품질", "사이즈"],
      "sentiment": "POSITIVE"
    },
    "status": "READY"
  }
}
```

> 리뷰가 5건 미만이면 `status: "INSUFFICIENT"`로 반환한다.
> 카테고리별 프롬프트 분기(의류: 착용감/사이즈/소재, 전자제품: 성능/배터리/호환성, 식품: 맛/식감/신선도)를 적용한다.

**에러** `PRODUCT_001` 상품 없음 · `AI_001` AI 서비스 일시 이용 불가

---

> 리뷰 작성/수정/삭제 이벤트가 발생하면 별도 트리거 없이 비동기로 요약이 증분 갱신되거나 재생성된다.
