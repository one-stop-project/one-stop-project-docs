## 16. 검색 (Search)

> 검색은 product 도메인에 속한다(코드: `domain/product`). 별도 `/api/search` 엔드포인트는 없고, 상품 목록 조회(`GET /api/products`)의 `keyword` 파라미터로 동작한다. 상품 목록 응답 스펙은 [product-api.md](./product-api.md) 참고.

### 검색 동작 개요

- **검색 방식**: MySQL FULLTEXT(`name`, `description`) — APPROVED 상품만 대상
- **검색어 길이**: 1자도 검색 자체는 허용하되, 1자 검색어는 인기 검색어 집계에서 제외(FULLTEXT는 2자 이상에서 정상 동작)
- **비로그인 검색 허용**
- **가격 필터**: 옵션 가격 중 하나라도 `minPrice`~`maxPrice` 범위에 포함되면 노출
- **정렬**: `LATEST` / `PRICE_ASC` / `PRICE_DESC` / `POPULAR`
  - `POPULAR` 정렬: 필터(keyword·categoryId·가격)가 **없으면** 전역 인기 랭킹을 그대로 반환하고, 필터가 **있으면** 필터링된 결과를 판매량 순으로 정렬한다 (#508)
- **캐싱**: 상품 목록 Redis TTL 5분(POPULAR 정렬 제외)

검색 호출은 상품 목록과 동일하다.

```
GET /api/products?keyword=맥북&sort=LATEST&page=0&size=20
```

---

### GET /api/products/popular-keywords — 인기 검색어 (공개)

**권한** 전체 | Redis ZSET 실시간 집계 (검색 시 ZINCRBY, 시간 가중치 10%/30%/60%, 매일 00:00 초기화)

**Query** `limit` 1~10, 기본 10

**Response** `200` — `List<PopularKeywordResponse>`

```json
{
  "success": true,
  "data": [
    { "rank": 1, "keyword": "맥북", "count": 342 },
    { "rank": 2, "keyword": "에어팟", "count": 215 }
  ]
}
```

> 노출은 TOP 10, ZSET에는 순위 안정성을 위해 상위 50위까지 보관한다.

**에러** `PRODUCT_014` limit 허용 범위(1~10) 위반

---

### GET /api/admin/search/popular — 관리자 인기 검색어 조회 (특정일)

**권한** ADMIN / SUPER_ADMIN

**Query**

| 파라미터 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| date | LocalDate (ISO) |  | 조회 기준일. 미지정 시 오늘(Asia/Seoul) |
| limit | int |  | 1~50, 기본 20 |

**Response** `200` — `PopularKeywordAdminResponse`

```json
{
  "success": true,
  "data": {
    "date": "2026-06-23",
    "keywords": [
      { "rank": 1, "keyword": "맥북", "count": 342 },
      { "rank": 2, "keyword": "에어팟", "count": 215 }
    ]
  }
}
```

**에러** `PRODUCT_014` limit 허용 범위(1~50) 위반
