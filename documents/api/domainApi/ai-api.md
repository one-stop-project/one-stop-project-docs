## 14. AI

### POST /api/ai/assistant — AI 쇼핑 어시스턴트

**권한** BUYER | Spring AI Tool Calling | Resilience4j Circuit Breaker

**Request**

```json
{ "message": "3만원 이하 반팔티 재고 있는 거 추천해줘" }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "message": "조건에 맞는 반팔티 3개를 찾았어요. 재고가 있는 상품은 'A반팔티(M·L)'입니다.",
    "products": [
      {
        "productId": 1,
        "name": "A반팔티",
        "price": 25000,
        "stock": 10,
        "thumbnailUrl": "https://cdn.example.com/products/1/thumb.jpg"
      }
    ]
  }
}
```

> Tool 호출: `searchProducts` → `checkStock` 순차 실행
Tool 실패 시: 실패 컨텍스트 LLM 전달 → 자연어 안내
LLM 장애 시: Circuit Breaker → Fallback 응답 반환
>

---

### POST /api/ai/reviews/{productId}/summarize — 리뷰 요약 트리거

**권한** ADMIN (또는 리뷰 누적 시 자동)

**Response** `200`

```json
{
  "success": true,
  "data": {
    "productId": 42,
    "status": "PROCESSING",
    "message": "리뷰 요약이 생성 중입니다"
  }
}
```

> 카테고리별 프롬프트 분기:
>
> - 의류: 착용감 / 사이즈 / 소재 중심
> - 전자제품: 성능 / 배터리 / 호환성 중심
> - 식품: 맛 / 식감 / 신선도 중심
>
> LLM 장애 시: 캐시된 이전 요약 반환 (Fallback)
