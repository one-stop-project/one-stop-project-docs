## 3. 상품 (Product)

> 카테고리 조회/관리는 [category-api.md](./category-api.md), 검색·인기 검색어는 [search-api.md](./search-api.md), 상품 승인/반려는 [admin-api.md](./admin-api.md) 참고.

### GET /api/products — 상품 검색/목록 조회

**권한** 전체(비로그인 허용) | QueryDSL 동적 쿼리 | Redis 캐싱 TTL 5분 (POPULAR 정렬 제외)

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| keyword | String |  | 검색어 (FULLTEXT, 입력 시 인기 검색어 ZINCRBY 집계) |
| categoryId | Long |  | 카테고리 ID |
| minPrice | Long |  | 최소 가격 (0 이상) |
| maxPrice | Long |  | 최대 가격 (0 이상, minPrice 이상) |
| sort | String |  | LATEST(기본) / PRICE_ASC / PRICE_DESC / POPULAR |
| page | int |  | 기본 0 |
| size | int |  | 10~20, 기본 20 |

**Response** `200` — `Page<ProductSummaryResponse>`

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
        "salesCount": 342,
        "viewCount": 5821
      }
    ],
    "page": 0, "size": 20, "totalElements": 45, "totalPages": 3
  }
}
```

**에러** `PRODUCT_014` page size 허용 범위 위반 | `PRODUCT_015` 가격 범위 위반

---

### GET /api/products/popular — 인기 상품

**권한** 전체 | Redis ZSET (점수 = 조회수 70% + 판매수 30%)

**Query** `limit` 1~20, 기본 20

**Response** `200` — `List<PopularProductResponse>`

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

---

### GET /api/products/{productId} — 상품 단건 상세 (구매자)

**권한** 전체 | Redis 캐싱(productDetail) TTL 10분 | 조회수는 캐시와 무관하게 매 요청 집계

구매자용 응답이라 APPROVED 상품·ON_SALE 옵션만 노출하고, 옵션 재고 수량은 숨기고 품절 여부(`soldOut`)만 내려준다.

**Response** `200` — `BuyerProductDetailResponse`

```json
{
  "success": true,
  "data": {
    "productId": 42,
    "name": "맥북 프로 14인치 M3",
    "description": "Apple M3 칩 탑재",
    "thumbnailUrl": "https://cdn.example.com/products/42/thumb.jpg",
    "viewCount": 5821,
    "salesCount": 342,
    "shopName": "애플스토어",
    "optionNames": ["색상", "용량"],
    "items": [
      {
        "itemId": 101,
        "optionName": "스페이스 그레이 / 512GB",
        "price": 2190000,
        "soldOut": false
      }
    ],
    "imageUrls": ["https://cdn.example.com/products/42/1.jpg"],
    "categoryNames": ["노트북", "애플"],
    "tags": ["고성능", "프로용"]
  }
}
```

**에러** `PRODUCT_001` 상품 없음 | `PRODUCT_002` 판매하지 않는 상품

---

### GET /api/products/{productId}/related — 연관 상품

**권한** 전체 | 같은 카테고리 인기순(조회수 70% + 판매수 30%) 추천, 자기 자신·품절·비활성 제외

**Response** `200` — `List<ProductSummaryResponse>` (필드는 목록 조회 항목과 동일)

```json
{
  "success": true,
  "data": [
    {
      "productId": 43,
      "name": "맥북 에어 M3",
      "thumbnailUrl": "https://cdn.example.com/products/43/thumb.jpg",
      "minPrice": 1590000,
      "salesCount": 210,
      "viewCount": 3120
    }
  ]
}
```

> 상품 리뷰 목록·AI 요약은 product 도메인 API가 아니다. 판매자용 상품 리뷰 조회는 seller 도메인(`GET /api/seller/products/{productId}/reviews`), AI 리뷰 요약은 ai 도메인(`GET /api/products/{productId}/reviews/ai-summary`, [ai-api.md](./ai-api.md)) 참고.

---

## 4. 판매자 상품 관리 (Seller - Product / Item)

> 모든 엔드포인트 권한: SELLER (본인 상품·옵션). 상품 승인/반려/강제비활성은 관리자 API([admin-api.md](./admin-api.md)).

### GET /api/seller/products/tags/popular — 인기 태그 자동완성

**권한** SELLER

**Query** `keyword` (선택), `limit` 1~10, 기본 10

**Response** `200` — `List<PopularTagResponse>`

```json
{
  "success": true,
  "data": [
    { "tag": "고성능", "usageCount": 128 }
  ]
}
```

---

### POST /api/seller/products — 상품 등록

**권한** SELLER | `multipart/form-data`

- `data` 파트 (application/json) — `ProductCreateRequest`
- `images` 파트 — 이미지 파일 목록 (1~10장, 첫 번째가 썸네일)

```json
{
  "name": "맥북 프로 14인치 M3",
  "description": "Apple M3 칩 탑재",
  "categoryIds": [5, 10],
  "tags": ["고성능", "프로용"],
  "optionNames": ["색상", "용량"],
  "items": [
    { "optionValue1": "스페이스 그레이", "optionValue2": "512GB", "price": 2190000, "stock": 100 }
  ]
}
```

> 제약: 카테고리 1~3개 / 옵션 1~5개 / 태그 ≤10개·각 ≤30자 / 가격 ≥100원.

**Response** `201` — `ProductCreateResponse`

```json
{
  "success": true,
  "data": {
    "productId": 42,
    "name": "맥북 프로 14인치 M3",
    "status": "APPROVE_REQUESTED",
    "items": [
      { "itemId": 101, "optionName": "스페이스 그레이 / 512GB", "price": 2190000, "stock": 100, "status": "ON_SALE" }
    ]
  }
}
```

> 등록 즉시 `APPROVE_REQUESTED` 상태로 관리자 승인 대기.

**에러** `PRODUCT_005`·`PRODUCT_006` 이미지 1~10장 위반 | `PRODUCT_007` 카테고리 없음 | `PRODUCT_016` 옵션 조합 중복 | `SELLER_003` 미승인 판매자 (가격 100원·카테고리 1~3개·옵션 1~5개 등 필드 유효성 위반은 400 검증 에러로 응답)

---

### GET /api/seller/products — 내 상품 목록

**권한** SELLER

**Query** `page`, `size` (기본 10, id DESC 정렬)

**Response** `200` — `Page<SellerProductListResponse>`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "productId": 42,
        "name": "맥북 프로 14인치 M3",
        "status": "APPROVED",
        "thumbnailUrl": "https://cdn.example.com/products/42/thumb.jpg",
        "minPrice": 2190000,
        "salesCount": 342,
        "categoryNames": ["노트북", "애플"]
      }
    ],
    "page": 0, "size": 10, "totalElements": 15, "totalPages": 2
  }
}
```

---

### GET /api/seller/products/{productId} — 내 상품 단건 상세

**권한** SELLER (본인 상품) | 미승인 상품·STOP 옵션·옵션별 재고 포함

**Response** `200` — `ProductDetailResponse`

```json
{
  "success": true,
  "data": {
    "productId": 42,
    "name": "맥북 프로 14인치 M3",
    "description": "Apple M3 칩 탑재",
    "thumbnailUrl": "https://cdn.example.com/products/42/thumb.jpg",
    "status": "APPROVED",
    "viewCount": 5821,
    "salesCount": 342,
    "shopName": "애플스토어",
    "optionNames": ["색상", "용량"],
    "items": [
      { "itemId": 101, "optionName": "스페이스 그레이 / 512GB", "price": 2190000, "stock": 23, "status": "ON_SALE" }
    ],
    "imageUrls": ["https://cdn.example.com/products/42/1.jpg"],
    "images": [
      { "imageId": 7, "imageUrl": "https://cdn.example.com/products/42/1.jpg", "displayOrder": 1, "thumbnail": true }
    ],
    "categoryNames": ["노트북", "애플"],
    "tags": ["고성능", "프로용"],
    "rejectReason": null
  }
}
```

> `rejectReason`은 REJECTED 상품에서만 채워지고 그 외에는 null.

**에러** `PRODUCT_001` 상품 없음 | `PRODUCT_008` 타 판매자 상품

---

### PATCH /api/seller/products/{productId} — 상품 수정

**권한** SELLER (본인 상품)

**Request** `ProductUpdateRequest` (모든 필드 선택 — null이면 해당 필드 변경 안 함)

```json
{
  "name": "맥북 프로 14인치 M3 (2024)",
  "description": "Apple M3 칩 탑재 (개정)",
  "thumbnailUrl": "https://cdn.example.com/products/42/thumb2.jpg",
  "categoryIds": [5],
  "tags": ["고성능"]
}
```

**Response** `200` — `ProductDetailResponse` (단건 상세와 동일 구조)

> 이름·설명·이미지·카테고리가 실제 변경되면 `APPROVE_REQUESTED`로 전환되어 재승인 필요. 가격·재고·태그 변경은 즉시 반영(재승인 불필요).

**에러** `PRODUCT_008` 타 판매자 상품 | `PRODUCT_010` 현재 상태에서 수정 불가(DISCONTINUED)

---

### DELETE /api/seller/products/{productId} — 상품 삭제 (Soft Delete)

**권한** SELLER (본인 상품) | Soft Delete → DISCONTINUED

**Response** `200` — `ProductDeleteResponse`

```json
{ "success": true, "data": { "productId": 42, "status": "DISCONTINUED" } }
```

**에러** `PRODUCT_009` 주문 진행 중인 상품 삭제 불가

---

### POST /api/seller/products/{productId}/images — 상품 이미지 추가

**권한** SELLER (본인 상품) | `multipart/form-data`, `images` 파트

**Response** `201` — `ProductImageAddResponse`

```json
{
  "success": true,
  "data": {
    "addedImageCount": 2,
    "totalImageCount": 5,
    "thumbnailUrl": "https://cdn.example.com/products/42/1.jpg",
    "addedImages": [
      { "imageId": 12, "imageUrl": "https://cdn.example.com/products/42/4.jpg", "displayOrder": 4, "thumbnail": false }
    ]
  }
}
```

**에러** `PRODUCT_006` 이미지 최대 10장 초과

---

### DELETE /api/seller/products/{productId}/images/{imageId} — 상품 이미지 삭제

**권한** SELLER (본인 상품) | DB Soft Delete + S3 비동기 삭제

**Response** `200` — `ProductImageDeleteResponse`

```json
{
  "success": true,
  "data": { "deletedImageId": 12, "remainingImageCount": 4, "thumbnailUrl": "https://cdn.example.com/products/42/1.jpg" }
}
```

> 대표 이미지 삭제 시 다음 display_order 이미지가 자동으로 썸네일 승격.

**에러** `PRODUCT_005` 마지막 1장 삭제 불가(최소 1장) | `PRODUCT_011` 이미지 없음

---

### PATCH /api/seller/products/{productId}/images/{imageId}/thumbnail — 대표 이미지 변경

**권한** SELLER (본인 상품)

**Response** `200` — `ProductImageThumbnailResponse`

```json
{ "success": true, "data": { "thumbnailImageId": 7, "thumbnailUrl": "https://cdn.example.com/products/42/1.jpg" } }
```

**에러** `PRODUCT_011` 이미지 없음

---

### PATCH /api/seller/items/{itemId} — 옵션 수정

**권한** SELLER (본인 옵션) | 재고 변경 시 비관적 락

**Request** `ItemUpdateRequest` (모든 필드 선택 — 전달된 값만 수정. `stock`은 절대값 덮어쓰기)

```json
{ "price": 2290000, "stock": 80, "status": "ON_SALE" }
```

**Response** `200` — `ItemUpdateResponse`

```json
{ "success": true, "data": { "itemId": 101, "price": 2290000, "stock": 80, "status": "ON_SALE" } }
```

> 보정(PATCH) 재고 변경은 inventory_history에 ADJUSTMENT로 기록.

**에러** `PRODUCT_001` 옵션 없음 | `PRODUCT_008` 타 판매자 옵션 | `PRODUCT_010` 수정 불가 상태 | `SELLER_003` 미승인 판매자 | `INVENTORY_005` 보정 재고 최대 한도 초과

---

### POST /api/seller/items/{itemId}/inbound — 재고 입고

**권한** SELLER (본인 옵션) | 비관적 락

**Request** `InboundRequest`

```json
{ "quantity": 50, "reason": "5월 정기 입고" }
```

**Response** `200` — `InboundResponse`

```json
{
  "success": true,
  "data": { "itemId": 101, "previousStock": 80, "addedQuantity": 50, "currentStock": 130 }
}
```

> 입고는 inventory_history에 INBOUND로 기록.

**에러** `PRODUCT_001` 옵션 없음 | `PRODUCT_008` 타 판매자 옵션 | `PRODUCT_010` 수정 불가 상태 | `SELLER_003` 미승인 판매자 | `INVENTORY_003` 입고 후 재고 최대 한도 초과 (입고 수량 1개 미만은 400 검증 에러로 응답)

---
