## 5. 장바구니 (Cart)

> **저장 방식**
비로그인: Redis Hash 임시 저장 (TTL 7일) `guest:cart:{guestCartId}` (itemId→quantity) + 담기 순서 ZSet `guest:cart-order:{guestCartId}`, guest_cart_id 쿠키로 식별
로그인: MySQL DB 저장 (cart, cart_item 테이블)
비로그인 → 로그인 시: Redis 데이터 DB로 병합 후 Redis 삭제
>
> 경로 변수 `{itemId}`는 cart_item ID가 아니라 상품 옵션 ID(product_item ID) 기준이다.
>

### GET /api/carts — 장바구니 조회

**권한** 전체 (비로그인 포함)

**Query** `page`, `size` (기본 size 20)

> 비로그인: Redis Hash 조회 (응답 `cartId`는 null)
로그인: DB 조회
>

**Response** `200`

```json
{
  "success": true,
  "data": {
    "cartId": 5,
    "content": [
      {
        "cartItemId": 12,
        "itemId": 101,
        "productId": 42,
        "productName": "맥북 프로 14인치 M3",
        "optionName": "스페이스 그레이 / 512GB",
        "price": 2190000,
        "quantity": 1,
        "thumbnailUrl": "https://cdn.example.com/products/42/thumb.jpg",
        "stock": 23,
        "available": true
      },
      {
        "cartItemId": 15,
        "itemId": 205,
        "productId": 15,
        "productName": "에어팟 프로 2세대",
        "optionName": "기본",
        "price": 359000,
        "quantity": 2,
        "thumbnailUrl": "https://cdn.example.com/products/15/thumb.jpg",
        "stock": 0,
        "available": false
      }
    ],
    "totalPrice": 2908000,
    "itemCount": 3,
    "page": 0,
    "size": 20,
    "totalElements": 2,
    "totalPages": 1
  }
}
```

> `available: false` → 재고 부족(품절) 또는 판매 중지
비로그인 응답은 `cartItemId: null`, `cartId: null`
>

---

### POST /api/carts/items — 장바구니 담기

**권한** 전체 (비로그인 포함)

**Request**

```json
{ "itemId": 101, "quantity": 1 }
```

> `itemId`는 상품 옵션 ID, `quantity`는 1~99
>

**Response** `200`

```json
{
  "success": true,
  "data": {
    "cartItemId": 12,
    "itemId": 101,
    "quantity": 1,
    "message": "장바구니에 추가되었습니다."
  }
}
```

> 동일 itemId 이미 있으면 수량 증가 (비로그인은 `cartItemId: null`)
>

**에러**

| 코드 | 설명 |
| --- | --- |
| CART_001 | 담을 수 없는(판매 중지) 상품 |
| CART_002 | 수량 범위(1~99) 초과 |
| CART_003 | 장바구니 최대 50종 초과 |
| CART_005 | 본인 상품 |
| INVENTORY_001 | 재고 초과 |
| PRODUCT_001 | 존재하지 않는 상품 옵션 |

---

### PATCH /api/carts/items/{itemId} — 수량 변경

**권한** 전체 (비로그인 포함)

> 경로 변수 `{itemId}`는 상품 옵션 ID 기준, `quantity`는 변경 후 최종 수량
>

**Request**

```json
{ "quantity": 3 }
```

**Response** `200`

```json
{
  "success": true,
  "data": { "itemId": 101, "quantity": 3 }
}
```

**에러**

| 코드 | 설명 |
| --- | --- |
| CART_004 | 장바구니에 해당 상품이 없음 |
| CART_001 | 담을 수 없는(판매 중지) 상품 |
| CART_002 | 수량 범위(1~99) 초과 |
| INVENTORY_001 | 재고 초과 |

---

### DELETE /api/carts/items/{itemId} — 장바구니 삭제

**권한** 전체 (비로그인 포함)

> 경로 변수 `{itemId}`는 상품 옵션 ID 기준
>

**Response** `200`

```json
{ "success": true, "data": null }
```

**에러** `CART_004` 장바구니에 해당 상품이 없음

---
