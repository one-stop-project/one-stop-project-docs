# API

> Version: v1.1 | Base URL: `http://localhost:8080`
인증: `Authorization: Bearer {accessToken}` (Auth API 제외)
>

---

## 공통 응답 형식

### 성공

```json
{ "success": true, "data": { ... } }
```

### 페이징

```json
{
  "success": true,
  "data": {
    "content": [ ... ],
    "page": 0, "size": 20,
    "totalElements": 100, "totalPages": 5
  }
}
```

### 실패

```json
{
  "success": false,
  "status": 400,
  "code": "ORDER_002",
  "message": "재고가 부족합니다",
  "timestamp": "2025-05-12T14:30:00"
}
```

---

## API 개요

| 도메인 | 상세 API 문서 |
|---------|---------|
| 인증 (Auth) | [auth-api.md](./domainApi/auth-api.md) |
| 회원 (User) | [user-api.md](./domainApi/user-api.md) |
| 상품 (Product) | [product-api.md](./domainApi/product-api.md) |
| 카테고리 (Category) | [category-api.md](./domainApi/category-api.md) |
| 검색 (Search) | [search-api.md](./domainApi/search-api.md) |
| 장바구니 (Cart) | [cart-api.md](./domainApi/cart-api.md) |
| 주문 / 결제 (Order / Payment) | [order-payment-api.md](./domainApi/order-payment-api.md) |
| 배송 (Delivery) | [delivery-api.md](./domainApi/delivery-api.md) |
| 리뷰 (Review) | [review-api.md](./domainApi/review-api.md) |
| 포인트 (Point) | [point-api.md](./domainApi/point-api.md) |
| 쿠폰 (Coupon) | [coupon-api.md](./domainApi/coupon-api.md) |
| 구독 (Subscription) | [subscription-api.md](./domainApi/subscription-api.md) |
| 관리자 (Admin) | [admin-api.md](./domainApi/admin-api.md) |
| AI | [ai-api.md](./domainApi/ai-api.md) |

