## 3. 상품 (Product)

### GET /api/products — 상품 목록 조회

**권한** 전체 | QueryDSL 동적 쿼리 | Redis 캐싱 TTL 5분

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| keyword | String |  | 검색어 (Redis ZINCRBY 인기 검색어 반영) |
| categoryId | Long |  | 카테고리 ID |
| minPrice | Integer |  | 최소 가격 |
| maxPrice | Integer |  | 최대 가격 |
| sort | String |  | LATEST(기본) / PRICE_ASC / PRICE_DESC / POPULAR |
| page | int |  | 기본 0 |
| size | int |  | 기본 20 |

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "productId": 42,
        "name": "맥북 프로 14인치 M3",
        "thumbnailUrl": "https://cdn.example.com/products/42/thumb.jpg",
        "minPrice": 2190000,
        "maxPrice": 3590000,
        "sellerName": "애플스토어",
        "reviewCount": 127,
        "averageRating": 4.7,
        "categories" :
        [{"categoryId": 5, "name": "노트북"}, {"categoryId": 10, "name": "애플"}]
      }
    ],
    "page": 0, "size": 20, "totalElements": 45, "totalPages": 3
  }
}
```

---

### GET /api/products/{productId} — 상품 단건 조회

**권한** 전체 | Redis 캐싱 TTL 10분 | 조회수 Redis INCR

**Response** `200`

```json
{{
  "success": true,
  "data": {
    "productId": 42,
    "images" : 111
    "name": "맥북 프로 14인치 M3",
    // 추가: 상품 레벨에서 어떤 옵션 항목들을 사용하는지 명시
    "optionNames": {
      "option_name1": "색상",
      "option_name2": "용량",
      "option_name3": null,
      "option_name4": null,
      "option_name5": null
    },
    "items": [
      {
        "itemId": 101,
        // 추가: 단일 문자열이 아닌 분리된 값으로 응답
        "optionValues": {
          "option_value1": "스페이스 그레이",
          "option_value2": "512GB",
          "option_value3": null,
          "option_value4": null,
          "option_value5": null
        },
        "price": 2190000,
        "stock": 23,
        "status": "ON_SALE"
      }
    ]
  }
}
```

**에러** `PRODUCT_001` 상품 없음 | `PRODUCT_002` 비활성 상품

---

### GET /api/categories — 카테고리 트리

**권한** 전체 | Caffeine 로컬 캐시 TTL 1시간

**Response** `200`

```json
{
  "success": true,
  "data": [
    {
      "categoryId": 1,
      "name": "전자기기",
      "parentId": null,
      "children": [
        { "categoryId": 5, "name": "노트북", "parentId": 1, "children": [] },
        { "categoryId": 6, "name": "스마트폰", "parentId": 1, "children": [] }
      ]
    }
  ]
}
```

---

### GET /api/products/popular-keywords — 인기 검색어

**권한** 전체 | Redis ZSET TOP 10

**Response** `200`

```json
{
  "success": true,
  "data": [
    { "rank": 1, "keyword": "맥북", "count": 342 },
    { "rank": 2, "keyword": "에어팟", "count": 215 }
  ]
}
```

---

### GET /api/products/popular — 인기 상품

**권한** 전체 | Redis ZSET TTL 5분

**Query** `limit` 기본 20

```json
{
  "success": true,
  "data": [
    {
      "rank": 1,
      "productId": 42,
      "name": "맥북 프로 14인치 M3",
      "thumbnailUrl": "https://cdn.example.com/products/42/thumb.jpg",
      "minPrice": 2190000,
      "salesCount": 342
    }
  ]
}
```

**Response** `200`

---

### GET /api/products/{productId}/related — 연관 상품

**권한** 전체 | 같은 카테고리 인기순 10개

**Response** `200`

```json
{
  "success": true,
  "data": [
    {
      "productId": 43,
      "name": "맥북 에어 M3",
      "thumbnailUrl": "https://cdn.example.com/products/43/thumb.jpg",
      "minPrice": 1590000,
      "averageRating": 4.6
    }
  ]
}
```

---

### GET /api/products/{productId}/reviews — 상품 리뷰 목록

**권한** 전체

**Query** `sort` (LATEST / RATING_DESC / RATING_ASC), `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "averageRating": 4.7,
    "reviewCount": 127,
    "ratingDistribution": { "5": 89, "4": 25, "3": 8, "2": 3, "1": 2 },
    "content": [
      {
        "reviewId": 201,
        "userName": "홍*동",
        "rating": 5,
        "content": "배송도 빠르고 제품 상태도 완벽합니다.",
        "images": ["https://cdn.example.com/reviews/r1-1.jpg"],
        "optionName": "스페이스 그레이 / 512GB",
        "createdAt": "2025-05-11T10:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 127, "totalPages": 7
  }
}
```

---

### GET /api/products/{productId}/reviews/summary — AI 리뷰 요약

**권한** 전체 | Redis 캐시 우선, 없으면 DB

**Response** `200`

```json
{
  "success": true,
  "data": {
    "productId": 42,
    "average_rate" : 5,
    "summary": "전반적으로 만족도가 높은 프리미엄 노트북입니다.",
    "pros": ["뛰어난 성능", "배터리 지속력"],
    "cons": ["고온 발열", "높은 가격"],
    "keywords": ["고성능", "빠른배송", "프로용"],
    "sentiment": "POSITIVE",
    "updatedAt": "2025-05-12T03:00:00"
  }
}
```

> 리뷰 없으면 null 반환
AI 장애 시 캐시된 이전 요약 반환 (Fallback)
>

---

## 4. 판매자 상품 관리 (Seller - Product)

### POST /api/seller/products — 상품 등록

**권한** SELLER (APPROVED)

**Request** `multipart/form-data`

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| name | String | ✅ | 상품명 |
| description | String | ✅ | 상품 설명 |
| categoryIds | Long | ✅ | 카테고리 IDs |
| images | File | ✅ | 대표 이미지 (S3 Pre-signed URL) |
| items | JSON | ✅ | 옵션 목록 |

**Response** `201`

```json
{
  "success": true,
  "data": {
    "productId": 42,
    "name": "맥북 프로 14인치 M3",
    "status": "APPROVE_REQUESTED",
    "items": [
      { "itemId": 101, "optionName": "스페이스 그레이 / 512GB", "price": 2190000, "stock": 100 }
    ]
  }
}
```

> 등록 즉시 APPROVE_REQUESTED 상태. 관리자 승인 대기.
>

---

### POST /api/seller/products/{productId}/request-approval — 승인 요청

**권한** SELLER (본인 상품)

상태 전이: `TEMP_SAVED → APPROVE_REQUESTED`

**Response** `200`

```json
{
  "success": true,
  "data": { "productId": 42, "status": "APPROVE_REQUESTED" }
}
```

**에러** `PRODUCT_003` TEMP_SAVED 상태 아님

---

### GET /api/seller/products — 내 상품 목록

**권한** SELLER

**Query** `status`, `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "productId": 42,
        "name": "맥북 프로 14인치 M3",
        "status": "APPROVED",
        "itemCount": 2,
        "totalStock": 150,
        "salesCount": 342,
        "createdAt": "2025-04-15T09:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 15, "totalPages": 1
  }
}
```

---

### PATCH /api/seller/products/{productId} — 상품 수정

**권한** SELLER (본인 상품)

**Request**

```json
{
  "name": "맥북 프로 14인치 M3 (2024)",
  "description": "Apple M3 칩 탑재 (개정)"
}
```

**Response** `200`

```json
{
  "success": true,
  "data": { "productId": 42, "name": "맥북 프로 14인치 M3 (2024)" }
}
```

---

### DELETE /api/seller/products/{productId} — 상품 삭제

**권한** SELLER (본인 상품) | Soft Delete → DISCONTINUED

**Response** `200`

```json
{
  "success": true,
  "data": { "productId": 42, "status": "DISCONTINUED" }
}
```

**에러** `PRODUCT_009` 미완료 주문 존재

---

### PATCH /api/seller/items/{itemId} — 옵션 수정

**권한** SELLER | 재고 변경 시 비관적 락

**Request**

```json
{ "price": 2290000, "stock": 80, "status": "ON_SALE" }
```

**Response** `200`

```json
{
  "success": true,
  "data": { "itemId": 101, "price": 2290000, "stock": 80, "status": "ON_SALE" }
}
```

---

### POST /api/seller/items/{itemId}/inbound — 재고 입고

**권한** SELLER | 비관적 락

**Request**

```json
{ "quantity": 50, "description": "5월 정기 입고" }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "itemId": 101,
    "previousStock": 80,
    "addedQuantity": 50,
    "currentStock": 130
  }
}
```

---
