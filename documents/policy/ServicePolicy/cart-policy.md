## 장바구니

- 비로그인 사용자:
    - Redis Hash 기반 수량 저장
    - Redis ZSet 기반 담기 순서 저장
    - guest_cart_id 쿠키로 식별
    - TTL 7일
- 로그인 사용자:
    - MySQL DB 저장
- 로그인 사용자 장바구니 조회 시 장바구니가 아직 생성되지 않은 경우 예외가 아닌 빈 장바구니를 반환한다.
    - cartId = null
    - items = []
    - totalPrice = 0
    - itemCount = 0
- 로그인 시:
    - Redis 비로그인 장바구니(Hash/ZSet) → DB 장바구니 merge
    - 동일 `guestCartId` 동시 merge 방지를 위해 `lock:cart:merge:{guestCartId}` 기준 Redis Lock을 사용한다.
    - Lock 획득에 성공한 요청만 Redis 장바구니를 조회하고 DB 장바구니로 merge한다.
    - Lock 획득에 실패한 요청은 Redis 장바구니 조회와 DB merge를 수행하지 않는다.
    - merge 성공 시 Redis Hash key, ZSet key, guest_cart_id 쿠키를 삭제한다.
- 장바구니 상품 수정/삭제는 itemId 기준으로 처리한다.
- 장바구니 최대 50종 상품 저장 가능(페이징 기능 추가)
- 장바구니 조회 / 담기 / 수량 변경 / 삭제 시 Redis TTL과 guest_cart_id 쿠키 만료 시간을 함께 7일로 갱신한다.
- 수량:
    - 최소 1개
    - 최대 99개
    - 재고(stock) 초과 불가
- 동일 상품 옵션 존재 시 quantity 증가
- STOP / 품절 상품은 장바구니 추가 및 수량 증가 불가
- STOP / 품절 상품 장바구니 유지
- 본인 상품 장바구니 담기 불가
- CART 주문 생성 성공 시 해당 cart_item 자동 삭제
- 비로그인 장바구니 사용자가 로그인한 경우, Redis 장바구니는 DB 장바구니와 merge되며 이후 주문 생성은 merge된 DB 장바구니 기준으로 처리한다.
- 기존 DB 장바구니 상품도 함께 주문 대상에 포함될 수 있으므로, 로그인 후 장바구니 화면에서 최종 상품 목록을 확인한 뒤 주문을 생성한다.

### 로그인 장바구니 담기 검증 우선순위

- 로그인 사용자가 장바구니에 상품을 담을 때 검증 우선순위는 다음과 같다.
    1. 본인 상품 여부 검증
    2. STOP 상품 여부 검증
    3. 수량 범위 검증
        - 수량은 최소 1개, 최대 99개까지 허용한다.
    4. stock = 0 / 재고 초과 여부 검증
- 하나의 상품이 여러 제한 조건에 동시에 해당하는 경우, 위 우선순위에 따라 먼저 검증되는 정책의 예외를 반환한다.
- 본인 상품은 구매 대상이 아니므로 판매 상태나 수량/재고보다 먼저 검증한다.
- 수량 범위는 상품 재고와 무관한 요청값 자체 검증이므로 재고 검증보다 먼저 수행한다.

### 비로그인 장바구니 정책 핵심 정리

(자세한 건 아래 비로그인 장바구니 정책 참고)

- Redis 비로그인 장바구니는 Hash와 ZSet을 함께 사용하여 관리한다.
- Redis Hash는 itemId → quantity 구조로 상품 수량을 저장한다.
- Redis Hash key는 guest:cart:{guestCartId} 형식을 사용한다.
- Redis ZSet order key는 guest:cart-order:{guestCartId} 형식을 사용한다.
- 로그인 장바구니도 cart_id + item_id 유니크 제약을 기반으로 상품을 식별한다.
- 따라서 로그인/비로그인 모두 장바구니 상품 수정/삭제 기준은 itemId로 통일한다.
- cartItemId는 로그인 장바구니에서만 존재하는 DB 식별자이며, 비로그인 응답에서는 null로 내려준다.
- guest_cart_id 쿠키와 Redis TTL은 모두 7일로 맞춘다.
- 장바구니 조회 / 담기 / 수량 변경 / 삭제 시 TTL을 갱신한다.
- 로그인 merge 시 DB 장바구니를 우선 유지하고, Redis 장바구니는 50종 한도 내에서만 merge한다.
- 로그인 merge 시 Redis ZSet의 담기 순서를 기준으로 선착순 merge한다.
- 동일 `guestCartId`로 동시에 로그인 요청이 들어올 수 있으므로, 로그인 merge는 `guestCartId` 기준 Redis Lock으로 직렬화한다.
- Redis Lock key는 `lock:cart:merge:{guestCartId}` 형식을 사용한다.
- Lock 획득에 성공한 요청만 Redis Hash/ZSet을 조회하고 `CartMergeExecutor`를 통해 DB 장바구니에 merge한다.
- Lock 획득에 실패한 요청은 Redis Hash/ZSet 조회와 DB merge를 수행하지 않는다.
- 로그인 시 비로그인 장바구니 merge가 실패해도 로그인은 성공 처리한다.
- merge 실패 시 guest_cart_id 쿠키와 Redis 장바구니는 유지한다.
- merge 성공 시에만 Redis Hash key / ZSet key와 guest_cart_id 쿠키를 삭제한다.
- 로그인 시 merge 실패는 Fail-Open 정책으로 처리하며, 로그인은 항상 성공으로 응답한다.
- 비로그인 장바구니 조회 시 Redis Hash의 `itemId` 또는 `quantity` 값이 파싱 불가능하거나 수량 범위를 벗어난 경우 오염 데이터로 간주하고, 응답 및 계산 대상에서 제외한다.
- 동일 상품을 다시 장바구니에 담을 때 Redis에 저장된 기존 수량이 오염되어 있으면 기존 수량은 0으로 간주하고, 현재 요청 수량을 기준으로 Redis quantity 값을 다시 저장한다.

## 비로그인 장바구니 정책

- 비로그인 장바구니는 Redis Hash와 Redis ZSet을 함께 사용한다.
- Redis Hash는 장바구니 상품 수량 저장용으로 사용한다.
- Redis ZSet은 장바구니 상품의 최초 담기 순서 저장용으로 사용한다.
- 비로그인 사용자는 guest_cart_id 쿠키로 식별한다.
- guest_cart_id가 없으면 서버에서 UUID를 생성하여 쿠키로 내려준다.
- guest_cart_id 쿠키 만료 시간은 7일로 설정한다.
- Redis Hash Key는 guest:cart:{guestCartId} 형식을 사용한다.
- Redis Hash 구조:
    - Field: itemId
    - Value: quantity
- Redis ZSet Key는 guest:cart-order:{guestCartId} 형식을 사용한다.
- Redis ZSet 구조:
    - Member: itemId
    - Score: 최초 장바구니 담기 시각 timestamp
- Redis Hash key와 ZSet key의 TTL은 모두 7일로 설정한다.
- 장바구니 조회 / 담기 / 수량 변경 / 삭제 시 Redis Hash/ZSet TTL과 guest_cart_id 쿠키 만료 시간을 함께 7일로 갱신한다.
- 같은 itemId가 이미 존재하면 quantity를 증가시킨다.
- 장바구니 상품 종류는 최대 50개까지 허용한다.
- 수량은 최소 1개, 최대 99개까지 허용한다.
- 수량은 현재 재고를 초과할 수 없다.
- STOP 또는 stock = 0 상품은 장바구니 추가 및 수량 증가가 불가능하다.
- STOP 또는 stock = 0 상품이 이미 장바구니에 있는 경우 조회 시 유지한다.
- 비로그인 장바구니는 DB cart_item_id가 없으므로 Redis에서는 itemId로 장바구니 상품을 식별한다.

### 로그인 장바구니 식별 정책

- 로그인 장바구니는 MySQL DB에 저장한다.
- 로그인 장바구니 상품은 cart_id + item_id로 유일하게 식별한다.
- cart_items 테이블에는 cart_id + item_id 유니크 제약을 둔다.
- 같은 itemId가 이미 장바구니에 존재하면 새로운 cart_item을 생성하지 않고 기존 quantity를 증가시킨다.
- 장바구니 수량 변경 / 삭제 API는 로그인 여부와 관계없이 itemId 기준으로 처리한다.
    - 로그인 사용자: cartId + itemId로 DB CartItem 조회 후 수정/삭제
    - 비로그인 사용자: guest:cart:{guestCartId} Hash에서 itemId field 수정/삭제

### 장바구니 API 식별 기준

- 장바구니 상품 수정/삭제 요청은 cartItemId가 아닌 itemId 기준으로 처리한다.
- 로그인/비로그인 여부와 관계없이 프론트는 수정/삭제 요청 시 itemId를 사용한다.
    - PATCH /api/carts/items/{itemId}
    - DELETE /api/carts/items/{itemId}

### 로그인 사용자 처리

- 인증된 사용자인 경우:
    - 사용자의 장바구니를 조회한다.
    - cart_id + item_id 기준으로 CartItem을 조회한다.
    - 해당 CartItem의 수량을 변경하거나 삭제한다.
- 로그인 장바구니는 DB에 저장되므로 cartItemId가 존재한다.
- 단, API 요청의 식별 기준은 cartItemId가 아니라 itemId로 통일한다.

### 비로그인 사용자 처리

- 인증되지 않은 사용자인 경우:
    - guest_cart_id 쿠키를 조회한다.
    - guest_cart_id 쿠키가 없으면 UUID를 생성하여 쿠키로 내려준다.
    - Redis Hash Key guest:cart:{guestCartId}를 조회한다.
    - Redis ZSet Key guest:cart-order:{guestCartId}를 함께 사용하여 담기 순서를 관리한다.
    - 수량 변경은 Redis Hash Field인 itemId 기준으로 처리한다.
    - 상품 삭제 시 Redis Hash Field와 Redis ZSet Member에서 모두 해당 itemId를 제거한다.
- 비로그인 장바구니는 DB에 저장되지 않으므로 cartItemId가 존재하지 않는다.

### 장바구니 응답 정책

- 로그인 장바구니 응답에는 DB의 cartItemId를 포함할 수 있다.
- 비로그인 장바구니 응답에는 DB cartItemId가 없으므로 cartItemId = null로 내려준다.
- 프론트는 장바구니 상품 수정/삭제 시 항상 itemId를 사용한다.
- 따라서 cartItemId는 응답 참고용이며, 수정/삭제 API의 기준 값으로 사용하지 않는다.

### 로그인 사용자 응답 예시json

{
"cartItemId": 25,
"itemId": 10,
"itemName": "상품명",
"quantity": 2
}

### 비로그인 사용자 응답 예시json

{
"cartItemId": null,
"itemId": 10,
"itemName": "상품명",
"quantity": 2
}

### 로그인 시 장바구니 merge 정책

- 로그인 성공 시 guest_cart_id 쿠키가 있으면 Redis 장바구니를 조회한다.
- 로그인 merge는 동일 `guestCartId`에 대한 중복 merge를 방지하기 위해 Redis Lock을 사용한다.
- Redis Lock key는 `lock:cart:merge:{guestCartId}` 형식을 사용한다.
- Lock 획득에 성공한 요청만 Redis Hash key `guest:cart:{guestCartId}`와 Redis ZSet key `guest:cart-order:{guestCartId}`를 조회한다.
- Lock 획득에 실패한 요청은 Redis 조회와 DB merge를 수행하지 않는다.
- Lock 획득 실패는 로그인 실패로 처리하지 않으며, 비로그인 장바구니 merge 실패와 동일하게 Fail-Open 정책을 적용한다.
- Redis Lock은 merge 처리 완료 후 현재 스레드가 보유한 경우에만 해제한다.
- 현재 guest cart Redis 데이터는 Hash와 ZSet 두 key로 관리되므로, 단일 key `GETDEL` 대신 Redis Lock으로 merge 진입을 제어한다.
- 사용자의 DB 장바구니가 없으면 새로 생성한다.
- Redis 장바구니의 itemId가 DB 장바구니에 이미 있으면 수량을 합산한다.
- DB 장바구니의 기존 상품은 cart_id + item_id 기준으로 조회한다.
- 합산 결과는 최대 99개를 초과할 수 없다.
- 합산 결과가 재고보다 많으면 재고 수량까지만 반영한다.
- STOP 또는 stock = 0 상품은 merge하지 않는다.
- 로그인 merge 시 DB 장바구니와 Redis 장바구니를 합쳐 최대 50종까지만 유지한다.
- DB 장바구니에 이미 담긴 상품을 우선 유지한다.
- Redis 장바구니 상품은 50종 한도 내에서만 merge한다.
- 50종 한도를 초과한 Redis 상품은 merge하지 않고 삭제한다.
- merge가 정상 완료된 경우 Redis Hash key guest:cart:{guestCartId}를 삭제한다.
- merge가 정상 완료된 경우 Redis ZSet key guest:cart-order:{guestCartId}를 삭제한다.
- merge가 정상 완료된 경우 guest_cart_id 쿠키를 삭제한다.
- merge 중 예외가 발생하면 로그인은 성공 처리하고, Redis Hash/ZSet key와 guest_cart_id 쿠키는 유지한다.

### 50종 초과 merge 예시DB 장바구니: 30종

Redis 장바구니: 30종
겹치는 상품 없음

결과:

- DB 30종 유지
- Redis 상품 중 20종만 merge
- Redis 나머지 10종은 merge하지 않고 삭제
- 최종 DB 장바구니 50종

### 로그인 시 비로그인 장바구니 merge 실패 정책

- 비로그인 장바구니 merge는 Fail-Open 정책을 적용한다.
    - merge 실패가 로그인 실패로 이어지지 않도록 처리한다.
    - 사용자 경험을 최우선으로 하여 로그인은 항상 성공으로 응답한다.
- 로그인 성공 후 guest_cart_id 쿠키가 있으면 Redis 비로그인 장바구니를 DB 장바구니로 merge한다.
- 사용자 편의를 위해 비로그인 장바구니 merge 실패가 로그인 실패로 이어지지 않도록 처리한다.
- merge 중 예외가 발생해도 로그인은 정상 성공으로 응답한다.
- merge 실패 시 서버는 경고 로그를 남긴다.
- merge 실패 시 guest_cart_id 쿠키는 삭제하지 않는다.
- merge 실패 시 Redis Hash key와 ZSet key는 삭제하지 않는다.
- Redis Lock 획득 실패 또는 Lock 대기 중 인터럽트가 발생한 경우에도 merge 실패로 보고 Fail-Open 정책을 적용한다.
- 이 경우 Redis Hash/ZSet key와 guest_cart_id 쿠키는 삭제하지 않는다.
- Lock 획득 실패 요청은 Redis 장바구니 조회와 DB merge를 수행하지 않는다.
- 이후 사용자가 다시 로그인하거나 장바구니를 조회/수정할 때 재시도 또는 후속 처리가 가능하도록 한다.
- merge가 정상 완료된 경우에만 Redis Hash key와 ZSet key를 삭제한다.
- merge가 정상 완료된 경우에만 guest_cart_id 쿠키를 삭제한다.

### Redis 비로그인 장바구니 데이터 파싱 실패 처리 정책

- Redis 비로그인 장바구니는 외부 저장소에 저장되므로, 수동 조작이나 직렬화 문제 등으로 애플리케이션이 기대하는 형식과 다른 값이 저장될 수 있다.
- Redis Hash의 field인 `itemId`가 숫자로 파싱되지 않는 경우, 해당 항목은 오염된 데이터로 간주하고 조회/계산 대상에서 제외한다.
- Redis Hash의 value인 `quantity`가 숫자로 파싱되지 않는 경우, 해당 항목은 오염된 데이터로 간주하고 조회/계산 대상에서 제외한다.
- Redis Hash의 `quantity`가 1 미만이거나 99를 초과하는 경우, 장바구니 수량 정책을 위반한 오염 데이터로 간주하고 조회/계산 대상에서 제외한다.
- 파싱 실패 또는 수량 범위 오류가 발생해도 500 에러로 전파하지 않고, 경고 로그를 남긴 뒤 해당 항목만 skip한다.
- 비로그인 장바구니 조회 시 오염된 항목은 응답 목록, `totalPrice`, `itemCount` 계산에서 제외한다.
- 동일 상품을 다시 장바구니에 담을 때 Redis에 저장된 기존 수량이 오염되어 있으면 기존 수량은 0으로 간주하고, 현재 요청 수량을 기준으로 Redis quantity 값을 다시 저장한다.
- Redis 파싱 실패 로그에는 추적을 위해 `itemId field`와 `quantity value`를 포함한다.