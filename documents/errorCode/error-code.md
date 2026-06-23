# 공통 에러 코드

> 본 문서는 서버가 내려주는 에러 응답의 `code` 값을 도메인 prefix별로 정리한 레퍼런스입니다.
> 모든 항목은 `com.sparta.one_stop.global.exception.ErrorCode` enum을 기준으로 작성되었으며, 코드와 메시지는 enum 정의와 1:1로 일치합니다.

---

## 에러 응답 형식

에러가 발생하면 서버는 아래 형식의 JSON을 반환합니다. (성공/페이징 응답 형식은 [API.md](../api/API.md#공통-응답-형식)를 참고하세요.)

```json
{
  "success": false,
  "status": 400,
  "code": "ORDER_002",
  "message": "재고가 부족합니다",
  "timestamp": "2025-05-12T14:30:00"
}
```

- `status` HTTP 상태 코드입니다.
- `code` 도메인 prefix와 번호로 구성된 식별자(`{도메인}_{번호}`)이며, 클라이언트는 이 값으로 에러를 분기합니다.
- `message` 사용자에게 노출 가능한 한국어 안내 문구입니다.

> 번호는 도메인별로 일반 비즈니스 에러(`001`~)와 엔티티 검증 에러(`020`~)를 구분해 부여합니다. 일부 prefix는 중간에 결번이 있으며, 본 문서는 실제 enum에 존재하는 코드만 그대로 반영합니다.

---

## 프리픽스 목록

| Prefix | 도메인 | 개수 |
|--------|--------|------|
| [COMMON](#common--공통) | 공통 | 10 |
| [AUTH](#auth--인증인가) | 인증/인가 | 20 |
| [MEMBER](#member--회원) | 회원 | 9 |
| [PRODUCT](#product--상품) | 상품 | 17 |
| [CATEGORY](#category--카테고리) | 카테고리 | 4 |
| [CART](#cart--장바구니) | 장바구니 | 12 |
| [ORDER](#order--주문) | 주문 | 45 |
| [PAYMENT](#payment--결제) | 결제 | 19 |
| [OUTBOX](#outbox--이벤트-발행) | 이벤트 발행 | 14 |
| [NOTIFICATION](#notification--알림) | 알림 | 5 |
| [POINT](#point--포인트) | 포인트 | 32 |
| [SHIPPING](#shipping--배송) | 배송 | 6 |
| [INVENTORY](#inventory--재고) | 재고 | 5 |
| [COUPON](#coupon--쿠폰) | 쿠폰 | 36 |
| [REVIEW](#review--리뷰) | 리뷰 | 8 |
| [SUBSCRIPTION](#subscription--구독) | 구독 | 10 |
| [SELLER](#seller--판매자) | 판매자 | 12 |
| [AI](#ai--ai-서비스) | AI 서비스 | 1 |
| [ADMIN](#admin--관리자) | 관리자 | 19 |
| [SECURITY](#security--보안-조치) | 보안 조치 | 7 |

---

## COMMON — 공통

| 코드 | HTTP | 메시지 |
|------|------|--------|
| COMMON_001 | 400 BAD_REQUEST | 입력값이 올바르지 않습니다 |
| COMMON_002 | 400 BAD_REQUEST | 요청 형식이 올바르지 않습니다 |
| COMMON_003 | 400 BAD_REQUEST | 지원하지 않는 HTTP 메서드입니다 |
| COMMON_004 | 400 BAD_REQUEST | 필수 파라미터가 누락되었습니다 |
| COMMON_005 | 400 BAD_REQUEST | 파일 크기가 제한을 초과했습니다 |
| COMMON_006 | 400 BAD_REQUEST | 지원하지 않는 파일 형식입니다 |
| COMMON_007 | 500 INTERNAL_SERVER_ERROR | 서버 내부 오류가 발생했습니다 |
| COMMON_008 | 503 SERVICE_UNAVAILABLE | 서비스가 일시적으로 이용 불가합니다 |
| COMMON_009 | 429 TOO_MANY_REQUESTS | 요청이 너무 많습니다. 잠시 후 다시 시도해주세요 |
| COMMON_010 | 400 BAD_REQUEST | 요청 파라미터 값이 올바르지 않습니다 |

## AUTH — 인증/인가

| 코드 | HTTP | 메시지 |
|------|------|--------|
| AUTH_001 | 400 BAD_REQUEST | 이메일 형식이 올바르지 않습니다 |
| AUTH_002 | 409 CONFLICT | 이미 가입된 이메일입니다 |
| AUTH_003 | 400 BAD_REQUEST | 비밀번호 규칙에 맞지 않습니다 |
| AUTH_004 | 401 UNAUTHORIZED | 이메일 또는 비밀번호가 일치하지 않습니다 |
| AUTH_005 | 403 FORBIDDEN | 정지된 계정입니다. 고객센터에 문의해주세요 |
| AUTH_006 | 403 FORBIDDEN | 탈퇴한 계정입니다 |
| AUTH_007 | 401 UNAUTHORIZED | 로그인이 필요합니다 |
| AUTH_008 | 401 UNAUTHORIZED | 인증 토큰이 만료되었습니다 |
| AUTH_009 | 401 UNAUTHORIZED | 유효하지 않은 인증 토큰입니다 |
| AUTH_010 | 401 UNAUTHORIZED | 갱신 토큰이 만료되었습니다. 다시 로그인해주세요 |
| AUTH_011 | 403 FORBIDDEN | 접근 권한이 없습니다 |
| AUTH_012 | 401 UNAUTHORIZED | 다른 기기에서 로그인되어 세션이 만료되었습니다 |
| AUTH_013 | 429 TOO_MANY_REQUESTS | 너무 많은 요청입니다. 잠시 후 다시 시도해주세요 |
| AUTH_014 | 401 UNAUTHORIZED | 최대 기기 수를 초과했습니다. 다른 기기에서 로그아웃해주세요 |
| AUTH_015 | 401 UNAUTHORIZED | 다른 기기에서 로그인되어 자동으로 로그아웃되었습니다 |
| AUTH_016 | 409 CONFLICT | 이미 OAuth2 계정이 연결되어 있습니다 |
| AUTH_017 | 400 BAD_REQUEST | 지원하지 않는 OAuth2 제공자입니다 |
| AUTH_018 | 401 UNAUTHORIZED | OAuth2 인증 처리에 실패했습니다 |
| AUTH_019 | 409 CONFLICT | 이미 다른 방법으로 가입된 이메일입니다 |
| AUTH_020 | 401 UNAUTHORIZED | 기기 인증 정보가 일치하지 않습니다. 다시 로그인해주세요 |

## MEMBER — 회원

| 코드 | HTTP | 메시지 |
|------|------|--------|
| MEMBER_001 | 404 NOT_FOUND | 회원 정보를 찾을 수 없습니다 |
| MEMBER_002 | 400 BAD_REQUEST | 현재 비밀번호가 일치하지 않습니다 |
| MEMBER_003 | 400 BAD_REQUEST | 새 비밀번호는 기존 비밀번호와 다르게 설정해주세요 |
| MEMBER_004 | 404 NOT_FOUND | 배송지 정보를 찾을 수 없습니다 |
| MEMBER_005 | 400 BAD_REQUEST | 기본 배송지는 삭제할 수 없습니다 |
| MEMBER_006 | 400 BAD_REQUEST | 배송지는 최대 10개까지 등록할 수 있습니다 |
| MEMBER_010 | 400 BAD_REQUEST | 이미 탈퇴한 회원입니다 |
| MEMBER_011 | 400 BAD_REQUEST | 탈퇴한 회원은 처리할 수 없습니다 |
| MEMBER_012 | 400 BAD_REQUEST | 정지 상태가 아닌 회원입니다 |

## PRODUCT — 상품

| 코드 | HTTP | 메시지 |
|------|------|--------|
| PRODUCT_001 | 404 NOT_FOUND | 상품을 찾을 수 없습니다 |
| PRODUCT_002 | 400 BAD_REQUEST | 현재 판매하지 않는 상품입니다 |
| PRODUCT_003 | 400 BAD_REQUEST | 판매 가격은 100원 이상이어야 합니다 |
| PRODUCT_004 | 400 BAD_REQUEST | 할인가는 정가보다 낮아야 합니다 |
| PRODUCT_005 | 400 BAD_REQUEST | 상품 이미지는 최소 1장 이상 필요합니다 |
| PRODUCT_006 | 400 BAD_REQUEST | 상품 이미지는 최대 10장까지 등록할 수 있습니다 |
| PRODUCT_007 | 404 NOT_FOUND | 카테고리를 찾을 수 없습니다 |
| PRODUCT_008 | 403 FORBIDDEN | 다른 판매자의 상품은 수정할 수 없습니다 |
| PRODUCT_009 | 400 BAD_REQUEST | 주문 진행 중인 상품은 삭제할 수 없습니다 |
| PRODUCT_010 | 400 BAD_REQUEST | 현재 상태에서는 수정할 수 없습니다 |
| PRODUCT_011 | 404 NOT_FOUND | 상품 이미지를 찾을 수 없습니다 |
| PRODUCT_012 | 400 BAD_REQUEST | 상품 카테고리는 최소 1개 이상 매핑되어야 합니다 |
| PRODUCT_013 | 400 BAD_REQUEST | 상품 카테고리는 최대 3개까지 매핑할 수 있습니다 |
| PRODUCT_014 | 400 BAD_REQUEST | 조회 페이지 크기가 허용 범위를 벗어났습니다 |
| PRODUCT_015 | 400 BAD_REQUEST | 가격 범위가 올바르지 않습니다 |
| PRODUCT_016 | 400 BAD_REQUEST | 상품 옵션 조합이 중복됩니다 |
| PRODUCT_017 | 500 INTERNAL_SERVER_ERROR | 상품 반려 이력을 찾을 수 없습니다 |

## CATEGORY — 카테고리

| 코드 | HTTP | 메시지 |
|------|------|--------|
| CATEGORY_001 | 404 NOT_FOUND | 카테고리를 찾을 수 없습니다 |
| CATEGORY_002 | 409 CONFLICT | 동일 부모 아래 중복된 카테고리명입니다 |
| CATEGORY_003 | 400 BAD_REQUEST | 카테고리 깊이는 최대 3단계입니다 |
| CATEGORY_004 | 400 BAD_REQUEST | 상품이 매핑된 카테고리는 삭제할 수 없습니다 |

## CART — 장바구니

| 코드 | HTTP | 메시지 |
|------|------|--------|
| CART_001 | 400 BAD_REQUEST | 장바구니에 담을 수 없는 상품입니다 |
| CART_002 | 400 BAD_REQUEST | 수량은 1개 이상 99개 이하여야 합니다 |
| CART_003 | 400 BAD_REQUEST | 장바구니에 담을 수 있는 상품은 최대 50개입니다 |
| CART_004 | 404 NOT_FOUND | 장바구니에 해당 상품이 없습니다 |
| CART_005 | 400 BAD_REQUEST | 본인의 상품은 장바구니에 담을 수 없습니다 |
| CART_006 | 403 FORBIDDEN | 본인의 장바구니만 접근할 수 있습니다 |
| CART_020 | 400 BAD_REQUEST | 장바구니 소유자는 필수입니다 |
| CART_021 | 400 BAD_REQUEST | 장바구니 정보는 필수입니다 |
| CART_022 | 400 BAD_REQUEST | 상품 옵션 정보는 필수입니다 |
| CART_023 | 400 BAD_REQUEST | 수량은 1 이상이어야 합니다 |
| CART_024 | 400 BAD_REQUEST | 장바구니 최대 수량은 99개입니다 |
| CART_025 | 400 BAD_REQUEST | 수량은 1 미만이 될 수 없습니다 |

## ORDER — 주문

| 코드 | HTTP | 메시지 |
|------|------|--------|
| ORDER_001 | 400 BAD_REQUEST | 주문할 상품이 없습니다 |
| ORDER_002 | 400 BAD_REQUEST | 재고가 부족합니다 |
| ORDER_003 | 400 BAD_REQUEST | 판매하지 않는 상품이 포함되어 있습니다 |
| ORDER_004 | 404 NOT_FOUND | 배송지 정보를 찾을 수 없습니다 |
| ORDER_005 | 400 BAD_REQUEST | 사용할 수 없는 쿠폰입니다 |
| ORDER_006 | 404 NOT_FOUND | 주문 정보를 찾을 수 없습니다 |
| ORDER_007 | 403 FORBIDDEN | 본인의 주문만 조회할 수 있습니다 |
| ORDER_008 | 400 BAD_REQUEST | 현재 상태에서는 취소할 수 없습니다 |
| ORDER_009 | 409 CONFLICT | 주문 상태가 변경되어 처리할 수 없습니다 |
| ORDER_010 | 400 BAD_REQUEST | 최소 주문 금액은 1,000원입니다 |
| ORDER_011 | 400 BAD_REQUEST | 미승인 판매자의 상품은 주문할 수 없습니다 |
| ORDER_012 | 400 BAD_REQUEST | 쿠폰 할인 금액과 사용 포인트의 합은 상품 금액을 초과할 수 없습니다 |
| ORDER_013 | 409 CONFLICT | 주문 처리 중입니다. 잠시 후 다시 시도해주세요. |
| ORDER_020 | 400 BAD_REQUEST | 주문 정보는 필수입니다 |
| ORDER_021 | 400 BAD_REQUEST | 상품 옵션 정보는 필수입니다 |
| ORDER_022 | 400 BAD_REQUEST | 판매자 정보는 필수입니다 |
| ORDER_023 | 400 BAD_REQUEST | 상품명은 필수입니다 |
| ORDER_024 | 400 BAD_REQUEST | 수량은 1 이상이어야 합니다 |
| ORDER_025 | 400 BAD_REQUEST | 가격은 0원 이상이어야 합니다 |
| ORDER_026 | 400 BAD_REQUEST | 주문 접수는 결제 대기 상태에서만 처리할 수 있습니다 |
| ORDER_027 | 400 BAD_REQUEST | 주문 확정은 주문 접수 상태에서만 가능합니다 |
| ORDER_028 | 400 BAD_REQUEST | 배송 시작은 주문 확정 상태에서만 가능합니다 |
| ORDER_029 | 400 BAD_REQUEST | 배송 완료는 배송 중 상태에서만 가능합니다 |
| ORDER_030 | 400 BAD_REQUEST | 주문 거절은 주문 접수 상태에서만 가능합니다 |
| ORDER_031 | 400 BAD_REQUEST | 현재 상태에서는 주문 상품을 취소할 수 없습니다 |
| ORDER_032 | 400 BAD_REQUEST | 처리 주체 유형은 필수입니다 |
| ORDER_033 | 400 BAD_REQUEST | 취소/거절 유형은 필수입니다 |
| ORDER_034 | 400 BAD_REQUEST | 처리자 ID는 필수입니다 |
| ORDER_035 | 400 BAD_REQUEST | 해당 처리 주체는 actorId를 가질 수 없습니다 |
| ORDER_036 | 400 BAD_REQUEST | 처리자 ID는 1 이상이어야 합니다 |
| ORDER_037 | 400 BAD_REQUEST | 취소/거절 금액은 0원 이상이어야 합니다 |
| ORDER_038 | 400 BAD_REQUEST | 복구 포인트는 0 이상이어야 합니다 |
| ORDER_039 | 400 BAD_REQUEST | 주문자는 필수입니다 |
| ORDER_040 | 400 BAD_REQUEST | 총 주문 금액은 0원 이상이어야 합니다 |
| ORDER_041 | 400 BAD_REQUEST | 총 할인 금액은 0원 이상이어야 합니다 |
| ORDER_042 | 400 BAD_REQUEST | 최종 결제 금액은 0원 이상이어야 합니다 |
| ORDER_043 | 400 BAD_REQUEST | 사용 포인트는 0 이상이어야 합니다 |
| ORDER_044 | 400 BAD_REQUEST | 구독 할인 금액은 0원 이상이어야 합니다 |
| ORDER_045 | 400 BAD_REQUEST | 수령인 이름은 필수입니다 |
| ORDER_046 | 400 BAD_REQUEST | 수령인 연락처는 필수입니다 |
| ORDER_047 | 400 BAD_REQUEST | 배송 주소는 필수입니다 |
| ORDER_048 | 400 BAD_REQUEST | 배송비는 0원 이상이어야 합니다 |
| ORDER_049 | 400 BAD_REQUEST | 주문 유형은 필수입니다 |
| ORDER_050 | 400 BAD_REQUEST | 결제 대기 상태에서만 결제 완료 처리할 수 있습니다 |
| ORDER_051 | 400 BAD_REQUEST | 이미 취소된 주문입니다 |

## PAYMENT — 결제

| 코드 | HTTP | 메시지 |
|------|------|--------|
| PAYMENT_001 | 400 BAD_REQUEST | 이미 결제가 완료된 주문입니다 |
| PAYMENT_002 | 400 BAD_REQUEST | 결제 금액이 주문 금액과 일치하지 않습니다 |
| PAYMENT_003 | 409 CONFLICT | 이미 처리된 결제 요청입니다 |
| PAYMENT_004 | 502 BAD_GATEWAY | 결제 처리 중 오류가 발생했습니다 |
| PAYMENT_005 | 400 BAD_REQUEST | 취소할 수 없는 결제입니다 |
| PAYMENT_006 | 400 BAD_REQUEST | 결제 대기 시간이 초과되었습니다 |
| PAYMENT_007 | 400 BAD_REQUEST | 지원하지 않는 결제 수단입니다 |
| PAYMENT_008 | 400 BAD_REQUEST | 취소된 주문은 결제할 수 없습니다 |
| PAYMENT_009 | 404 NOT_FOUND | 결제 정보를 찾을 수 없습니다 |
| PAYMENT_010 | 409 CONFLICT | 결제 처리 중입니다. 잠시 후 다시 시도해주세요. |
| PAYMENT_011 | 400 BAD_REQUEST | 결제 대기 상태의 주문만 결제할 수 있습니다. |
| PAYMENT_020 | 400 BAD_REQUEST | 주문 정보는 필수입니다 |
| PAYMENT_021 | 400 BAD_REQUEST | 결제 키는 필수입니다 |
| PAYMENT_022 | 400 BAD_REQUEST | 결제 금액은 0원 이상이어야 합니다 |
| PAYMENT_023 | 400 BAD_REQUEST | 결제 수단은 필수입니다 |
| PAYMENT_024 | 400 BAD_REQUEST | 결제 대기 상태에서만 승인할 수 있습니다 |
| PAYMENT_025 | 400 BAD_REQUEST | 결제 대기 상태에서만 실패 처리할 수 있습니다 |
| PAYMENT_026 | 400 BAD_REQUEST | 이미 취소된 결제입니다 |
| PAYMENT_027 | 400 BAD_REQUEST | 결제 완료 상태에서만 취소할 수 있습니다 |

## OUTBOX — 이벤트 발행

| 코드 | HTTP | 메시지 |
|------|------|--------|
| OUTBOX_001 | 409 CONFLICT | 이미 저장된 Outbox 이벤트입니다 |
| OUTBOX_002 | 404 NOT_FOUND | Outbox 이벤트를 찾을 수 없습니다 |
| OUTBOX_003 | 400 BAD_REQUEST | 처리할 수 없는 Outbox 이벤트 상태입니다 |
| OUTBOX_020 | 400 BAD_REQUEST | 이벤트 ID는 필수입니다 |
| OUTBOX_021 | 400 BAD_REQUEST | 이벤트 타입은 필수입니다 |
| OUTBOX_022 | 400 BAD_REQUEST | Aggregate Type은 필수입니다 |
| OUTBOX_023 | 400 BAD_REQUEST | Aggregate ID는 필수입니다 |
| OUTBOX_024 | 400 BAD_REQUEST | Kafka Topic은 필수입니다 |
| OUTBOX_025 | 400 BAD_REQUEST | Partition Key는 필수입니다 |
| OUTBOX_026 | 400 BAD_REQUEST | 이벤트 Payload는 필수입니다 |
| OUTBOX_027 | 400 BAD_REQUEST | PENDING 상태의 이벤트만 PROCESSING 처리할 수 있습니다 |
| OUTBOX_028 | 400 BAD_REQUEST | PROCESSING 상태의 이벤트만 PUBLISHED 처리할 수 있습니다 |
| OUTBOX_029 | 400 BAD_REQUEST | PROCESSING 상태의 이벤트만 실패 처리할 수 있습니다 |
| OUTBOX_030 | 400 BAD_REQUEST | 최대 재시도 횟수는 0 이상이어야 합니다 |

## NOTIFICATION — 알림

| 코드 | HTTP | 메시지 |
|------|------|--------|
| NOTIFICATION_020 | 400 BAD_REQUEST | 알림 대상 사용자는 필수입니다 |
| NOTIFICATION_021 | 400 BAD_REQUEST | 이벤트 ID는 필수입니다 |
| NOTIFICATION_022 | 400 BAD_REQUEST | 알림 유형은 필수입니다 |
| NOTIFICATION_023 | 400 BAD_REQUEST | 알림 제목은 필수입니다 |
| NOTIFICATION_024 | 400 BAD_REQUEST | 알림 내용은 필수입니다 |

## POINT — 포인트

| 코드 | HTTP | 메시지 |
|------|------|--------|
| POINT_001 | 404 NOT_FOUND | 포인트 계정을 찾을 수 없습니다 |
| POINT_002 | 400 BAD_REQUEST | 포인트가 부족합니다 |
| POINT_003 | 400 BAD_REQUEST | 포인트 금액은 1 이상이어야 합니다 |
| POINT_004 | 400 BAD_REQUEST | 포인트 충전 금액은 1,000원 이상 1,000,000원 이하여야 합니다 |
| POINT_005 | 409 CONFLICT | 포인트 처리 중 충돌이 발생했습니다. 다시 시도해주세요 |
| POINT_006 | 400 BAD_REQUEST | 적립률은 null일 수 없습니다. |
| POINT_007 | 400 BAD_REQUEST | 적립률은 음수일 수 없습니다. |
| POINT_008 | 400 BAD_REQUEST | 적립률은 100%를 초과할 수 없습니다. |
| POINT_009 | 400 BAD_REQUEST | 안전상한선 초과 |
| POINT_010 | 400 BAD_REQUEST | 해시 입력값 오류 |
| POINT_020 | 400 BAD_REQUEST | 포인트 소유자는 필수입니다 |
| POINT_021 | 400 BAD_REQUEST | 포인트 소유자 ID는 필수입니다 |
| POINT_022 | 400 BAD_REQUEST | 포인트 계정은 필수입니다 |
| POINT_023 | 400 BAD_REQUEST | 포인트 사용자는 필수입니다 |
| POINT_024 | 400 BAD_REQUEST | 포인트 변동 금액은 0일 수 없습니다 |
| POINT_025 | 400 BAD_REQUEST | 잔여 포인트는 0 이상이어야 합니다 |
| POINT_026 | 400 BAD_REQUEST | 포인트 이력 유형은 필수입니다 |
| POINT_027 | 400 BAD_REQUEST | 적립/충전/복구 금액은 양수여야 합니다 |
| POINT_028 | 400 BAD_REQUEST | 생성 시 잔여 포인트는 변동 금액과 같아야 합니다 |
| POINT_029 | 400 BAD_REQUEST | 포인트 만료일은 필수입니다 |
| POINT_030 | 400 BAD_REQUEST | 사용/만료 금액은 음수여야 합니다 |
| POINT_031 | 400 BAD_REQUEST | 사용/만료 이력의 잔여 포인트는 0이어야 합니다 |
| POINT_032 | 400 BAD_REQUEST | 차감 가능한 포인트 이력이 아닙니다 |
| POINT_033 | 400 BAD_REQUEST | 잔여 포인트가 부족합니다 |
| POINT_034 | 400 BAD_REQUEST | 만료 가능한 포인트 이력이 아닙니다 |
| POINT_035 | 400 BAD_REQUEST | 포인트 사용 이력은 필수입니다 |
| POINT_036 | 400 BAD_REQUEST | 차감 대상 포인트 이력은 필수입니다 |
| POINT_037 | 400 BAD_REQUEST | USE 이력만 사용 상세 이력과 연결할 수 있습니다 |
| POINT_038 | 400 BAD_REQUEST | 차감 대상은 CHARGE/EARN/REFUND 이력이어야 합니다 |
| POINT_039 | 400 BAD_REQUEST | 차감 금액은 1 이상이어야 합니다 |
| POINT_040 | 400 BAD_REQUEST | 차감 금액은 원본 이력의 잔여 포인트를 초과할 수 없습니다 |
| POINT_041 | 400 BAD_REQUEST | 원본 포인트 만료일은 필수입니다 |

## SHIPPING — 배송

| 코드 | HTTP | 메시지 |
|------|------|--------|
| SHIPPING_001 | 400 BAD_REQUEST | 주문 확인 전에는 배송을 시작할 수 없습니다 |
| SHIPPING_002 | 400 BAD_REQUEST | 유효하지 않은 배송 상태 변경입니다 |
| SHIPPING_003 | 400 BAD_REQUEST | 송장 번호는 필수 입력값입니다 |
| SHIPPING_004 | 400 BAD_REQUEST | 지원하지 않는 택배사입니다 |
| SHIPPING_005 | 404 NOT_FOUND | 배송 정보를 찾을 수 없습니다 |
| SHIPPING_006 | 403 FORBIDDEN | 다른 판매자의 배송은 관리할 수 없습니다 |

## INVENTORY — 재고

| 코드 | HTTP | 메시지 |
|------|------|--------|
| INVENTORY_001 | 400 BAD_REQUEST | 재고가 부족합니다 |
| INVENTORY_002 | 400 BAD_REQUEST | 입고 수량은 1개 이상이어야 합니다 |
| INVENTORY_003 | 400 BAD_REQUEST | 입고 수량이 최대 한도를 초과합니다 |
| INVENTORY_004 | 404 NOT_FOUND | 재고 정보를 찾을 수 없습니다 |
| INVENTORY_005 | 400 BAD_REQUEST | 보정 재고가 최대 한도를 초과합니다 |

## COUPON — 쿠폰

| 코드 | HTTP | 메시지 |
|------|------|--------|
| COUPON_001 | 400 BAD_REQUEST | 쿠폰이 모두 소진되었습니다 |
| COUPON_002 | 409 CONFLICT | 이미 다운로드한 쿠폰입니다 |
| COUPON_003 | 400 BAD_REQUEST | 쿠폰 발급 기간이 아닙니다 |
| COUPON_004 | 404 NOT_FOUND | 쿠폰을 찾을 수 없습니다 |
| COUPON_005 | 400 BAD_REQUEST | 이 쿠폰은 최소 주문 금액 미달입니다 |
| COUPON_006 | 400 BAD_REQUEST | 이미 사용한 쿠폰입니다 |
| COUPON_007 | 400 BAD_REQUEST | 만료된 쿠폰입니다 |
| COUPON_008 | 400 BAD_REQUEST | 할인 값이 유효하지 않습니다 |
| COUPON_009 | 500 INTERNAL_SERVER_ERROR | 지원하지 않는 쿠폰 발급 전략입니다. |
| COUPON_010 | 400 BAD_REQUEST | 이미 비활성화된 쿠폰입니다 |
| COUPON_011 | 404 NOT_FOUND | 사용자 쿠폰을 찾을 수 없습니다 |
| COUPON_012 | 503 SERVICE_UNAVAILABLE | 쿠폰 발급 처리 중 일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요 |
| COUPON_020 | 400 BAD_REQUEST | 쿠폰명은 필수입니다 |
| COUPON_021 | 400 BAD_REQUEST | 쿠폰 할인 타입은 필수입니다 |
| COUPON_022 | 400 BAD_REQUEST | 쿠폰 할인 값은 1 이상이어야 합니다 |
| COUPON_023 | 400 BAD_REQUEST | 최소 주문 금액은 0원 이상이어야 합니다 |
| COUPON_024 | 400 BAD_REQUEST | 쿠폰 발급 가능 총 수량은 1 이상이어야 합니다 |
| COUPON_025 | 400 BAD_REQUEST | 쿠폰 시작일은 필수입니다 |
| COUPON_026 | 400 BAD_REQUEST | 쿠폰 만료일은 필수입니다 |
| COUPON_027 | 400 BAD_REQUEST | 쿠폰 만료일은 시작일 이후여야 합니다 |
| COUPON_028 | 400 BAD_REQUEST | 정률 쿠폰 할인율은 100 이하이어야 합니다 |
| COUPON_029 | 400 BAD_REQUEST | 정률 쿠폰은 최대 할인 금액이 필수입니다 |
| COUPON_030 | 400 BAD_REQUEST | 정액 쿠폰은 최대 할인 금액을 사용할 수 없습니다 |
| COUPON_031 | 400 BAD_REQUEST | 주문 금액은 0원 이상이어야 합니다 |
| COUPON_032 | 400 BAD_REQUEST | 현재 시각은 필수입니다 |
| COUPON_033 | 400 BAD_REQUEST | 비활성 쿠폰은 발급할 수 없습니다 |
| COUPON_034 | 400 BAD_REQUEST | 쿠폰 소유자는 필수입니다 |
| COUPON_035 | 400 BAD_REQUEST | 쿠폰 정보는 필수입니다 |
| COUPON_036 | 400 BAD_REQUEST | 쿠폰을 사용할 주문 정보는 필수입니다 |
| COUPON_037 | 400 BAD_REQUEST | 쿠폰 사용 일시는 필수입니다 |
| COUPON_038 | 400 BAD_REQUEST | 사용 가능한 쿠폰이 아닙니다 |
| COUPON_039 | 400 BAD_REQUEST | 사용 완료된 쿠폰만 복구할 수 있습니다 |
| COUPON_040 | 400 BAD_REQUEST | 이미 사용된 쿠폰은 만료 처리할 수 없습니다 |
| COUPON_041 | 400 BAD_REQUEST | 아직 만료되지 않은 쿠폰입니다 |
| COUPON_042 | 400 BAD_REQUEST | 사용자 ID는 필수입니다 |
| COUPON_043 | 403 FORBIDDEN | 본인 쿠폰만 사용할 수 있습니다 |

## REVIEW — 리뷰

| 코드 | HTTP | 메시지 |
|------|------|--------|
| REVIEW_001 | 400 BAD_REQUEST | 배송 완료 후에만 리뷰를 작성할 수 있습니다 |
| REVIEW_002 | 409 CONFLICT | 이미 리뷰를 작성한 주문 상품입니다 |
| REVIEW_003 | 400 BAD_REQUEST | 별점은 1점에서 5점 사이여야 합니다 |
| REVIEW_004 | 400 BAD_REQUEST | 리뷰 내용은 10자 이상 1,000자 이하여야 합니다 |
| REVIEW_005 | 404 NOT_FOUND | 리뷰를 찾을 수 없습니다 |
| REVIEW_006 | 403 FORBIDDEN | 본인의 리뷰만 수정/삭제할 수 있습니다 |
| REVIEW_007 | 400 BAD_REQUEST | 작성 후 30일이 지나 수정할 수 없습니다 |
| REVIEW_008 | 400 BAD_REQUEST | 리뷰 이미지는 최대 5개까지 등록할 수 있습니다 |

## SUBSCRIPTION — 구독

| 코드 | HTTP | 메시지 |
|------|------|--------|
| SUBSCRIPTION_001 | 404 NOT_FOUND | 구독 정보를 찾을 수 없습니다 |
| SUBSCRIPTION_002 | 409 CONFLICT | 이미 유효한 구독이 존재합니다 |
| SUBSCRIPTION_003 | 400 BAD_REQUEST | 구독이 이미 해지되었습니다 |
| SUBSCRIPTION_004 | 400 BAD_REQUEST | 현재 상태에서는 구독을 변경할 수 없습니다 |
| SUBSCRIPTION_005 | 400 BAD_REQUEST | 자동 결제에 실패했습니다 |
| SUBSCRIPTION_007 | 403 FORBIDDEN | 본인의 구독만 조회할 수 있습니다 |
| SUBSCRIPTION_008 | 403 FORBIDDEN | 본인의 구독만 수정할 수 있습니다 |
| SUBSCRIPTION_009 | 400 BAD_REQUEST | 다음 결제일이 유효하지 않습니다 |
| SUBSCRIPTION_010 | 400 BAD_REQUEST | 구독 상태가 변경되어 처리할 수 없습니다 |
| SUBSCRIPTION_011 | 400 BAD_REQUEST | 이미 만료된 구독입니다 |

## SELLER — 판매자

| 코드 | HTTP | 메시지 |
|------|------|--------|
| SELLER_001 | 404 NOT_FOUND | 판매자 정보를 찾을 수 없습니다 |
| SELLER_002 | 409 CONFLICT | 이미 입점 신청이 완료되었습니다 |
| SELLER_003 | 403 FORBIDDEN | 판매자 승인이 필요합니다 |
| SELLER_004 | 403 FORBIDDEN | 정지된 판매자 계정입니다 |
| SELLER_005 | 400 BAD_REQUEST | 사업자등록번호 형식이 올바르지 않습니다 |
| SELLER_006 | 409 CONFLICT | 이미 등록된 사업자등록번호입니다 |
| SELLER_007 | 403 FORBIDDEN | 해당 주문 상품의 판매자가 아닙니다 |
| SELLER_008 | 400 BAD_REQUEST | 이미 처리된 주문입니다 |
| SELLER_010 | 400 BAD_REQUEST | 판매자 상호명은 필수입니다 |
| SELLER_011 | 400 BAD_REQUEST | 사업자 등록번호는 필수입니다 |
| SELLER_012 | 400 BAD_REQUEST | 은행명은 필수입니다 |
| SELLER_013 | 400 BAD_REQUEST | 계좌번호는 필수입니다 |

## AI — AI 서비스

| 코드 | HTTP | 메시지 |
|------|------|--------|
| AI_001 | 503 SERVICE_UNAVAILABLE | AI 서비스가 일시적으로 이용 불가합니다. 잠시 후 다시 시도해주세요 |

## ADMIN — 관리자

| 코드 | HTTP | 메시지 |
|------|------|--------|
| ADMIN_001 | 403 FORBIDDEN | 관리자 권한이 필요합니다 |
| ADMIN_002 | 400 BAD_REQUEST | 이미 승인된 판매자입니다 |
| ADMIN_003 | 400 BAD_REQUEST | 이미 거절된 판매자입니다 |
| ADMIN_004 | 400 BAD_REQUEST | 관리자 계정은 정지할 수 없습니다 |
| ADMIN_005 | 400 BAD_REQUEST | 하위 카테고리가 존재하여 삭제할 수 없습니다 |
| ADMIN_006 | 400 BAD_REQUEST | 상품이 등록된 카테고리는 삭제할 수 없습니다 |
| ADMIN_007 | 400 BAD_REQUEST | 이미 강제 비활성화된 상품입니다 |
| ADMIN_008 | 400 BAD_REQUEST | 이미 승인된 상품입니다 |
| ADMIN_009 | 400 BAD_REQUEST | 이미 반려된 상품입니다 |
| ADMIN_010 | 400 BAD_REQUEST | 정지 상태인 판매자가 아닙니다 |
| ADMIN_011 | 400 BAD_REQUEST | 판매자 계정에는 관리자 권한을 부여할 수 없습니다 |
| ADMIN_012 | 400 BAD_REQUEST | 이미 관리자 권한이 있는 계정입니다 |
| ADMIN_013 | 403 FORBIDDEN | SUPER_ADMIN 계정의 권한은 회수할 수 없습니다 |
| ADMIN_014 | 403 FORBIDDEN | 본인 계정의 권한은 회수할 수 없습니다 |
| ADMIN_015 | 400 BAD_REQUEST | 관리자 권한이 없는 계정입니다 |
| ADMIN_016 | 400 BAD_REQUEST | 비활성(정지/탈퇴) 상태의 회원은 관리자로 승격할 수 없습니다. |
| ADMIN_017 | 400 BAD_REQUEST | 승인 요청 상태의 상품만 승인/반려할 수 있습니다 |
| ADMIN_018 | 400 BAD_REQUEST | 승인 대기 상태인 판매자만 승인/반려할 수 있습니다 |
| ADMIN_019 | 400 BAD_REQUEST | 이미 정지된 판매자입니다 |

## SECURITY — 보안 조치

| 코드 | HTTP | 메시지 |
|------|------|--------|
| SECURITY_001 | 400 BAD_REQUEST | 지원하지 않는 보안 조치입니다 |
| SECURITY_002 | 404 NOT_FOUND | 조치 대상 사용자를 찾을 수 없습니다 |
| SECURITY_003 | 403 FORBIDDEN | 보안 조치 권한이 없습니다 |
| SECURITY_004 | 400 BAD_REQUEST | 자기 자신은 제재할 수 없습니다 |
| SECURITY_005 | 400 BAD_REQUEST | 조치 사유에 민감정보를 포함할 수 없습니다 |
| SECURITY_006 | 403 FORBIDDEN | 해당 역할의 사용자는 보안 제재 대상이 아닙니다 |
| SECURITY_007 | 409 CONFLICT | 현재 사용자 상태에서는 요청한 보안 조치를 수행할 수 없습니다 |
