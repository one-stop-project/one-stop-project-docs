# ERD

## 1. 회원 (User & Seller)

### `users` (구매자/기본 회원)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `user_id` | bigint | PK, increment | 회원 고유 ID |
| `email` | varchar(100) | unique, not null | 이메일 (로그인 ID) |
| `password` | varchar(255) | not null | 비밀번호 (암호화) |
| `name` | varchar(50) | not null | 이름 |
| `phone` | varchar(20) |  | 전화번호 |
| `address` | varchar(255) |  | 기본 배송지 주소 |
| `detail_address` | varchar(255) |  | 상세 주소 |
| `role` | varchar(20) | not null | `BUYER` / `SELLER` / `ADMIN` / `SUPER_ADMIN` |
| `status` | varchar(20) | not null, default: `ACTIVE` | `ACTIVE` / `SUSPENDED` / `WITHDRAWN` |
| `provider` | varchar(20) |  | OAuth2 Provider (kakao/google/naver), 일반 가입은 null |
| `provider_id` | varchar(255) |  | OAuth2 Provider 측 사용자 ID |
| `last_login_at` | datetime |  | 마지막 로그인 시각 |
| `token_version` | int | not null, default: 0 | 토큰 무효화 기준 버전 |
| `created_at` | datetime | not null | 가입 일시 |
| `updated_at` | datetime | not null | 수정 일시 |

> 💡 **Indexes**
>
> - `idx_user_email`: UNIQUE (email)
> - `idx_user_provider`: UNIQUE (provider, provider_id)
> - `idx_user_status`: (status)

### `seller` (판매자 정보)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `seller_id` | bigint | PK, increment | 판매자 고유 ID |
| `user_id` | bigint | unique, not null | 연결된 user_id (1:1) |
| `shop_name` | varchar(100) | not null | 상호명 |
| `business_number` | varchar(20) | not null | 사업자 등록번호 |
| `bank_name` | varchar(50) |  | 정산용 은행명 |
| `bank_account` | varchar(50) |  | 정산용 계좌번호 |
| `status` | varchar(20) | not null, default: `PENDING` | `PENDING` / `APPROVED` / `REJECTED` / `SUSPENDED` |

---

## 2. 상품 (Product & Category)

### `category` (카테고리)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `category_id` | bigint | PK, increment | 카테고리 ID |
| `name` | varchar(50) | not null | 카테고리명 |
| `parent_id` | bigint |  | 상위 카테고리 ID (자기 참조) |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `uk_category_parent_name`: UNIQUE (parent_id, name) — 같은 부모 아래 카테고리명 중복 방지

### `product` (상품 기본 정보)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `product_id` | bigint | PK, increment | 상품 ID |
| `seller_id` | bigint | not null | 등록 판매자 ID |
| `name` | varchar(200) | not null | 상품명(길이 제한) |
| `description` | text |  | 상품 설명 |
| `thumbnail_url` | varchar(500) |  | 대표 이미지 URL |
| `option_name_1` | varchar(100) | null | 옵션명 1 |
| `option_name_2` | varchar(100) | null | 옵션명 2 |
| `option_name_3` | varchar(100) | null | 옵션명 3 |
| `option_name_4` | varchar(100) | null | 옵션명 4 |
| `option_name_5` | varchar(100) | null | 옵션명 5 |
| `status` | varchar(30) | not null, default: `APPROVE_REQUESTED` | 상품 상태 (APPROVE_REQUESTED / APPROVED / REJECTED / DISCONTINUED / FORCE_INACTIVE) |
| `view_count` | bigint | not null, default: 0 | 조회수 (Redis → MySQL 5분마다 동기화) |
| `sales_count` | bigint | not null, default: 0 | 판매 수량 (인기 정렬용) |
| `version` | int | not null, default: 0 | 낙관적 락 |
| `created_at` | datetime | not null | 등록 일시 |
| `updated_at` | datetime | not null | 수정 일시 |
| `deleted_at` | datetime | null | Soft Delete 시각 (`DISCONTINUED` 시 기록) |

> 💡 **Indexes**
>
> - `idx_product_seller`: (seller_id, status)
> - `idx_product_status`: (status, created_at)
> - `idx_product_sales`: (status, sales_count)
> - `idx_product_name_fulltext`: FULLTEXT(name, description) — DB 마이그레이션으로 별도 관리

### `product_item` (상품 옵션 및 재고)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `item_id` | bigint | PK, increment | 옵션별 고유 ID |
| `product_id` | bigint | not null | 소속 상품 ID |
| `option_value_1` | varchar(100) | not null | 옵션값 1 |
| `option_value_2` | varchar(100) | not null | 옵션값 2 |
| `option_value_3` | varchar(100) | not null | 옵션값 3 |
| `option_value_4` | varchar(100) | not null | 옵션값 4 |
| `option_value_5` | varchar(100) | not null | 옵션값 5 |
| `price` | bigint | not null | 옵션 가격 |
| `stock` | bigint | not null, default: 0 | 잔여 재고 (비관적 락 대상) |
| `status` | varchar(20) | not null, default: `ON_SALE` | `ON_SALE` / `STOP` |
| `created_at` | datetime | not null |  |
| `updated_at` | datetime | not null |  |

> 💡 **Indexes**
>
> - `idx_item_product`: (product_id, status)
> - `idx_item_status_price`: (status, price)
> - `uk_product_item_options`: UNIQUE (product_id, option_value_1, option_value_2, option_value_3, option_value_4, option_value_5) — 동일 상품 내 옵션 조합 중복 방지

### `inventory_history` (재고 변동 이력)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `history_id` | bigint | PK, increment | 이력 ID |
| `item_id` | bigint | not null | 대상 옵션 ID (FK → product_item) |
| `type` | varchar(20) | not null | `INBOUND` / `ADJUSTMENT` / `OUTBOUND` / `RESTORE` |
| `quantity` | bigint | not null | 변동 수량 (양수) |
| `before_stock` | bigint | not null | 변동 전 재고 |
| `after_stock` | bigint | not null | 변동 후 재고 |
| `reason` | varchar(255) | null | 변동 사유 |
| `ref_type` | varchar(20) | null | 참조 출처 (`ORDER` / `RETURN` / `MANUAL`) |
| `ref_id` | bigint | null | 참조 ID (예: order_id) |
| `created_by` | bigint | not null | 처리자 user_id |
| `created_at` | datetime | not null | 변동 시각 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **타입별 사용**
>
> - `INBOUND`: 재고 입고 (ref_type/ref_id 없음)
> - `ADJUSTMENT`: 수동 보정 (PATCH /items로 stock 변경 시)
> - `OUTBOUND`: 향후 주문 차감 이력 (이번 이슈 스코프 외)

> 💡 **Indexes**
>
> - `idx_ih_item`: (item_id, created_at)
> - `idx_ih_type`: (type, created_at)

### `product_image` (상품 이미지)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `image_id` | bigint | PK, increment | 이미지 고유 ID |
| `product_id` | bigint | not null | 소속 상품 ID |
| `image_url` | varchar(500) | not null | 이미지 S3 URL |
| `display_order` | int | not null | 이미지 노출 순서 (1이 썸네일) |
| `status` | varchar(20) | not null, default : ACTIVE | ACTIVE / DELETED (부분 삭제용) |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `idx_pi_product_status_order`: (product_id, status, display_order)

### `product_category_mapping` (상품-카테고리 N:N 매핑 테이블)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `mapping_id` | bigint | PK, increment | 매핑 고유 ID |
| `product_id` | bigint | not null | 상품 ID |
| `category_id` | bigint | not null | 카테고리 ID |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `idx_pcm_product`: (product_id)
> - `idx_pcm_category`: (category_id, product_id)
> - `uk_pcm`: UNIQUE (product_id, category_id) — 중복 매핑 방지

### `product_tag` (상품 태그 — `@ElementCollection`)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `product_id` | bigint | not null | 소속 상품 ID |
| `tag` | varchar(30) | not null | 태그 (trim + 소문자 정규화 저장) |

> 💡 **Indexes**
>
> - `idx_product_tag_product_tag`: (product_id, tag)

### `search_history` (인기 검색어 관리를 위한 로그 테이블)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `search_id` | bigint | PK, increment | 검색 로그 ID |
| `event_id` | varchar(36) | unique | 재처리 중복 INSERT 방지 멱등키 (레거시 행은 null 가능) |
| `keyword` | varchar(100) | not null | 검색어 |
| `user_id` | bigint | null | 검색한 유저 ID (비회원은 null) |
| `searched_at` | datetime | not null | 검색 일시 |

> 💡 **Indexes**
>
> - `idx_sh_searched_at`: (searched_at)
> - `idx_sh_keyword`: (keyword, searched_at)
> - `uk_sh_event_id`: UNIQUE (event_id)

### `related_product` (연관 상품 매핑 테이블)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| relation_id | bigint | PK, increment | 연관 고유 ID |
| base_product_id | bigint | not null | 기준 상품 ID |
| target_product_id | bigint | not null | 연관 추천되는 상품 ID |
| relation_weight | double | not null, default : 0 | 연관성 가중치 (AI가 분석한 연관도 점수 등) |
| created_at | datetime | not null | 생성일 |
| updated_at | datetime | not null | 수정일 |

---

## 3. 장바구니 (Cart)

### `carts` (장바구니 헤더)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `cart_id` | bigint | PK, increment | 장바구니 ID |
| `user_id` | bigint | unique, not null | 소유자 ID (1:1) |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

### `cart_items` (장바구니 상품 상세)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `cart_item_id` | bigint | PK, increment |  |
| `cart_id` | bigint | not null | 소속 장바구니 ID |
| `item_id` | bigint | not null | 담은 상품 옵션 ID |
| `quantity` | int | not null | 수량 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `uk_cart_item` (Unique): (cart_id, item_id) - 동일 장바구니 내 중복 옵션 방지

---

## 4. 주문 (Order)

### `orders` (주문 헤더)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `order_id` | bigint | PK, increment | 주문 번호 |
| `user_id` | bigint | not null | 구매자 ID |
| `subscription_id` | bigint |  | 구독상태 연계하여 배송비 적용 |
| `total_price` | bigint | not null | 총 주문 금액 |
| `subscription_discount` | bigint | default: 0 | 구독 할인 금액 |
| `discount_price` | bigint | default: 0 | 총 할인 금액 |
| `final_price` | bigint | not null | 최종 결제 금액 |
| `user_coupon_id` | bigint |  | 적용한 사용자 쿠폰 ID |
| `used_point` | int | default: 0 | 사용한 포인트 |
| `status` | varchar(20) | not null | `PENDING_PAYMENT` / `PAID` / `CANCELLED` |
| `receiver_name` | varchar(50) | not null | 수령인 이름 |
| `receiver_phone` | varchar(20) | not null | 수령인 연락처 |
| `receiver_address` | varchar(255) | not null | 배송 주소 |
| `delivery_fee` | bigint | not null | 배송비 |
| `delivery_message`  | varchar(50) |  | 배송시 요청사항 |
| `order_type` | varchar(20) | not null, default: cart | 주문 유입 경로(`DIRECT` / `CART` ) |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `idx_orders_user`: (user_id, created_at)
> - `idx_orders_status`: (status, created_at)
> - `idx_orders_user_coupon`: (user_coupon_id)

### `order_items` (주문 상품 상세)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `order_item_id` | bigint | PK, increment |  |
| `order_id` | bigint | not null | 소속 주문 ID |
| `item_id` | bigint | not null | 구매한 상품 옵션 ID |
| `seller_id` | bigint | not null | 해당 상품의 판매자 ID |
| `item_name` | varchar(200) | not null | 주문 시점의 상품명 (스냅샷) |
| `quantity` | int | not null | 구매 수량 |
| `price` | bigint | not null | 주문 시점의 가격 (스냅샷) |
| `status` | varchar(20) | not null, default: `ORDERED`  | `PENDING_PAYMENT` / `ORDERED` / `CONFIRMED` / `SHIPPING` / `DELIVERED`  / `CANCELLED` / `REJECTED` |
| `thumbnail_url` | varchar(500) |  | 대표 이미지 URL(스냅샷) |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `idx_oi_order`: (order_id)
> - `idx_oi_seller`: (seller_id, status)
> - `idx_oi_status`: (status, created_at)

### `order_cancel_history` (주문 취소 이력)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `cancel_history_id` | bigint | PK, increment | 취소/거절 이력 ID |
| `order_id` | bigint | not null | 취소/거절 대상 주문 ID |
| `order_item_id` | bigint |  | 주문 상품 단위 거절/취소인 경우 사용 |
| `actor_type` | varchar(30) | not null | 처리 주체 유형: `BUYER` / `SELLER` / `ADMIN` / `SUPER_ADMIN` / `SYSTEM`  |
| `actor_id` | bigint |  | 실제 처리자 ID, SYSTEM 처리 시 null 가능 |
| `cancel_type`  | varchar(30) | not null | 취소/거절 유형: `BUYER_CANCEL` / `SELLER_REJECT` / `ADMIN_CANCEL` / `PAYMENT_FAILED` / `PAYMENT_CANCELLED` / `SYSTEM_CANCEL` |
| `reason` | varchar(255) |  | 취소/거절 사유 |
| `cancelled_price` | bigint | not null | 이 이력 건에서 취소/거절 처리된 금액 |
| `restored_point` | int | not null | 이 이력 건으로 복구된 포인트 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

---

## 5. 배송 (Delivery)

### `delivery` (운송장 정보)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `delivery_id` | bigint | PK, increment |  |
| `order_item_id` | bigint | unique, not null | 연결된 주문 상품 ID (1:1) |
| `status` | varchar(30) | not null, default: `ACCEPT` | ACCEPT, INSTRUCT, DEPARTURE, DELIVERING, FINAL_DELIVERY, ORDER_CANCELLED |
| `invoice_number` | varchar(50) |  | 운송장 번호 |
| `delivery_company` | varchar(50) |  | 택배사 명 |
| `created_at` | datetime | not null |  |
| `updated_at` | datetime | not null |  |

### `delivery_history` (배송 상태 이력)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `history_id` | bigint | PK, increment |  |
| `delivery_id` | bigint | not null | 배송 ID |
| `status` | varchar(30) | not null | 변경된 상태 |
| `changed_at` | datetime | not null | 변경 일시 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

---

## 6. 리뷰 (Review)

### `review` (리뷰 원본)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `review_id` | bigint | PK, increment |  |
| `user_id` | bigint | not null | 작성자 ID |
| `order_item_id` | bigint | unique, not null | 1주문 1리뷰 원칙 (1:1) |
| `product_id` | bigint | not null | 상품 ID |
| `rating` | int | not null | 별점 (1~5) |
| `content` | text |  | 리뷰 내용 |
| `created_at` | datetime | not null |  |
| `updated_at` | datetime | not null |  |
| `status` | varchar(20) | not null | `ACTIVE` /`DELETED` 부분 삭제 및 softDelete 처리용 |

> 💡 **Indexes**
>
> - `idx_review_product`: (product_id, created_at)

### `review_image` (리뷰 첨부 이미지)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `image_id` | bigint | PK, increment |  |
| `review_id` | bigint | not null | 리뷰 ID |
| `image_url` | varchar(500) | not null | 이미지 S3 URL |
| `display_order` | int |  | 여러 이미지 중 순서 노출 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

### `product_review_summary` (AI 리뷰 요약 정보)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `id` | bigint | PK, increment | 요약 ID |
| `product_id` | bigint | unique, not null | 대상 상품 ID (1:1, FK `fk_review_summary_product`) |
| `pros` | text | not null | 장점 키워드 목록 |
| `cons` | text | not null | 단점 키워드 목록 |
| `keywords` | text | not null | 핵심 키워드 목록 |
| `sentiment` | varchar(255) | not null | `POSITIVE` / `NEUTRAL` / `NEGATIVE` |
| `last_included_review_id` | bigint |  | 마지막으로 요약에 포함된 리뷰 ID (증분 업데이트 경계점) |
| `review_count` | bigint | not null | 요약 기준 리뷰 수 캐시 |
| `average_rating` | double | not null | 상품 평균 별점 |
| `version` | bigint |  | 낙관적 락 버전 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 요약 갱신 일시 |

---

## 7. 포인트 (Point)

### `points` (포인트 잔액)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `point_id` | bigint | PK, increment | 포인트 계정 ID |
| `user_id` | bigint | unique, not null | 소유자 ID, BUYER 기준 1:1 |
| `balance` | int | not null, default: 0 | 현재 사용 가능 포인트 총액 |
| `version` | int | not null, default: 0 | 낙관적 락 버전 |
| `integrity_hash` | varchar(64) | not null | 무결성 해시 (HMAC-SHA256) |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `uk_point_user`: UNIQUE (user_id)

### `point_history` (포인트 변동 이력)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `history_id` | bigint | PK, increment | 포인트 이력 ID |
| `point_id` | bigint | not null | 포인트 계정 ID |
| `user_id` | bigint | not null | 사용자 ID, 조회 편의를 위한 중복 저장 |
| `order_id` | bigint |  | 관련 주문 ID, 충전/만료 등 주문과 무관하면 null |
| `amount` | int | not null | 변동 금액, 적립/충전/복구는 양수, 사용/만료는 음수 |
| `remaining_amount` | int | not null, default: 0 | 차감 가능 잔여 금액, CHARGE/EARN/REFUND 이력에서 사용 |
| `type` | varchar(20) | not null | `CHARGE` / `EARN` / `USE` / `EXPIRE` / `REFUND`  |
| `description` | varchar(255) |  | 이력 설명 |
| `expire_at` | date |  | 포인트 만료일, CHARGE/EARN/REFUND 이력에서 사용 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `idx_ph_point`: (point_id, created_at)
> - `idx_ph_user`: (user_id, created_at)
> - `idx_ph_order`: (order_id)
> - `idx_ph_expire`: (type, expire_at, remaining_amount)
> - `idx_ph_deduct`: (point_id, expire_at, created_at, remaining_amount)

### `point_usage_detail` (포인트 사용 상세 이력)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `usage_detail_id` | bigint | PK, increment | 포인트 사용 상세 ID |
| `use_history_id` | bigint | not null | USE 타입 PointHistory ID |
| `source_history_id` | bigint | not null | 실제 차감된 원본 PointHistory ID, CHARGE/EARN/REFUND |
| `used_amount` | int | not null | 해당 원본 이력에서 차감된 금액 |
| `source_expire_at` | date | not null | 원본 포인트 만료일 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `idx_pud_use_history`: (use_history_id)
> - `idx_pud_source_history`: (source_history_id)

---

## 8. 구독 (Subscription)

### `subscription` (구독 상태 관리)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `subscription_id` | bigint | PK, increment |  |
| `user_id` | bigint | not null | 구독자 ID |
| `start_at` | datetime | not null | 구독 시작 일시 |
| `end_at` | datetime | not null | 구독 만료 예정 일시 |
| `status` | varchar(20) | not null, default: `ACTIVE` | `ACTIVE` / `EXPIRED` / `CANCELLED` |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |
| `next_payment_date` | datetime | not null | 다음 자동 결제 예정일시 |
| `cancel_reason` | varchar(255) | null | 구독 해지 사유 |

> 💡 **Indexes**
>
> - `idx_sub_user`: (user_id, status, created_at)

---

## 9. 쿠폰 (Coupon)

### `coupon` (쿠폰 마스터 정보)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `coupon_id` | bigint | PK, increment | 쿠폰 ID |
| `name` | varchar(100) | not null | 쿠폰명 |
| `discount_type` | varchar(20) | not null | `RATE` (정률) / `FIXED` (정액) |
| `discount_value` | int | not null | 할인율(%) 또는 할인 금액 |
| `min_order_price` | bigint | not null, default: 0 | 최소 주문 금액 조건 |
| `max_discount_price` | bigint |  | 정률 쿠폰 최대 할인 한도, RATE 타입 필수 / FIXED 타입 미사용 |
| `total_quantity` | int | not null | 발급 가능 총 수량 |
| `issued_quantity` | int | not null, default: 0 | DB 기준 실제 발급 완료 수량 |
| `status` | varchar(20) | not null, default: `ACTIVE` | `ACTIVE` / `INACTIVE` |
| `start_at` | datetime | not null | 발급/사용 시작일 |
| `expired_at` | datetime | not null | 발급/사용 만료일 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `idx_coupon_status_period`: (status, start_at, expired_at)
> - `idx_coupon_created_at`: (created_at)

### `user_coupon` (사용자 발급 쿠폰)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `user_coupon_id` | bigint | PK, increment | 사용자 쿠폰 ID |
| `user_id` | bigint | not null | 소유자 ID |
| `coupon_id` | bigint | not null | 발급받은 쿠폰 마스터 ID |
| `used_order_id` | bigint |  | 쿠폰이 사용된 주문 ID |
| `status` | varchar(20) | not null, default: `AVAILABLE` | `AVAILABLE` / `USED` / `EXPIRED` |
| `used_at` | datetime |  | 실제 사용 일시 |
| `created_at` | datetime | not null | 생성일/발급일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `uk_user_coupon_user_coupon`: UNIQUE (user_id, coupon_id)
> - `uk_uc_used_order`: UNIQUE (used_order_id)
> - `idx_uc_user`: (user_id, status)
> - `idx_uc_coupon`: (coupon_id)

---

## 10. 아웃박스 이벤트 (Outbox Event)

### `outbox_event` (이벤트 발행 보장용 Outbox 테이블)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `outbox_id` | bigint | PK, increment | Outbox 이벤트 ID |
| `event_id` | varchar(100) | unique, not null | 이벤트 고유 ID, 중복 발행 및 Consumer 멱등 처리용 |
| `event_type` | varchar(50) | not null | 이벤트 타입. 예: `PAYMENT_APPROVED` |
| `aggregate_type` | varchar(50) | not null | 이벤트 기준 도메인. 예: `ORDER` |
| `aggregate_id` | bigint | not null | 기준 도메인 ID. 예: `order_id` |
| `topic` | varchar(100) | not null | Kafka 전송 대상 토픽 |
| `partition_key` | varchar(100) | not null | Kafka 메시지 Key. 동일 aggregate의 순서 보장용 |
| `payload` | text | not null | Kafka로 전송할 JSON 데이터 |
| `status` | varchar(20) | not null, default: `PENDING` | `PENDING` / `PROCESSING` / `PUBLISHED` / `DEAD` |
| `retry_count` | int | not null, default: 0 | 전송 실패 시 재시도 횟수 |
| `last_error_message` | varchar(500) |  | 마지막 발행 실패 사유 |
| `processed_at` | datetime |  | 발행 성공 또는 최종 실패 처리 일시 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `uk_outbox_event_event_id`: UNIQUE (`event_id`) - 이벤트 중복 식별용
> - `idx_outbox_status_created`: (`status`, `created_at`) - PENDING 이벤트 폴링 조회용
> - `idx_outbox_cleanup`: (`status`, `processed_at`) - PUBLISHED / DEAD 이벤트 정리 배치용
> - `idx_outbox_aggregate`: (`aggregate_type`, `aggregate_id`) - 특정 주문/결제 기준 이벤트 추적용
> - `idx_outbox_event_type`: (`event_type`, `created_at`) - 이벤트 타입별 조회용

---

## 11. 결제 (Payment)

### `payments` (결제 정보)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `payment_id` | bigint | PK, increment | 결제 ID |
| `order_id` | bigint | unique, not null | 주문 ID (결제 1:1 주문) |
| `payment_key` | varchar(100) | not null | Mock 결제 키 |
| `amount` | bigint | not null | 결제 금액 |
| `status` | varchar(20) | not null, default: `READY` | `READY` / `PAID` / `FAILED` / `CANCELLED` |
| `method` | varchar(20) | not null, default: `MOCK` | `MOCK` / `CARD` / `POINT` |
| `approved_at` | datetime |  | 승인 시간 |
| `cancelled_at` | datetime |  | 취소 시간 |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

---

## 12. 알림 (Notification)

### `notification` (알림 정보)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `notification_id` | BIGINT | PK, increment | 알림 ID |
| `user_id` | BIGINT | not null | 알림 대상 사용자 |
| `event_id` | VARCHAR(100) | unique, not null | 멱등성 보장용 고유 이벤트 ID |
| `type` | VARCHAR(50) | not null | 알림 유형 (`PAYMENT_APPROVED` 등) |
| `title` | VARCHAR(100) | not null | 알림 제목 |
| `message` | VARCHAR(500) | not null | 알림 내용 |
| `is_read` | BOOLEAN | not null, default: `false` | 읽음 여부 |
| `created_at` | DATETIME | not null | 생성 시각 |
| `updated_at` | DATETIME | not null | 수정 시각 |

> 💡 **Indexes**
>
> - `uk_notification_event_id`: UNIQUE (`event_id`) - 멱등성 보장 (중복 알림 방지)
> - `idx_notification_user`: (`user_id`, `created_at`) - 사용자별 알림 조회

---

## 13. 관리자 (Admin)

### `admin_action_history` (관리자 처리 이력)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `history_id` | bigint | PK, increment | 이력 ID |
| `actor_id` | bigint | not null | 처리한 관리자 ID |
| `target_type` | varchar(20) | not null | `SELLER` / `PRODUCT` / `ADMIN_USER` |
| `target_id` | bigint | not null | 처리 대상 ID |
| `action` | varchar(20) | not null | `APPROVE` / `REJECT` / `FORCE_INACTIVE` / `REACTIVATE` / `GRANT_ADMIN` / `REVOKE_ADMIN` |
| `reason` | varchar(255) |  | 처리 사유 (반려/강제비활성화 시 필수) |
| `created_at` | datetime | not null | 생성일 |
| `updated_at` | datetime | not null | 수정일 |

> 💡 **Indexes**
>
> - `idx_aah_actor`: (actor_id, created_at)
> - `idx_aah_target`: (target_type, target_id, created_at)

### `user_security_actions` (회원 보안 조치 이력)

| **Column** | **Type** | **Attributes** | **Description** |
| --- | --- | --- | --- |
| `id` | bigint | PK, increment | 보안 조치 ID |
| `target_user_id` | bigint | not null | 대상 회원 ID |
| `admin_user_id` | bigint | not null | 조치한 관리자 ID |
| `action_type` | varchar(50) | not null | `SUSPEND` / `UNSUSPEND` / `FORCE_LOGOUT` |
| `reason_code` | varchar(100) | not null | 사유 코드 |
| `reason_detail` | varchar(1000) |  | 사유 상세 |
| `started_at` | datetime | not null | 조치 시작 시각 |
| `expires_at` | datetime |  | 조치 만료 시각 (정지 등) |
| `active` | boolean | not null | 활성 여부 |

> 💡 **Indexes**
>
> - `idx_usa_target_active`: (target_user_id, action_type, active)

---

## 🔗 테이블 간 관계 (Relations)

```
seller.user_id (1) ── (1) user.user_id
category.parent_id (N) ── (1) category.category_id

product.seller_id (N) ── (1) seller.seller_id
product_item.product_id (N) ── (1) product.product_id
product_category_mapping.product_id (N) ── (1) product.product_id
product_category_mapping.category_id (N) ── (1) category.category_id
product_image.product_id (N) ── (1) product.product_id
inventory_history.item_id (N) ── (1) product_item.item_id
product_tag.product_id (N) ── (1) product.product_id
related_product.base_product_id (N) ── (1) product.product_id
related_product.target_product_id (N) ── (1) product.product_id
search_history.user_id (N) ── (1) user.user_id

cart.user_id (1) ── (1) user.user_id
cart_item.cart_id (N) ── (1) cart.cart_id
cart_item.item_id (N) ── (1) product_item.item_id

orders.user_id (N) ── (1) user.user_id
orders.subscription_id (N) ── (1) subscription.subscription_id
orders.user_coupon_id (N) ── (1) user_coupon.user_coupon_id
order_item.order_id (N) ── (1) orders.order_id
order_item.item_id (N) ── (1) product_item.item_id
order_item.seller_id (N) ── (1) seller.seller_id
order_cancel_history.order_id (N) ── (1) orders.order_id
order_cancel_history.order_item_id (N) ── (1) order_item.order_item_id

delivery.order_item_id (1) ── (1) order_item.order_item_id
delivery_history.delivery_id (N) ── (1) delivery.delivery_id

review.user_id (N) ── (1) user.user_id
review.order_item_id (1) ── (1) order_item.order_item_id
review.product_id (N) ── (1) product.product_id
review_image.review_id (N) ── (1) review.review_id
product_review_summary.product_id (1) ── (1) product.product_id

point.user_id (1) ── (1) user.user_id
point_history.point_id (N) ── (1) point.point_id
point_history.user_id (N) ── (1) user.user_id
point_history.order_id (N) ── (1) orders.order_id
point_usage_detail.use_history_id (N) ── (1) point_history.history_id
point_usage_detail.source_history_id (N) ── (1) point_history.history_id

subscription.user_id (N) ── (1) user.user_id

user_coupon.user_id (N) ── (1) user.user_id
user_coupon.coupon_id (N) ── (1) coupon.coupon_id
user_coupon.used_order_id (1) ── (1) orders.order_id

payment.order_id (1) ── (1) orders.order_id

notification.user_id (N) ── (1) user.user_id
```