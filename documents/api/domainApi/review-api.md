## 9. 리뷰 (Review)

### POST /api/reviews — 리뷰 작성

**권한** BUYER | FINAL_DELIVERY 이후 + 1주문상품 1리뷰

**Request** `multipart/form-data`

| 필드 | 타입 | 필수 | 제약 |
| --- | --- | --- | --- |
| orderItemId | Long | ✅ | order_item_id |
| rating | int | ✅ | 1~5 |
| content | String | ✅ | 10~1000자 |
| images | File[] |  | 최대 5장 |

**Response** `200`

```json
{
  "success": true,
  "data": {
    "reviewId": 201,
    "productId": 42,
    "orderItemId": 5,
    "rating": 5,
    "content": "배송도 빠르고 제품 상태도 완벽합니다.",
    "images": ["https://cdn.example.com/reviews/201/1.jpg"],
    "createdAt": "2025-05-15T10:00:00",
    "updatedAt": "2025-05-15T10:00:00"
  }
}
```

**에러** `REVIEW_001` 배송 미완료 | `REVIEW_002` 중복 리뷰 | `REVIEW_008` 이미지 5장 초과 | 별점 1~5 외 또는 내용 10~1000자 위반 시 `400` (요청 검증)

---

### PATCH /api/reviews/{reviewId} — 리뷰 수정

**권한** BUYER (본인)

| 필드 | 타입 | 필수 | 제약 |
| --- | --- | --- | --- |
| rating | int | ✅ | 1~5 |
| content | String | ✅ | 10~1000자 |
| retainedImageUrls | String[] | ✅ | 유지할 기존 이미지 URL (빈 배열 허용, 최대 5개) |
|  newImages  | File[] |  | 새로 추가할 이미지 |
- retainedImageUrls + newImages 합계 최대 5개

**Request** `multipart/form-data`

```json
{ "rating": 4, "content": "수정된 리뷰 내용",retainedImageUrls: ["https://..."], newImages: [file1] }
```

**Response** `200`

```json
{
  "success": true,
  "data": {
    "reviewId": 201,
    "productId": 42,
    "orderItemId": 5,
    "rating": 4,
    "content": "수정된 리뷰 내용",
    "images": ["https://cdn.example.com/reviews/201/1.jpg"],
    "createdAt": "2025-05-15T10:00:00",
    "updatedAt": "2025-05-16T10:00:00"
  }
}
```

**에러** `REVIEW_005` 리뷰 없음/삭제됨 | `REVIEW_006` 권한 없음 | `REVIEW_007` 작성 후 30일 초과 | `REVIEW_008` 이미지 5장 초과

---

### DELETE /api/reviews/{reviewId} — 리뷰 삭제

**권한** BUYER (본인)

**Response** `200`

```json
{ "success": true, "data": null }
```

---

### GET /api/users/me/reviews — 내 리뷰 목록

**권한** BUYER

**Query** `page`, `size`

**Response** `200`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "reviewId": 201,
        "productId": 42,
        "orderItemId": 5,
        "rating": 5,
        "content": "배송도 빠르고...",
        "images": ["https://cdn.example.com/reviews/201/1.jpg"],
        "createdAt": "2025-05-15T10:00:00",
        "updatedAt": "2025-05-15T10:00:00"
      }
    ],
    "page": 0, "size": 20, "totalElements": 5, "totalPages": 1
  }
}
```

---

### GET /api/users/me/reviewable — 리뷰 작성 가능 목록

**권한** BUYER | FINAL_DELIVERY 완료 + 리뷰 미작성

**Response** `200`

```json
{
  "success": true,
  "data": [
    {
      "orderItemId": 5,
      "productId": 42,
      "productName": "맥북 프로 14인치 M3",
      "optionName": "스페이스 그레이 / 512GB",
      "deliveredAt": "2025-05-13T14:00:00"
    }
  ]
}
```

---
