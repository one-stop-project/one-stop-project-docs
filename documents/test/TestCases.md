# 테스트 케이스

> # 목차
>
>
> - [ADMIN](#admin)
> - [AI](#ai)
> - [AUTH](#auth)
> - [CART](#cart)
> - [CATEGORY](#category)
> - [COUPON](#coupon)
> - [DELIVERY](#delivery)
> - [NOTIFICATION](#notification)
> - [ORDER](#order)
> - [PAYMENT](#payment)
> - [PRODUCT](#product)
> - [POINT](#point)
> - [REVIEW](#review)
> - [SECURITY](#security)
> - [SEARCH](#search)
> - [SUBSCRIPTION](#subscription)
> - [USER](#user)

## ADMIN 

| NO.    | 도메인   | 기능명                       | 시나리오                                                   | Method | Endpoint                                       | 사전 조건                             | Request                                                      | Response                                                      | 상태코드 | 통과여부 |
| ------ | ----- | ------------------------- | ------------------------------------------------------ | ------ | ---------------------------------------------- | --------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------- | ---- | ---- |
| ADM001 | ADMIN | 상품 승인                     | 관리자가 대기 중인 상품을 정상 승인한다                                 | POST   | `/api/admin/products/{productId}/approve`      | ADMIN 권한 사용자 로그인 / 승인 대기 상품 존재    | 없음                                                           | `{"code": 200, "data": {"productId": 1, "status": "ACTIVE"}}` | 200  | 통과   |
| ADM002 | ADMIN | 상품 반려                     | APPROVE_REQUESTED 상태 상품을 반려하면 REJECTED로 변경된다           | PATCH  | `/api/admin/products/{productId}/reject`       | ADMIN 권한 사용자 로그인 / 승인 대기 상품 존재    | `{"reason": "반려 사유"}`                                        | `{"status": "REJECTED"}`                                      | 200  | 통과   |
| ADM003 | ADMIN | 상품 반려 - 비정상 상태            | APPROVE_REQUESTED 아닌 상태의 상품 반려 시도 → ADMIN_017 예외       | PATCH  | `/api/admin/products/{productId}/reject`       | ADMIN 권한 / 이미 승인된(APPROVED) 상품 존재 | `{"reason": "반려 사유"}`                                        | `{"code": "ADMIN_017"}`                                       | 400  | 통과   |
| ADM004 | ADMIN | 판매자 승인                    | PENDING 상태 판매자를 승인하면 APPROVED로 변경된다                    | PATCH  | `/api/admin/sellers/{sellerId}/approve`        | ADMIN 권한 / PENDING 판매자 존재         | 없음                                                           | `{"status": "APPROVED"}`                                      | 200  | 통과   |
| ADM005 | ADMIN | 판매자 반려                    | PENDING 상태 판매자를 반려하면 REJECTED로 변경된다                    | PATCH  | `/api/admin/sellers/{sellerId}/reject`         | ADMIN 권한 / PENDING 판매자 존재         | `{"reason": "반려 사유"}`                                        | `{"status": "REJECTED"}`                                      | 200  | 통과   |
| ADM006 | ADMIN | 판매자 정지                    | APPROVED 판매자를 정지하면 SUSPENDED로 변경된다                     | PATCH  | `/api/admin/sellers/{sellerId}/force-inactive` | ADMIN 권한 / APPROVED 판매자 존재        | `{"reason": "정지 사유"}`                                        | `{"status": "SUSPENDED"}`                                     | 200  | 통과   |
| ADM007 | ADMIN | 판매자 복구                    | SUSPENDED 판매자를 복구하면 APPROVED로 변경된다                     | PATCH  | `/api/admin/sellers/{sellerId}/reactivate`     | ADMIN 권한 / SUSPENDED 판매자 존재       | 없음                                                           | `{"status": "APPROVED"}`                                      | 200  | 통과   |
| ADM008 | ADMIN | 관리자 권한 부여                 | SUPER_ADMIN이 BUYER에게 ADMIN 권한을 부여하면 role이 ADMIN으로 변경된다 | PATCH  | `/api/admin/users/{userId}/grant`              | SUPER_ADMIN 권한 / BUYER 회원 존재      | 없음                                                           | `{"role": "ADMIN"}`                                           | 200  | 통과   |
| ADM009 | ADMIN | 관리자 권한 회수                 | SUPER_ADMIN이 ADMIN 권한을 회수하면 role이 BUYER로 변경된다          | PATCH  | `/api/admin/users/{userId}/revoke`             | SUPER_ADMIN 권한 / ADMIN 회원 존재      | 없음                                                           | `{"role": "BUYER"}`                                           | 200  | 통과   |
| ADM010 | ADMIN | 처리 이력 조회 - ADMIN          | ADMIN은 본인이 처리한 이력만 조회된다                                | GET    | `/api/admin/action-histories`                  | ADMIN 권한 / 본인 처리 이력 존재            | 없음                                                           | 본인 처리 이력 목록                                                   | 200  | 통과   |
| ADM011 | ADMIN | 처리 이력 조회 - SUPER_ADMIN    | SUPER_ADMIN은 전체 관리자 처리 이력을 조회할 수 있다                    | GET    | `/api/admin/action-histories`                  | SUPER_ADMIN 권한 / 여러 관리자 처리 이력 존재  | 없음                                                           | 전체 처리 이력 목록                                                   | 200  | 통과   |
| ADM012 | ADMIN | 쿠폰 생성                     | 유효한 요청으로 쿠폰을 생성하면 저장된 쿠폰 응답이 반환된다                      | POST   | `/api/admin/coupons`                           | ADMIN 권한                          | `{"name": "쿠폰명", "discountAmount": 1000, "totalCount": 100}` | `{"couponId": 1, "status": "ACTIVE"}`                         | 201  | 통과   |
| ADM013 | ADMIN | 쿠폰 비활성화                   | ACTIVE 쿠폰을 비활성화하면 INACTIVE 상태로 변경된다                    | PATCH  | `/api/admin/coupons/{couponId}/deactivate`     | ADMIN 권한 / ACTIVE 쿠폰 존재           | 없음                                                           | `{"status": "INACTIVE"}`                                      | 200  | 통과   |
| ADM014 | ADMIN | 이미 비활성 쿠폰 비활성화 실패         | 이미 INACTIVE 쿠폰 비활성화 시도 → COUPON_010 예외                 | PATCH  | `/api/admin/coupons/{couponId}/deactivate`     | ADMIN 권한 / INACTIVE 쿠폰 존재         | 없음                                                           | `{"code": "COUPON_010"}`                                      | 400  | 통과   |
| ADM015 | ADMIN | 선행 상태 검증 - 비정상 판매자 승인     | PENDING 아닌 판매자 승인 시도 → ADMIN_018 예외                    | PATCH  | `/api/admin/sellers/{sellerId}/approve`        | ADMIN 권한 / 이미 APPROVED 판매자 존재     | 없음                                                           | `{"code": "ADMIN_018"}`                                       | 400  | 통과   |
| ADM016 | ADMIN | 선행 상태 검증 - 비정상 판매자 반려     | PENDING 아닌 판매자 반려 시도 → ADMIN_018 예외                    | PATCH  | `/api/admin/sellers/{sellerId}/reject`         | ADMIN 권한 / 이미 APPROVED 판매자 존재     | `{"reason": "반려 사유"}`                                        | `{"code": "ADMIN_018"}`                                       | 400  | 통과   |
| ADM017 | ADMIN | 선행 상태 검증 - 이미 정지된 판매자 재정지 | 이미 SUSPENDED 판매자 재정지 시도 → ADMIN_019 예외                 | PATCH  | `/api/admin/sellers/{sellerId}/force-inactive` | ADMIN 권한 / 이미 SUSPENDED 판매자 존재    | `{"reason": "정지 사유"}`                                        | `{"code": "ADMIN_019"}`                                       | 400  | 통과   |
| ADM018 | ADMIN | 선행 상태 검증 - 비정지 판매자 복구     | SUSPENDED 아닌 판매자 복구 시도 → ADMIN_010 예외                  | PATCH  | `/api/admin/sellers/{sellerId}/reactivate`     | ADMIN 권한 / APPROVED 판매자 존재        | 없음                                                           | `{"code": "ADMIN_010"}`                                       | 400  | 통과   |


## AI
| NO.   | 도메인 | 기능명 | 시나리오 | Method | Endpoint | 사전 조건 | Request | Response | 상태코드 | 통과여부 |
|-------|--------|--------|----------|--------|----------|------------|----------|-----------|-----------|----------|
| AI001 | AI | AI 리뷰 요약 조회 - READY | 요약 DB에 존재 → READY 상태 반환 | GET | /api/products/{productId}/review-summary | 요약이 DB에 저장된 상품 존재 | 없음 | {"status":"READY","summary":{...}} | 200 | 통과 |
| AI002 | AI | AI 리뷰 요약 조회 - INSUFFICIENT | 리뷰 5개 미만 → INSUFFICIENT 반환 | GET | /api/products/{productId}/review-summary | 리뷰 4개 이하 상품 | 없음 | {"status":"INSUFFICIENT"} | 200 | 통과 |
| AI003 | AI | AI 리뷰 요약 조회 - PENDING | 리뷰 5개 이상 + 요약 없음 | GET | /api/products/{productId}/review-summary | 리뷰 5개 이상 / 요약 미생성 | 없음 | {"status":"PENDING"} | 200 | 통과 |
| AI004 | AI | AI 리뷰 요약 생성 - 의류 | 의류 카테고리 리뷰 요약 생성 | POST | /api/admin/products/{productId}/review-summary/refresh | ADMIN / 리뷰 5개 이상 | 없음 | READY summary | 200 | 통과 |
| AI005 | AI | AI 리뷰 요약 생성 - 전자제품 | 전자제품 카테고리 리뷰 요약 생성 | POST | /api/admin/products/{productId}/review-summary/refresh | ADMIN / 리뷰 5개 이상 | 없음 | READY summary | 200 | 통과 |
| AI006 | AI | AI 요약 성공 시 토큰 로거 호출 | 성공 시 토큰 사용량 로깅 | POST | /api/admin/products/{productId}/review-summary/refresh | AI 성공 | 없음 | 토큰 로깅 | 200 | 통과 |
| AI007 | AI | ChatClient 예외 처리 | 예외 발생 시 실패 로그 + 예외 전파 | POST | /api/admin/products/{productId}/review-summary/refresh | ChatClient 에러 | 없음 | 500 | 500 | 통과 |
| AI008 | AI | Fallback - UNAVAILABLE | AI 장애 시 fallback 반환 | GET | /api/products/{productId}/review-summary | AI 장애 | 없음 | UNAVAILABLE | 200 | 통과 |
| AI009 | AI | Circuit OPEN fallback | Circuit OPEN 시 fallback | GET | /api/products/{productId}/review-summary | CB OPEN | 없음 | UNAVAILABLE | 200 | 통과 |
| AI010 | AI | 카테고리 fallback 통일 | 모든 카테고리 fallback 동일 | GET | /api/products/{productId}/review-summary | 장애 | 없음 | UNAVAILABLE | 200 | 통과 |
| AI011 | AI | 비인증 접근 차단 | JWT 없이 AI 호출 차단 | POST | /api/ai/assistant | 미로그인 | {message} | 401 | 401 | 통과 |
| AI012 | AI | 존재하지 않는 상품 | 없는 상품 요청 | POST | /api/admin/products/{productId}/review-summary/refresh | ADMIN | 없음 | PRODUCT_001 | 404 | 통과 |
| AI013 | AI | 리뷰 부족 AI 미호출 | 리뷰 부족 시 AI 호출 안함 | POST | /api/admin/products/{productId}/review-summary/refresh | 리뷰 4개 이하 | 없음 | INSUFFICIENT | 200 | 통과 |
| AI014 | AI | 메시지 500자 초과 | validation 실패 | POST | /api/ai/assistant | BUYER | message > 500 | 400 | 400 | 통과 |
| AI015 | AI | 연관상품 품절 제외 | 품절 제외 추천 | GET | /api/products/{productId}/related | stock=0 존재 | 없음 | 목록 | 200 | 통과 |
| AI016 | AI | CB OPEN fallback | AI assistant fallback | POST | /api/ai/assistant | CB OPEN | message | fallback | 200 | 통과 |
| AI017 | AI | 판매수 내림차순 | 판매수 기준 정렬 | GET | /api/products/{productId}/related | 데이터 존재 | 없음 | 정렬 결과 | 200 | 통과 |
| AI018 | AI | 카테고리 없음 | 빈 목록 반환 | GET | /api/products/{productId}/related | category 없음 | 없음 | [] | 200 | 통과 |
| AI019 | AI | 자연어 추천(카테고리 포함) | categoryId 기반 추천 | POST | /api/ai/assistant | BUYER | message + categoryId | 500 (버그) | 200 | 미통과 |
| AI020 | AI | 빈 메시지 검증 | @NotBlank 실패 | POST | /api/ai/assistant | BUYER | "" | 400 | 400 | 통과 |
| AI021 | AI | 연관상품 포함 | 같은 카테고리 포함 | GET | /api/products/{productId}/related | category 존재 | 없음 | 목록 | 200 | 통과 |
| AI022 | AI | 자연어 추천(카테고리 없음) | LLM 추천 호출 | POST | /api/ai/assistant | BUYER | message | 500 (버그) | 200 | 미통과 |
| AI023 | AI | 자기 자신 제외 | 기준 상품 제외 | GET | /api/products/{productId}/related | 상품 존재 | 없음 | 제외된 목록 | 200 | 통과 |

## AUTH

| NO.     | 도메인  | 기능명                | 시나리오                           | Method | Endpoint                   | 사전 조건                    | Request                           | Response          | 상태코드      | 통과여부 |
| ------- | ---- | ------------------ | ------------------------------ | ------ | -------------------------- | ------------------------ | --------------------------------- | ----------------- | --------- | ---- |
| AUTH001 | Auth | 회원가입               | BUYER 회원가입 성공                  | POST   | `/api/auth/signup`         | 미가입 이메일                  | email, password, name, role=BUYER | 회원가입 성공           | 201       | 통과   |
| AUTH002 | Auth | 회원가입               | SELLER 회원가입 성공                 | POST   | `/api/auth/signup`         | 판매자 필수 정보 포함             | role=SELLER                       | 회원가입 성공           | 201       | 통과   |
| AUTH003 | Auth | 회원가입               | ADMIN/SUPER_ADMIN 가입 차단        | POST   | `/api/auth/signup`         | 일반 회원가입 API 사용           | role=ADMIN                        | 권한 가입 차단          | 400       | 통과   |
| AUTH004 | Auth | 회원가입               | 중복 이메일 가입 차단                   | POST   | `/api/auth/signup`         | 이미 존재하는 이메일              | 동일 email 요청                       | 중복 이메일 예외         | 409       | 통과   |
| AUTH005 | Auth | 회원가입 Rate Limit    | 동일 IP 회원가입 과다 차단               | POST   | `/api/auth/signup`         | 동일 IP 반복 요청              | 반복 signup 요청                      | Rate Limit 차단     | 429       | 통과   |
| AUTH006 | Auth | 로그인                | 정상 로그인 성공                      | POST   | `/api/auth/login`          | 가입된 활성 사용자               | email, password                   | 로그인 성공            | 200       | 통과   |
| AUTH007 | Auth | 로그인                | 로그인 전 Rate Limit 적용            | POST   | `/api/auth/login`          | 로그인 요청 발생                | email, password                   | 제한 내 요청 통과        | 200       | 통과   |
| AUTH008 | Auth | 로그인                | 계정 단위 로그인 과다 요청 차단             | POST   | `/api/auth/login`          | 동일 계정 반복 로그인             | 동일 email 반복                       | Rate Limit 차단     | 429       | 통과   |
| AUTH009 | Auth | 로그인                | IP 단위 로그인 과다 요청 차단             | POST   | `/api/auth/login`          | 동일 IP 반복 요청              | 반복 login 요청                       | Rate Limit 차단     | 429       | 통과   |
| AUTH010 | Auth | 로그인 부하             | 다수 계정 로그인 부하 검증                | POST   | `/api/auth/login`          | k6용 구매자 계정 생성            | k6 login users                    | 임계치 내 처리          | 200 중심    | 통과   |
| AUTH011 | Auth | 로그인 Rate Limit     | device_id 없는 신규 기기 로그인 반복 시 차단 | POST   | `/api/auth/login`          | 동일 IP + 신규 기기 반복         | device_id 없는 로그인 반복               | Rate Limit 응답     | 429       | 통과   |
| AUTH012 | Auth | Device ID          | 기존 device_id 로그인 시 재사용         | POST   | `/api/auth/login`          | device_id 쿠키 존재          | Cookie: device_id                 | 기존 기기 로그인 성공      | 200       | 통과   |
| AUTH013 | Auth | 신규 기기              | 신규 device_id 로그인 시 등록          | POST   | `/api/auth/login`          | 새로운 device_id            | Cookie: 신규 device_id              | 기기 등록 성공          | 200       | 통과   |
| AUTH014 | Auth | LRU Eviction       | 5대 초과 로그인 시 오래된 기기 제거          | POST   | `/api/auth/login`          | 이미 5대 등록                 | 신규 device_id 로그인                  | 가장 오래된 기기 제거      | 200       | 통과   |
| AUTH015 | Auth | Device Limit       | 10개 기기 로그인                     | POST   | `/api/auth/login`          | k6 계정 / 다수 device_id     | 10개 device login                  | 활성 기기 5개 유지       | 200       | 통과   |
| AUTH016 | Auth | OAuth2 기기 처리       | 기존 device_id 재사용               | GET    | `/login/oauth2/code/kakao` | OAuth2 성공 + device_id 존재 | Cookie: device_id                 | callback redirect | 302       | 통과   |
| AUTH017 | Auth | OAuth2 신규 기기       | 신규 OAuth2 로그인                  | GET    | `/login/oauth2/code/kakao` | 신규 device_id             | OAuth2 callback                   | 기기 등록 성공          | 302       | 통과   |
| AUTH018 | Auth | OAuth2 LRU 제거      | OAuth2 로그인 시 오래된 기기 제거         | GET    | `/login/oauth2/code/kakao` | 5대 초과 상태                 | OAuth2 callback                   | LRU 제거            | 302       | 통과   |
| AUTH019 | Auth | Refresh Token      | 정상 Refresh 성공                  | POST   | `/api/auth/refresh`        | 유효 RT + device_id        | refresh token + device_id         | Access Token 재발급  | 200       | 통과   |
| AUTH020 | Auth | Refresh Token      | device_id 불일치 차단               | POST   | `/api/auth/refresh`        | RT + 다른 device_id        | 불일치 요청                            | 인증 실패             | 401       | 통과   |
| AUTH021 | Auth | Refresh Token      | 미등록 기기 Refresh 차단              | POST   | `/api/auth/refresh`        | Redis 미등록 device         | RT + device_id                    | 등록되지 않은 기기 예외     | 401       | 통과   |
| AUTH022 | Auth | Refresh Rotation   | CAS 실패 시 Refresh 실패            | POST   | `/api/auth/refresh`        | 이미 회전된 RT                | old refresh token                 | 인증 실패             | 401       | 통과   |
| AUTH023 | Auth | RTR Race Condition | 100개 동시 Refresh                | POST   | `/api/auth/refresh`        | 동일 RT + device_id        | 100 concurrent                    | 1 성공, 나머지 실패      | 200 / 401 | 통과   |
| AUTH024 | Auth | Refresh Context    | 환경 변화 감지                       | POST   | `/api/auth/refresh`        | IP/UA 변경                 | changed context                   | audit 기록 + 성공     | 200       | 통과   |
| AUTH025 | Auth | Refresh 후 LRU 갱신   | 활동 시간 갱신                       | POST   | `/api/auth/refresh`        | 유효 기기                    | refresh request                   | 갱신 성공             | 200       | 통과   |
| AUTH026 | Auth | 로그아웃               | Refresh Token 로그아웃             | POST   | `/api/auth/logout`         | refresh token 존재         | cookie 기반                         | 로그아웃 성공           | 200       | 통과   |
| AUTH027 | Auth | 로그아웃               | Access Token 블랙리스트             | POST   | `/api/auth/logout`         | 유효 AT 존재                 | Authorization header              | 로그아웃 성공           | 200       | 통과   |
| AUTH028 | Auth | 로그아웃               | Rate Limit 적용                  | POST   | `/api/auth/logout`         | 반복 요청                    | 동일 IP 반복                          | Rate Limit 차단     | 429       | 통과   |



## CART

| NO.     | 도메인  | 기능명              | 시나리오                           | Method | Endpoint                    | 사전 조건                      | Request                          | Response                                                               | 상태코드 | 통과여부 |
| ------- | ---- | ---------------- | ------------------------------ | ------ | --------------------------- | -------------------------- | -------------------------------- | ---------------------------------------------------------------------- | ---- | ---- |
| CART001 | CART | 회원 장바구니 담기       | 로그인 사용자가 ON_SALE 상품을 장바구니에 담는다 | POST   | `/api/carts/items`          | BUYER 로그인 / ON_SALE 상품     | `{"itemId": 1, "quantity": 2}`   | `{"cartItemId": 1, "quantity": 2, "message": "장바구니에 추가되었습니다."}`        | 200  | 통과   |
| CART002 | CART | 동일 상품 수량 누적      | 이미 담긴 상품을 다시 담으면 수량이 누적된다      | POST   | `/api/carts/items`          | BUYER 로그인 / 기존 CartItem 존재 | `{"itemId": 1, "quantity": 2}`   | `{"cartItemId": 1, "quantity": 5, "message": "장바구니에 추가되었습니다."}`        | 200  | 통과   |
| CART003 | CART | STOP 상품 담기 차단    | STOP 상품을 담으면 에러 발생             | POST   | `/api/carts/items`          | BUYER 로그인 / STOP 상품        | `{"itemId": 1, "quantity": 1}`   | `{"code": "CART_001", "message": "장바구니에 담을 수 없는 상품입니다"}`               | 400  | 통과   |
| CART004 | CART | 99개 초과 차단        | 수량 100개 이상 입력 시 에러 발생          | POST   | `/api/carts/items`          | BUYER 로그인                  | `{"itemId": 1, "quantity": 100}` | `{"code": "CART_002", "message": "수량은 1개 이상 99개 이하여야 합니다"}`            | 400  | 통과   |
| CART005 | CART | 장바구니 조회          | 로그인 사용자의 장바구니 조회               | GET    | `/api/carts`                | BUYER 로그인 / CartItem 존재    | 없음                               | `{"cartId": 1, "content": [...], "totalPrice": 20000, "itemCount": 1}` | 200  | 통과   |
| CART006 | CART | 수량 변경            | 장바구니 상품 수량 변경                  | PATCH  | `/api/carts/items/{itemId}` | BUYER 로그인 / CartItem 존재    | `{"quantity": 5}`                | `{"itemId": 1, "quantity": 5}`                                         | 200  | 통과   |
| CART007 | CART | 상품 삭제            | 장바구니 상품 삭제                     | DELETE | `/api/carts/items/{itemId}` | BUYER 로그인 / CartItem 존재    | 없음                               | 없음                                                                     | 200  | 통과   |
| CART008 | CART | 비로그인 담기          | 비로그인 상태 장바구니 담기                | POST   | `/api/carts/items`          | 비로그인 / ON_SALE 상품          | `{"itemId": 1, "quantity": 2}`   | `{"cartItemId": null, "quantity": 2, "message": "장바구니에 추가되었습니다."}`     | 200  | 통과   |
| CART009 | CART | 비로그인 조회          | 비로그인 장바구니 조회                   | GET    | `/api/carts`                | guest_cart_id 쿠키           | 없음                               | `{"cartId": null, "content": [...], "totalPrice": 20000}`              | 200  | 통과   |
| CART010 | CART | 로그인 시 merge      | 비로그인 장바구니 로그인 시 병합             | POST   | `/api/auth/login`           | guest cart 존재              | login request                    | accessToken 반환                                                         | 200  | 통과   |
| CART011 | CART | 게스트 STOP 수량수정 차단 | 비로그인 STOP 상품 수량 수정             | PATCH  | `/api/carts/items/{itemId}` | STOP 상품                    | `{"quantity": 3}`                | `{"code": "CART_001"}`                                                 | 400  | 통과   |



## CATEGORY 

| NO.     | 도메인      | 기능명              | Method | Endpoint               | Request                           | Response       | 상태코드 | 권한/조건             | 시나리오           | 통과여부 |
| ------- | -------- | ---------------- | ------ | ---------------------- | --------------------------------- | -------------- | ---- | ----------------- | -------------- | ---- |
| CAT-001 | CATEGORY | 카테고리 트리 조회       | GET    | `/api/categories`      | 없음                                | 3단 트리          | 200  | 공개 API            | 트리 구조 및 정렬 검증  | 통과   |
| CAT-002 | CATEGORY | 카테고리 생성          | POST   | `/api/categories`      | `{"name":"전자기기","parentId":null}` | 생성된 카테고리       | 200  | ADMIN/SUPER_ADMIN | 루트/자식 생성       | 통과   |
| CAT-003 | CATEGORY | 카테고리 생성(trim)    | POST   | `/api/categories`      | `"  전자기기  "`                      | `"전자기기"`       | 200  | ADMIN/SUPER_ADMIN | trim 처리 검증     | 통과   |
| CAT-004 | CATEGORY | 중복 이름 차단         | POST   | `/api/categories`      | 동일 이름                             | `CATEGORY_002` | 409  | 동일 부모             | 중복 생성 차단       | 통과   |
| CAT-005 | CATEGORY | 존재하지 않는 parentId | POST   | `/api/categories`      | parentId=99999                    | `CATEGORY_001` | 404  | ADMIN/SUPER_ADMIN | 부모 없음          | 통과   |
| CAT-006 | CATEGORY | 최대 depth 초과      | POST   | `/api/categories`      | 4단 생성                             | `CATEGORY_003` | 400  | depth=3 제한        | 깊이 제한 검증       | 통과   |
| CAT-007 | CATEGORY | 카테고리 수정          | PATCH  | `/api/categories/{id}` | `{"name":"가전품"}`                  | 수정된 카테고리       | 200  | ADMIN/SUPER_ADMIN | 이름 변경          | 통과   |
| CAT-008 | CATEGORY | 수정 중복 검증         | PATCH  | `/api/categories/{id}` | 중복 이름                             | `CATEGORY_002` | 409  | 형제 중복             | 수정 중 중복 차단     | 통과   |
| CAT-009 | CATEGORY | 카테고리 삭제          | DELETE | `/api/categories/{id}` | 없음                                | success        | 200  | ADMIN/SUPER_ADMIN | 하위 포함 삭제       | 통과   |
| CAT-010 | CATEGORY | 상품 매핑 삭제 차단      | DELETE | `/api/categories/{id}` | 없음                                | `CATEGORY_004` | 400  | 상품 존재             | 삭제 제한          | 통과   |
| CAT-011 | CATEGORY | 존재하지 않는 삭제       | DELETE | `/api/categories/{id}` | 없음                                | `CATEGORY_001` | 404  | 존재하지 않음           | 삭제 실패          | 통과   |
| CAT-012 | CATEGORY | 동시성 생성           | POST   | `/api/categories`      | 동일 이름 동시 요청                       | `CATEGORY_002` | 409  | UNIQUE constraint | race condition | 통과   |


## COUPON

| NO.    | 도메인    | 기능명         | 시나리오                 | Method | Endpoint                        | 사전 조건                     | Request                      | Response                                                                                                                       | 상태코드 | 통과여부 |
| ------ | ------ | ----------- | -------------------- | ------ | ------------------------------- | ------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ---- | ---- |
| CPN001 | COUPON | 발급 가능 목록 조회 | ACTIVE 쿠폰 목록 조회      | GET    | `/api/coupons/available`        | BUYER 로그인                 | 없음                           | `[{couponId: 1, name: "3000원 할인 쿠폰", discountType: "FIXED", discountValue: 3000, minOrderPrice: 5000, remainingQuantity: 99}]` | 200  | 통과   |
| CPN002 | COUPON | 선착순 발급      | 쿠폰 발급 후 AVAILABLE 저장 | POST   | `/api/coupons/{couponId}/issue` | BUYER / ACTIVE 쿠폰 / 잔여 있음 | 없음                           | `{userCouponId: 1, couponName: "3000원 할인 쿠폰", status: "AVAILABLE"}`                                                            | 200  | 통과   |
| CPN003 | COUPON | 중복 발급 차단    | 동일 쿠폰 재발급 시도         | POST   | `/api/coupons/{couponId}/issue` | 이미 발급됨                    | 없음                           | `{code: "COUPON_002", message: "이미 다운로드한 쿠폰입니다"}`                                                                              | 409  | 통과   |
| CPN004 | COUPON | 수량 소진 발급 실패 | 잔여 수량 0 쿠폰 발급        | POST   | `/api/coupons/{couponId}/issue` | 수량 소진 쿠폰                  | 없음                           | `{code: "COUPON_001", message: "쿠폰이 모두 소진되었습니다"}`                                                                              | 400  | 통과   |
| CPN005 | COUPON | 내 쿠폰 목록 조회  | 보유 쿠폰 조회             | GET    | `/api/users/me/coupons`         | BUYER 로그인                 | 없음                           | `[{userCouponId: 1, couponName: "3000원 할인 쿠폰", status: "AVAILABLE", expiredAt: "2026-07-19T00:00:00"}]`                        | 200  | 통과   |
| CPN006 | COUPON | 주문 시 쿠폰 적용  | 주문 시 할인 적용           | POST   | `/api/orders`                   | BUYER / AVAILABLE 쿠폰      | order payload + userCouponId | `{orderId: 1, discountPrice: 3000, finalPrice: 20000}`                                                                         | 201  | 통과   |
| CPN007 | COUPON | 최소주문금액 미충족  | 쿠폰 조건 미달 주문          | POST   | `/api/orders`                   | minOrderPrice 미충족         | order payload                | `{code: "COUPON_005", message: "이 쿠폰은 최소 주문 금액 미달입니다"}`                                                                        | 400  | 통과   |
| CPN008 | COUPON | 주문 취소 쿠폰 복구 | 주문 취소 시 쿠폰 복구        | POST   | `/api/orders/{orderId}/cancel`  | PAID 주문 / 쿠폰 사용           | `{reason: "변심"}`             | `{orderId: 1, status: "CANCELLED", restoredCoupon: {userCouponId: 1, status: "AVAILABLE"}}`                                    | 200  | 통과   |
| CPN009 | COUPON | 기간 만료 쿠폰 사용 | 만료 쿠폰 주문 시도          | POST   | `/api/orders`                   | 만료 쿠폰                     | order payload                | `{code: "COUPON_003", message: "쿠폰 발급 기간이 아닙니다"}`                                                                              | 400  | 통과   |

## DELIVERY 

| NO.    | 도메인      | 기능명             | 시나리오                                    | Method | Endpoint                                     | 사전 조건              | Request           | Response       | 상태코드 | 통과여부 |
| ------ | -------- | --------------- | --------------------------------------- | ------ | -------------------------------------------- | ------------------ | ----------------- | -------------- | ---- | ---- |
| DLV001 | DELIVERY | 구매자 배송 목록 조회    | 본인 주문 배송 목록 조회                          | GET    | `/api/orders/{orderId}/deliveries`           | BUYER / 결제 완료 주문   | 없음                | 배송 목록 (ACCEPT) | 200  | 통과   |
| DLV002 | DELIVERY | 타인 주문 조회 차단     | 타인 주문 배송 조회                             | GET    | `/api/orders/{orderId}/deliveries`           | 다른 사용자 주문 ID       | 없음                | `ORDER_007`    | 403  | 통과   |
| DLV003 | DELIVERY | 배송 이력 조회        | 배송 상태 이력 조회 (시간순)                       | GET    | `/api/deliveries/{deliveryId}/history`       | 본인 배송 ID           | 없음                | history list   | 200  | 통과   |
| DLV004 | DELIVERY | 배송 이력 조회 - 없음   | 존재하지 않는 배송 ID                           | GET    | `/api/deliveries/{deliveryId}/history`       | invalid deliveryId | 없음                | `SHIPPING_005` | 404  | 통과   |
| DLV005 | DELIVERY | 발주 확인 (CONFIRM) | ORDERED → CONFIRMED + ACCEPT → INSTRUCT | POST   | `/api/seller/orders/{orderItemId}/confirm`   | SELLER / ORDERED   | 없음                | 상태 변경          | 200  | 통과   |
| DLV006 | DELIVERY | 타 판매자 발주 확인 차단  | 다른 판매자 주문 처리                            | POST   | `/api/seller/orders/{orderItemId}/confirm`   | 타 판매자 orderItem    | 없음                | `SELLER_007`   | 403  | 통과   |
| DLV007 | DELIVERY | 중복 발주 확인 차단     | 이미 처리된 주문 confirm                       | POST   | `/api/seller/orders/{orderItemId}/confirm`   | CONFIRMED 이상       | 없음                | `SELLER_008`   | 400  | 통과   |
| DLV008 | DELIVERY | 주문 거절           | ORDERED → REJECTED                      | POST   | `/api/seller/orders/{orderItemId}/reject`    | SELLER / ORDERED   | `{reason}`        | 재고 복구 + 상태 변경  | 200  | 통과   |
| DLV009 | DELIVERY | 타 판매자 주문 거절 차단  | 다른 판매자 reject                           | POST   | `/api/seller/orders/{orderItemId}/reject`    | 타 판매자 orderItem    | `{reason}`        | `SELLER_007`   | 403  | 통과   |
| DLV010 | DELIVERY | 운송장 등록          | INSTRUCT → DEPARTURE                    | POST   | `/api/seller/deliveries/{deliveryId}/ship`   | INSTRUCT 상태        | courier + invoice | 배송 시작          | 200  | 통과   |
| DLV011 | DELIVERY | 잘못된 상태에서 ship   | INSTRUCT 아닌 상태 ship                     | POST   | `/api/seller/deliveries/{deliveryId}/ship`   | ACCEPT 상태          | courier + invoice | `SHIPPING_001` | 400  | 통과   |
| DLV012 | DELIVERY | 타 판매자 배송 관리 차단  | 다른 배송 ship                              | POST   | `/api/seller/deliveries/{deliveryId}/ship`   | 다른 판매자 delivery    | courier + invoice | `SHIPPING_006` | 403  | 통과   |
| DLV013 | DELIVERY | 배송 상태 변경        | DEPARTURE → DELIVERING                  | PATCH  | `/api/seller/deliveries/{deliveryId}/status` | DEPARTURE          | `DELIVERING`      | 상태 변경          | 200  | 통과   |
| DLV014 | DELIVERY | 배송 완료           | DELIVERING → FINAL_DELIVERY             | PATCH  | `/api/seller/deliveries/{deliveryId}/status` | DELIVERING         | `FINAL_DELIVERY`  | 포인트/이벤트 처리     | 200  | 통과   |
| DLV015 | DELIVERY | 잘못된 상태 전이       | 허용되지 않은 상태 변경                           | PATCH  | `/api/seller/deliveries/{deliveryId}/status` | ACCEPT             | `DELIVERING`      | `SHIPPING_002` | 400  | 통과   |


## NOTIFICATION

| NO.     | 도메인          | 기능명      | 시나리오                           | Method     | Endpoint                       | 사전 조건             | Request | Response                                        | 상태코드 | 통과여부 |
| ------- | ------------ | -------- | ------------------------------ | ---------- | ------------------------------ | ----------------- | ------- | ----------------------------------------------- | ---- | ---- |
| NOTI001 | NOTIFICATION | SSE 구독   | SSE 연결 후 connect 이벤트 수신        | GET        | `/api/notifications/subscribe` | BUYER 로그인         | 없음      | `event: connect / data: {"message":"연결되었습니다."}` | 200  | 통과   |
| NOTI002 | NOTIFICATION | 결제 알림 수신 | 결제 완료 시 PAYMENT_APPROVED 알림 수신 | SSE Stream | -                              | BUYER / SSE 연결 상태 | 없음      | `PAYMENT_APPROVED notification message`         | -    | 통과   |
| NOTI003 | NOTIFICATION | SSE 재연결  | 페이지 새로고침 후 SSE 재연결             | GET        | `/api/notifications/subscribe` | 기존 SSE 연결 존재      | 없음      | connect 이벤트 (기존 연결 교체)                          | 200  | 통과   |


## ORDER

| NO.    | 도메인   | 기능명           | 시나리오                     | Method | Endpoint                       | 사전 조건                 | Request         | Response                | 상태코드 | 통과여부 |
| ------ | ----- | ------------- | ------------------------ | ------ | ------------------------------ | --------------------- | --------------- | ----------------------- | ---- | ---- |
| ORD001 | ORDER | DIRECT 주문 생성  | 바로구매 주문 생성 + 재고 차감       | POST   | `/api/orders`                  | BUYER / ON_SALE 재고 충분 | order payload   | 주문 생성 + PENDING_PAYMENT | 201  | 통과   |
| ORD002 | ORDER | CART 주문 생성    | 장바구니 주문 생성 + CartItem 삭제 | POST   | `/api/orders`                  | BUYER / CartItem 존재   | cartItemIds     | 주문 생성                   | 201  | 통과   |
| ORD003 | ORDER | 쿠폰 적용 주문      | 쿠폰 할인 적용 주문 생성           | POST   | `/api/orders`                  | AVAILABLE 쿠폰 보유       | userCouponId 포함 | 할인 적용 주문                | 201  | 통과   |
| ORD004 | ORDER | 포인트 사용 주문     | 포인트 차감 주문 생성             | POST   | `/api/orders`                  | 포인트 충분                | usedPoint 포함    | 포인트 반영 주문               | 201  | 통과   |
| ORD005 | ORDER | 재고 부족 실패      | 재고 부족 주문 실패              | POST   | `/api/orders`                  | 재고 부족                 | order payload   | `INVENTORY_001`         | 400  | 통과   |
| ORD006 | ORDER | STOP 상품 주문 실패 | 판매 중지 상품 주문              | POST   | `/api/orders`                  | STOP 상품               | order payload   | `ORDER_003`             | 400  | 통과   |
| ORD007 | ORDER | 주문 목록 조회      | 본인 주문 목록 조회              | GET    | `/api/orders`                  | 주문 존재                 | 없음              | paging response         | 200  | 통과   |
| ORD008 | ORDER | PENDING 주문 취소 | 결제 전 주문 취소 + 재고 복구       | POST   | `/api/orders/{orderId}/cancel` | PENDING_PAYMENT       | reason          | CANCELLED + refund      | 200  | 통과   |
| ORD009 | ORDER | PAID 주문 취소    | 결제 후 취소 + 쿠폰/포인트 복구      | POST   | `/api/orders/{orderId}/cancel` | PAID                  | reason          | 복구 처리                   | 200  | 통과   |
| ORD010 | ORDER | 배송 시작 후 취소 실패 | DEPARTURE 이후 취소 실패       | POST   | `/api/orders/{orderId}/cancel` | DEPARTURE             | reason          | `ORDER_008`             | 400  | 통과   |
| ORD011 | ORDER | 할인 한도 초과 실패   | 쿠폰+포인트 초과 할인             | POST   | `/api/orders`                  | 과도한 할인                | usedPoint 포함    | `ORDER_012`             | 400  | 통과   |
| ORD012 | ORDER | 타인 주문 취소 실패   | 타인 주문 취소 차단              | POST   | `/api/orders/{orderId}/cancel` | 다른 사용자 주문             | reason          | `ORDER_007`             | 403  | 통과   |


## PAYMENT

| NO.    | 도메인     | 기능명         | 시나리오                     | Method | Endpoint        | 사전 조건                   | Request             | Response      | 상태코드 | 통과여부 |
| ------ | ------- | ----------- | ------------------------ | ------ | --------------- | ----------------------- | ------------------- | ------------- | ---- | ---- |
| PAY001 | PAYMENT | 결제 승인       | 결제 승인 시 주문/결제 상태 PAID 변경 | POST   | `/api/payments` | BUYER / PENDING_PAYMENT | `{orderId, amount}` | PAID 처리       | 200  | 통과   |
| PAY002 | PAYMENT | 포인트 사용 결제   | 포인트 사용 주문 결제 승인          | POST   | `/api/payments` | usedPoint 존재            | `{orderId, amount}` | 포인트 차감 + PAID | 200  | 통과   |
| PAY003 | PAYMENT | 금액 불일치 실패   | 요청 금액 불일치 시 실패           | POST   | `/api/payments` | PENDING_PAYMENT         | wrong amount        | `PAYMENT_002` | 400  | 통과   |
| PAY004 | PAYMENT | 이미 결제 완료    | PAID 주문 재결제 실패           | POST   | `/api/payments` | PAID                    | `{orderId, amount}` | `PAYMENT_001` | 400  | 통과   |
| PAY005 | PAYMENT | 취소 주문 결제 실패 | CANCELLED 주문 결제 실패       | POST   | `/api/payments` | CANCELLED               | `{orderId, amount}` | `PAYMENT_008` | 400  | 통과   |
| PAY006 | PAYMENT | 중복 결제 차단    | 동일 주문 Payment 중복 요청      | POST   | `/api/payments` | Payment 존재              | `{orderId, amount}` | `PAYMENT_003` | 409  | 통과   |
| PAY007 | PAYMENT | 타인 주문 결제 차단 | 타인 주문 결제 시도              | POST   | `/api/payments` | 다른 사용자 주문               | `{orderId, amount}` | `ORDER_007`   | 403  | 통과   |


## PRODUCT 

| NO. | 도메인 | 기능명 | 시나리오 | Method | Endpoint | 사전 조건 | Request | Response | 상태코드 | 통과여부 |
|------|--------|--------|----------|--------|----------|------------|----------|----------|----------|----------|
| PRD-001 | PRODUCT | 구매자 상품 상세 조회 | 정상 조회 | GET | /api/products/{productId} | APPROVED + ON_SALE | 없음 | 상품 상세 | 200 | 통과 |
| PRD-002 | PRODUCT | 구매자 상품 상세 조회 | 품절 포함 상품 조회 | GET | /api/products/{productId} | SOLD OUT 옵션 포함 | 없음 | soldOut=true | 200 | 통과 |
| PRD-003 | PRODUCT | 구매자 상품 상세 조회 | 존재하지 않는 상품 | GET | /api/products/{productId} | invalid productId | 없음 | PRODUCT_001 | 404 | 통과 |
| PRD-004 | PRODUCT | 구매자 상품 상세 조회 | 미노출 상품 | GET | /api/products/{productId} | 미승인/비판매 | 없음 | PRODUCT_002 | 400 | 통과 |
| PRD-005 | PRODUCT | 구매자 연관 상품 조회 | 연관 상품 조회 | GET | /api/products/{productId}/related | 카테고리 존재 | 없음 | 리스트 | 200 | 통과 |
| PRD-006 | PRODUCT | 판매자 상품 상세 조회 | 본인 상품 조회 | GET | /api/seller/products/{productId} | SELLER OWN | 없음 | 상품 전체 정보 | 200 | 통과 |
| PRD-007 | PRODUCT | 판매자 상품 상세 조회 | 타인 상품 접근 | GET | /api/seller/products/{productId} | 다른 판매자 상품 | 없음 | PRODUCT_008 | 403 | 통과 |
| PRD-008 | PRODUCT | 상품 삭제 | 정상 삭제 | DELETE | /api/seller/products/{productId} | 진행 주문 없음 | 없음 | DISCONTINUED | 200 | 통과 |
| PRD-009 | PRODUCT | 상품 삭제 | 진행 주문 존재 | DELETE | /api/seller/products/{productId} | 주문 진행 중 | 없음 | PRODUCT_009 | 400 | 통과 |
| PRD-010 | PRODUCT | 카테고리 검증 | 카테고리 없음 | PATCH | /api/seller/products/{id} | categoryIds=[] | 없음 | PRODUCT_012 | 400 | 통과 |
| PRD-011 | PRODUCT | 카테고리 검증 | 카테고리 초과 | PATCH | /api/seller/products/{id} | 4개 이상 | 없음 | PRODUCT_013 | 400 | 통과 |
| PRD-012 | PRODUCT | 입력 검증 | 잘못된 이름 | PATCH | /api/seller/products/{id} | invalid name | 없음 | COMMON_001 | 400 | 통과 |
| PRD-013 | PRODUCT | 이미지 추가 | 정상 업로드 | POST | /api/seller/products/{id}/images | 1~10장 | multipart | 이미지 추가 | 200 | 통과 |
| PRD-014 | PRODUCT | 이미지 제한 | 10장 초과 | POST | /api/seller/products/{id}/images | 초과 업로드 | multipart | PRODUCT_006 | 400 | 통과 |
| PRD-015 | PRODUCT | 파일 검증 | 0byte 파일 | POST | /api/seller/products/{id}/images | invalid file | multipart | COMMON_006 | 400 | 통과 |
| PRD-016 | PRODUCT | 이미지 삭제 | 대표 이미지 삭제 | DELETE | /api/seller/products/{id}/images/{imageId} | 이미지 2개 이상 | 없음 | 썸네일 재설정 | 200 | 통과 |
| PRD-017 | PRODUCT | 이미지 삭제 | 마지막 이미지 | DELETE | /api/seller/products/{id}/images/{imageId} | 1개만 존재 | 없음 | PRODUCT_005 | 400 | 통과 |
| PRD-018 | PRODUCT | 대표 이미지 변경 | 썸네일 변경 | PATCH | /api/seller/products/{id}/thumbnail | 이미지 존재 | 없음 | thumbnail 변경 | 200 | 통과 |
| PRD-019 | PRODUCT | 옵션 수정 | 정상 수정 | PATCH | /api/seller/items/{itemId} | SELLER APPROVED | price/stock | 수정 성공 | 200 | 통과 |
| PRD-020 | PRODUCT | 재고 제한 | 최대 재고 초과 | PATCH | /api/seller/items/{itemId} | > 99,999 | stock | INVENTORY_005 | 400 | 통과 |
| PRD-021 | PRODUCT | 판매자 상태 제한 | 반려 판매자 | PATCH | /api/seller/items/{itemId} | REJECTED | price | SELLER_003 | 403 | 통과 |
| PRD-022 | PRODUCT | 재고 입고 | 정상 입고 | POST | /api/seller/items/{itemId}/inbound | 승인 판매자 | quantity | 재고 증가 | 200 | 통과 |
| PRD-023 | PRODUCT | 입고 제한 | 재고 초과 | POST | /api/seller/items/{itemId}/inbound | 99,999 기준 | quantity | INVENTORY_003 | 400 | 통과 |
| PRD-024 | PRODUCT | 인기 상품 | Redis 기반 조회 | GET | /api/products/popular | ZSET 존재 | limit | rank list | 200 | 통과 |
| PRD-025 | PRODUCT | 인기 상품 fallback | Redis 장애 | GET | /api/products/popular | Redis fail | limit | DB fallback | 200 | 통과 |

## POINT

| NO. | 도메인 | 기능명 | 시나리오 | Method | Endpoint | 사전 조건 | Request | Response | 상태코드 | 통과여부 |
|------|--------|--------|----------|--------|----------|------------|----------|----------|----------|----------|
| PT001 | POINT | 포인트 충전 | 정상 충전 | POST | /api/users/me/points/charge | BUYER 로그인 | {"amount": 50000} | {"balance": 50000, "earnedAmount": 50000, "expireAt": "2027-06-19"} | 200 | 통과 |
| PT002 | POINT | 최소금액 미만 충전 실패 | 1,000원 미만 충전 | POST | /api/users/me/points/charge | BUYER 로그인 | {"amount": 999} | POINT_004 | 400 | 통과 |
| PT003 | POINT | 최대금액 초과 충전 실패 | 1,000,001원 충전 | POST | /api/users/me/points/charge | BUYER 로그인 | {"amount": 1000001} | POINT_004 | 400 | 통과 |
| PT004 | POINT | 주문 시 포인트 사용 | 포인트 사용 주문 | POST | /api/orders | 포인트 잔액 충분 | usedPoint 포함 | order + usedPoint 반영 | 201 | 통과 |
| PT005 | POINT | 잔액 초과 사용 실패 | 보유 포인트 초과 사용 | POST | /api/orders | 잔액 부족 | {"usedPoint": 999999} | POINT_002 | 400 | 통과 |
| PT006 | POINT | 주문 취소 포인트 환불 | 포인트 복구 | POST | /api/orders/{orderId}/cancel | 포인트 사용 주문 | {"reason":"변심"} | restoredPoint 반환 | 200 | 통과 |
| PT007 | POINT | 포인트 이력 조회 | 타입별 이력 조회 | GET | /api/users/me/points?type=CHARGE | 이력 존재 | 없음 | page + content | 200 | 통과 |
| PT008 | POINT | 포인트 잔액 조회 | 잔액 + 만료 예정 조회 | GET | /api/users/me/points | BUYER 로그인 | 없음 | balance + expiringSoon | 200 | 통과 |


## REVIEW

| NO. | 도메인 | 기능명 | 시나리오 | Method | Endpoint | 사전 조건 | Request | Response | 상태코드 | 통과여부 |
|------|--------|--------|----------|--------|----------|------------|----------|----------|----------|----------|
| REV001 | REVIEW | 리뷰 작성 - 정상 | 배송 완료 주문 리뷰 작성 | POST | /api/reviews | FINAL_DELIVERY + 본인 주문 | multipart + rating/content | 리뷰 생성 | 200 | 통과 |
| REV002 | REVIEW | 리뷰 작성 + 이미지 | 이미지 포함 리뷰 작성 | POST | /api/reviews | FINAL_DELIVERY + 본인 주문 | images 1~5장 | 이미지 포함 리뷰 | 200 | 통과 |
| REV003 | REVIEW | 리뷰 작성 - 주문 없음 | 존재하지 않는 orderItemId | POST | /api/reviews | invalid orderItemId | orderItemId=99 | ORDER_006 | 404 | 통과 |
| REV004 | REVIEW | 리뷰 작성 - 타인 주문 | 타인 주문 리뷰 작성 | POST | /api/reviews | 다른 사용자 주문 | orderItemId=1 | REVIEW_006 | 403 | 통과 |
| REV005 | REVIEW | 리뷰 작성 - 미배송 | SHIPPING 상태 리뷰 작성 | POST | /api/reviews | SHIPPING | orderItemId=1 | REVIEW_001 | 400 | 통과 |
| REV006 | REVIEW | 리뷰 작성 - 취소 주문 | CANCELLED 주문 리뷰 | POST | /api/reviews | CANCELLED | orderItemId=1 | REVIEW_001 | 400 | 통과 |
| REV007 | REVIEW | 리뷰 작성 - 거절 주문 | REJECTED 주문 리뷰 | POST | /api/reviews | REJECTED | orderItemId=1 | REVIEW_001 | 400 | 통과 |
| REV008 | REVIEW | 리뷰 중복 작성 | 동일 주문 재작성 | POST | /api/reviews | 기존 리뷰 존재 | orderItemId=1 | REVIEW_002 | 409 | 통과 |
| REV009 | REVIEW | 별점 경계값 | rating 1/5 허용 | POST | /api/reviews | 정상 주문 | rating=1/5 | 정상 생성 | 200 | 통과 |
| REV010 | REVIEW | 내용 최소 길이 | content 10자 | POST | /api/reviews | 정상 주문 | 10자 content | 정상 생성 | 200 | 통과 |
| REV011 | REVIEW | 내용 길이 초과 | 1001자 | POST | /api/reviews | 정상 주문 | 1001 chars | REVIEW_004 | 400 | 통과 |
| REV012 | REVIEW | 리뷰 수정 - 정상 | 30일 이내 수정 | PATCH | /api/reviews/{reviewId} | 본인 리뷰 | rating/content | 수정 성공 | 200 | 통과 |
| REV013 | REVIEW | 리뷰 수정 - 경계 | 30일 미만 | PATCH | /api/reviews/{reviewId} | 29일 23:59 | 수정 요청 | 수정 성공 | 200 | 통과 |
| REV014 | REVIEW | 리뷰 수정 - 만료 | 30일+1초 | PATCH | /api/reviews/{reviewId} | 30일 초과 | 수정 요청 | REVIEW_007 | 400 | 통과 |
| REV015 | REVIEW | 리뷰 수정 - 없음 | 존재하지 않는 리뷰 | PATCH | /api/reviews/{reviewId} | invalid reviewId | rating/content | REVIEW_005 | 404 | 통과 |
| REV016 | REVIEW | 리뷰 수정 - 타인 | 타인 리뷰 수정 | PATCH | /api/reviews/{reviewId} | 다른 사용자 리뷰 | rating/content | REVIEW_006 | 403 | 통과 |
| REV017 | REVIEW | 리뷰 삭제 | 정상 삭제 | DELETE | /api/reviews/{reviewId} | 본인 ACTIVE 리뷰 | 없음 | soft delete | 200 | 통과 |
| REV018 | REVIEW | 리뷰 삭제 - 타인 | 타인 리뷰 삭제 | DELETE | /api/reviews/{reviewId} | 다른 사용자 리뷰 | 없음 | REVIEW_006 | 403 | 통과 |
| REV019 | REVIEW | 리뷰 삭제 - 이미 삭제 | DELETED 리뷰 | DELETE | /api/reviews/{reviewId} | DELETED | 없음 | REVIEW_005 | 404 | 통과 |
| REV020 | REVIEW | 리뷰 삭제 - 없음 | 존재하지 않는 리뷰 | DELETE | /api/reviews/{reviewId} | invalid reviewId | 없음 | REVIEW_005 | 404 | 통과 |
| REV021 | REVIEW | 내 리뷰 목록 | 내 리뷰 조회 | GET | /api/users/me/reviews | 로그인 | 없음 | page content | 200 | 통과 |
| REV022 | REVIEW | 리뷰 가능 목록 | 작성 가능 주문 조회 | GET | /api/users/me/reviewable | 배송 완료 주문 | 없음 | list | 200 | 통과 |


## SECURITY 

| NO.     | 도메인    | 기능명                     | 시나리오                                   | Method | Endpoint                                                   | 사전 조건 | Request | Response | 상태코드 | 통과여부 |
|--------|----------|---------------------------|--------------------------------------------|--------|------------------------------------------------------------|-----------|----------|----------|----------|----------|
| SEC001 | SECURITY | IP Spoofing 방어           | X-Forwarded-For 조작으로 Rate Limit 우회 시도 | POST   | /api/auth/login                                           | trusted proxy 정책 적용 | 조작된 XFF Header | Rate Limit 우회 실패 | 429 | 통과 |
| SEC002 | SECURITY | Client IP 추출             | trusted proxy 환경에서 XFF 기반 실제 IP 추출 | POST   | 인증 API                                                  | Nginx/trusted proxy 설정 | XFF Header 포함 | 실제 client IP 추출 | 200 | 통과 |
| SEC003 | SECURITY | Client IP 추출             | 비신뢰 요청의 XFF 조작 무시                | POST   | 인증 API                                                  | remoteAddr 비신뢰 | 임의 XFF Header | 조작된 IP 미신뢰 | 200 | 통과 |
| SEC004 | SECURITY | JWT 인증                   | 유효한 Access Token 보호 API 접근 허용     | GET    | 보호 API                                                  | 유효한 AT | Authorization Header | API 접근 성공 | 200 | 통과 |
| SEC005 | SECURITY | JWT 인증                   | 만료/변조 Access Token 차단                | GET    | 보호 API                                                  | 만료 또는 변조 AT | Authorization Header | 인증 실패 | 401 | 통과 |
| SEC006 | SECURITY | Token Version              | 비밀번호 변경 후 기존 토큰 무효화          | GET    | 보호 API                                                  | tokenVersion 증가 | 기존 AT 사용 | 인증 실패 | 401 | 통과 |
| SEC007 | SECURITY | Token Version              | 권한 변경 후 기존 토큰 무효화              | GET    | 보호 API                                                  | 권한 변경으로 tokenVersion 증가 | 기존 AT 사용 | 인증 실패 | 401 | 통과 |
| SEC008 | SECURITY | 권한 검증                  | 일반 사용자의 ADMIN API 접근 차단          | GET    | /api/admin/**                                             | BUYER/SELLER 로그인 | 일반 사용자 AT | 접근 거부 | 403 | 통과 |
| SEC009 | SECURITY | 보안 헤더                  | 응답 보안 헤더 적용 확인                   | GET    | 전체 API                                                  | 서버 실행 | 일반 요청 | 보안 헤더 포함 | 200 | 통과 |
| SEC010 | SECURITY | 관리자 자기자신 정지 불가   | 관리자가 자신의 계정 정지 시도             | PATCH  | /api/admin/security/actions/{adminId}/suspend             | SUPER_ADMIN / 본인 ID | 없음 | 예외 반환 | 400 | 통과 |
| SEC011 | SECURITY | 관리자 정지 처리           | 상태변경 + 이력저장 + 전체 로그아웃        | PATCH  | /api/admin/security/actions/{adminId}/suspend             | SUPER_ADMIN / 다른 ADMIN | 없음 | SUSPENDED + 로그아웃 | 200 | 통과 |
| SEC012 | SECURITY | 재정지 처리                | 기존 이력 비활성화 + 민감값 마스킹         | PATCH  | /api/admin/security/actions/{adminId}/suspend             | 이미 SUSPENDED 관리자 | 없음 | 이력 비활성화 + 마스킹 | 200 | 통과 |
| SEC013 | SECURITY | IP 암호화 저장             | IP AES 암호화 + 해시 분리 저장             | SERVICE | SecurityAuditCryptoService                                 | 32바이트 키 | IP 문자열 | AES + Hash 저장 | - | 통과 |
| SEC014 | SECURITY | 감사로그 장애 내성         | 로그 실패 시 비즈니스 영향 없음            | EVENT  | SecurityAuditListener                                      | 로그 저장 실패 상황 | 이벤트 | 비즈니스 정상 진행 | - | 통과 |


## SEARCH
| NO.     | 도메인  | 기능명                         | 시나리오                                   | Method | Endpoint | 사전 조건 | Request | Response | 상태코드 | 통과여부 |
|--------|--------|--------------------------------|--------------------------------------------|--------|----------|-----------|----------|----------|----------|----------|
| SRH-001 | SEARCH | 상품 검색/목록                | 정상 keyword/카테고리 검색                | GET    | /api/products/search?keyword=노트북&categoryId=2&sort=LATEST&page=0&size=20 | - | keyword+category | 검색 결과 페이지 | 200 | 통과 |
| SRH-002 | SEARCH | 키워드 정규화                | 공백/특수문자 keyword 정규화              | GET    | /api/products/search | - | keyword=공백/특수 | 전체 목록 | 200 | 통과 |
| SRH-003 | SEARCH | 인기순 검색                  | POPULAR (키워드 없음)                    | GET    | /api/products/search?sort=POPULAR | Redis ZSET | 없음 | 인기 랭킹 리스트 | 200 | 통과 |
| SRH-004 | SEARCH | 인기순 + keyword            | keyword 존재 시 repository 검색 전환      | GET    | /api/products/search?keyword=의자&sort=POPULAR | - | keyword | 판매량 정렬 | 200 | 통과 |
| SRH-005 | SEARCH | 인기순 + 필터               | category/minPrice 존재 시 POPULAR 무시     | GET    | /api/products/search | - | filter params | repository 검색 | 200 | 통과 |
| SRH-006 | SEARCH | 인기순 페이징               | POPULAR page 이동                         | GET    | /api/products/search?sort=POPULAR&page=1&size=10 | 20+ 인기상품 | page | 11~20위 | 200 | 통과 |
| SRH-007 | SEARCH | 인기순 노출 필터            | 판매불가 상품 제외                         | GET    | /api/products/search?sort=POPULAR | - | 없음 | 노출가능 상품만 | 200 | 통과 |
| SRH-008 | SEARCH | 인기순 fallback             | Redis 장애 fallback                        | GET    | /api/products/search?sort=POPULAR | Redis fail | 없음 | DB fallback | 200 | 통과 |
| SRH-009 | SEARCH | 정렬 파라미터 변환          | sort invalid → LATEST                      | GET    | /api/products/search | - | sort=BOGUS | LATEST fallback | 200 | 통과 |
| SRH-010 | SEARCH | LATEST 정렬                 | 최신순 조회                               | GET    | /api/products/search | DB | - | 최신순 | 200 | 통과 |
| SRH-011 | SEARCH | PRICE 정렬                  | 가격 오름/내림                            | GET    | /api/products/search | DB | sort=PRICE_ASC/DESC | 정렬 결과 | 200 | 통과 |
| SRH-012 | SEARCH | POPULAR 정렬                | 판매량 정렬 + stable sort                 | GET    | /api/products/search | DB | sort=POPULAR | 인기순 | 200 | 통과 |
| SRH-013 | SEARCH | 가격 필터                   | min/max price                             | GET    | /api/products/search | DB | price range | 필터 결과 | 200 | 통과 |
| SRH-014 | SEARCH | 태그 검색                   | tag normalization                          | GET    | /api/products/search | DB | keyword tag | 매칭 결과 | 200 | 통과 |
| SRH-015 | SEARCH | 페이징 / N+1 방지           | paging + 옵션 fetch                       | GET    | /api/products/search | DB | page/size | page result | 200 | 통과 |
| SRH-016 | SEARCH | 연관상품                    | related products                           | GET    | /api/products/{id}/related | category | - | related list | 200 | 통과 |
| SRH-017 | SEARCH | FULLTEXT 검색               | 상품명 검색                               | GET    | /api/products/search | MySQL FT | keyword | 매칭 결과 | 200 | 통과 |
| SRH-018 | SEARCH | FULLTEXT OR 검색            | 설명/토큰 검색                            | GET    | /api/products/search | MySQL FT | keyword | OR match | 200 | 통과 |
| SRH-019 | SEARCH | FULLTEXT + filter           | AND + category filter                      | GET    | /api/products/search | MySQL FT | keyword+category | filtered | 200 | 통과 |
| SRH-020 | SEARCH | 인기 검색어                 | Redis keyword ranking                     | GET    | /api/search/popular-keywords | Redis | limit | rank list | 200 | 통과 |
| SRH-021 | SEARCH | 인기 태그 자동완성          | prefix search                              | GET    | /api/search/tags?keyword=NI | Redis/DB | prefix | tag list | 200 | 통과 |


## SUBSCRIPTION 

| NO.     | 도메인        | 기능명                 | 시나리오 설명 | Method | Endpoint | 사전 조건 | Request | Response | 상태코드 | 통과여부 |
|--------|--------------|------------------------|-------------|--------|----------|-----------|----------|----------|----------|----------|
| SUB001 | SUBSCRIPTION | 구독 가입              | 구독 신규 생성 (ACTIVE) | POST | /api/subscriptions | 유효 구독 없음 | 없음 | 구독 생성 정보 (ACTIVE) | 200 | 통과 |
| SUB002 | SUBSCRIPTION | 구독 가입 - 사용자 없음 | 탈퇴/삭제된 사용자 | POST | /api/subscriptions | userId 없음 | 없음 | MEMBER_001 | 404 | 통과 |
| SUB003 | SUBSCRIPTION | 구독 가입 - 중복       | ACTIVE/CANCELLED 존재 | POST | /api/subscriptions | 기존 구독 존재 | 없음 | SUBSCRIPTION_002 | 409 | 통과 |
| SUB004 | SUBSCRIPTION | 내 구독 조회           | 현재 유효 구독 조회 | GET | /api/subscriptions/me | ACTIVE/CANCELLED 존재 | 없음 | 구독 정보 + daysLeft | 200 | 통과 |
| SUB005 | SUBSCRIPTION | 내 구독 조회 - 없음    | 구독 이력 없음 | GET | /api/subscriptions/me | 구독 없음 | 없음 | SUBSCRIPTION_001 | 404 | 통과 |
| SUB006 | SUBSCRIPTION | 구독 해지              | 정상 해지 | PATCH | /api/subscriptions/cancel | ACTIVE 구독 | reason | CANCELLED 상태 | 200 | 통과 |
| SUB007 | SUBSCRIPTION | 구독 해지 - 없음       | 구독 존재하지 않음 | PATCH | /api/subscriptions/cancel | 유효 구독 없음 | reason | SUBSCRIPTION_001 | 404 | 통과 |
| SUB008 | SUBSCRIPTION | 구독 해지 - 중복       | 이미 해지 상태 | PATCH | /api/subscriptions/cancel | CANCELLED 상태 | reason | SUBSCRIPTION_003 | 400 | 통과 |


## USER

| NO.     | 도메인 | 기능명                 | 시나리오                                   | Method | Endpoint | 사전 조건 | Request | Response | 상태코드 | 통과여부 |
|--------|--------|------------------------|--------------------------------------------|--------|----------|-----------|----------|----------|----------|----------|
| USER001 | USER   | 비밀번호 변경          | 기존 비밀번호 일치 시 변경 성공           | PATCH  | /api/users/me/password | 로그인 사용자 | currentPassword, newPassword | 비밀번호 변경 성공 | 200 | 통과 |
| USER002 | USER   | 비밀번호 변경          | 기존 비밀번호 불일치 시 실패               | PATCH  | /api/users/me/password | 로그인 사용자 | wrong currentPassword | 비밀번호 변경 실패 | 400 | 통과 |
| USER003 | USER   | 회원 탈퇴              | 회원 탈퇴 시 비활성화 및 토큰 무효화       | DELETE | /api/users/me | 로그인 사용자 | Authorization Header | 탈퇴 성공 | 200 | 통과 |
| USER004 | USER   | 회원 상태              | 비활성 사용자 토큰 접근 차단               | GET    | 보호 API | 비활성 사용자 | 기존 AT | 인증 실패 | 401/403 | 통과 |
| USER005 | USER   | 관리자 권한 변경       | 사용자 권한 변경 성공                      | PATCH  | /api/admin/users/{userId}/role | ADMIN/SUPER_ADMIN | role 변경 요청 | 권한 변경 성공 | 200 | 통과 |
| USER006 | USER   | 관리자 권한 변경       | 일반 사용자의 권한 변경 차단               | PATCH  | /api/admin/users/{userId}/role | BUYER/SELLER | role 변경 요청 | 접근 거부 | 403 | 통과 |
| USER007 | USER   | 사용자 비활성화        | 관리자 사용자 비활성화                     | PATCH  | /api/admin/users/{userId}/status | ADMIN/SUPER_ADMIN | active=false | 상태 변경 성공 | 200 | 통과 |

