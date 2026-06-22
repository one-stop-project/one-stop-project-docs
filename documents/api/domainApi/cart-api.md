## 5. 장바구니 (Cart)

> **저장 방식**
비로그인: Redis Hash 임시 저장 (TTL 7일) `cart:guest:{sessionId}`
로그인: MySQL DB 저장 (cart, cart_item 테이블)
비로그인 → 로그인 시: Redis 데이터 DB로 이전 후 Redis 삭제
>

### GET /api/cart — 장바구니 조회

**권한** 전체 (비로그인 포함)

> 비로그인: Redis Hash 조회
로그인: DB 조회
>

**Response** `200`

```json
{
  "success": true,
  "data": {
    "cartId": 5,
    "items": [
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
    "itemCount": 3
  }
}
```

> `available: false` → 재고 부족 (품절)
>

---

### POST /api/cart/items — 장바구니 담기

**권한** 전체 (비로그인 포함)

**Request**

```json
{ "itemId": 101, "quantity": 1 }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "cartItemId": 12,
    "itemId": 101,
    "quantity": 1,
    "message": "장바구니에 추가되었습니다"
  }
}
```

> 동일 itemId 이미 있으면 수량 증가
>

**에러** `CART_001` 비활성 상품 | `CART_002` 수량 범위 초과 | `CART_005` 본인 상품

---

### PATCH /api/cart/items/{cartItemId} — 수량 변경

**권한** 전체 (비로그인 포함)

**Request**

```json
{ "quantity": 3 }
```

**Response** `200`

```json
{
  "success": true,
  "data": { "cartItemId": 12, "quantity": 3 }
}
```

---

### DELETE /api/cart/items/{cartItemId} — 장바구니 삭제

**권한** 전체 (비로그인 포함)

**Response** `200`

```json
{ "success": true, "data": null }
```

---
