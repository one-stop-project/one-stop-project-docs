## 15. 카테고리 (Category)

> 카테고리는 product 도메인에 속한다(코드: `domain/product`). 조회는 전체 공개, 생성/수정/삭제는 관리자 전용. 상품과의 관계는 [product-api.md](./product-api.md) 참고.

### GET /api/categories — 카테고리 트리 조회

**권한** 전체(비로그인 허용) | Caffeine 로컬 캐시 TTL 1시간

**Response** `200` — `List<CategoryTreeResponse>` (children 재귀)

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "전자기기",
      "children": [
        { "id": 5, "name": "노트북", "children": [] },
        { "id": 6, "name": "스마트폰", "children": [] }
      ]
    }
  ]
}
```

> 노드 필드는 `id`, `name`, `children`뿐이다(부모 ID는 트리 구조로 표현하며 별도 `parentId` 필드는 없다).

---

### POST /api/admin/categories — 카테고리 생성

**권한** ADMIN / SUPER_ADMIN

**Request** `CategoryCreateRequest`

```json
{ "name": "노트북", "parentId": 1 }
```

> `name` 필수·50자 이하. `parentId`가 null이면 루트 카테고리. 깊이 최대 3단계(대 > 중 > 소). 생성 시 카테고리 캐시 무효화.

**Response** `201` — `CategoryResponse`

```json
{ "success": true, "data": { "id": 5, "name": "노트북", "parentId": 1 } }
```

**에러** `CATEGORY_001` 부모 카테고리 없음 | `CATEGORY_002` 동일 부모 아래 중복 이름 | `CATEGORY_003` 깊이 3단계 초과

---

### PATCH /api/admin/categories/{categoryId} — 카테고리 이름 수정

**권한** ADMIN / SUPER_ADMIN

**Request** `CategoryUpdateRequest`

```json
{ "name": "노트북/태블릿" }
```

> `name` 필수·50자 이하. 수정 시 카테고리 캐시 무효화.

**Response** `200` — `CategoryResponse`

```json
{ "success": true, "data": { "id": 5, "name": "노트북/태블릿", "parentId": 1 } }
```

**에러** `CATEGORY_001` 카테고리 없음 | `CATEGORY_002` 동일 부모 아래 중복 이름

---

### DELETE /api/admin/categories/{categoryId} — 카테고리 삭제

**권한** ADMIN / SUPER_ADMIN | 하위 카테고리 일괄 삭제

**Response** `200`

```json
{ "success": true, "data": null }
```

> 상위 카테고리 삭제 시 하위 전체가 함께 삭제된다. 삭제 시 카테고리 캐시 무효화.

**에러** `CATEGORY_001` 카테고리 없음 | `CATEGORY_004` 상품이 매핑된 카테고리 삭제 불가
