## 상품

- 상품 상태:
    - APPROVE_REQUESTED
    - APPROVED
    - REJECTED
    - DISCONTINUED
    - FORCE_INACTIVE
- APPROVE_REQUESTED 상태에서만 상품 승인/반려 가능
- REJECTED 상품 수정 시 APPROVE_REQUESTED 상태로 변경 후 재승인 요청 가능
- 구매자에게는 APPROVED 상품 중 판매중(ON_SALE) 옵션이 1개 이상인 상품만 노출 (전 옵션이 STOP인 상품은 0원·빈 옵션 노출 방지를 위해 비노출)
- 승인 후 상품 수정은 즉시 반영
    - 가격·재고(옵션) 변경 → 즉시 반영 (재승인 불필요)
    - 이름·설명·이미지·카테고리 값이 실제로 변경된 경우에만 APPROVE_REQUESTED로 전환되어 재승인 필요 (동일값 재전송 또는 태그만 변경 시에는 전환하지 않음)
- DISCONTINUED 상태는 수정 불가 (PRODUCT_010)
- FORCE_INACTIVE 상태는 판매자 소명/재승인을 위해 수정 가능
    - 구매자에게 노출되지 않음
    - 수정 시 APPROVE_REQUESTED로 전환되어 재승인 절차를 거침
    - FORCE_INACTIVE → APPROVED 전환은 관리자 검토 필요
- 상품 삭제는 Soft Delete(DISCONTINUED)
- 진행 중 주문 존재 시 상품 삭제 불가
- 판매자 정지 시 상품 상태 FORCE_INACTIVE 처리
- 상품당 카테고리 매핑:
    - 최소 1개
    - 최대 3개
- 상품 이미지:
    - 최소 1개
    - 최대 10개
    - 첫 번째 이미지가 썸네일(display_order = 1)
- 상품 이미지 삭제 방식:
    - DB는 Soft Delete (product_image.status = DELETED)
    - S3 객체는 비동기로 처리
- 대표 이미지(display_order = 1) 삭제 시 다음 display_order 이미지가 자동으로 썸네일로 승격
- 상품 태그:
    - 판매자 상품 등록/수정 시 자유 입력
    - 최대 10개, 각 30자 이하
    - 저장 시 trim + 소문자 정규화 (앞뒤 공백 제거, 대소문자 통일)
    - null이면 변경 없음, 빈 리스트면 전체 삭제
    - 태그 변경은 재심사 대상이 아니며 즉시 반영 (재승인 불필요)

---

## 상품 옵션 / 재고

- 상품 옵션:
    - 최소 1개(무옵션)
    - 최대 5개
    - 수량이 옵션 쪽에 있어서 무옵션 추가하여 상품 최소 1개로 시작
- 동일 상품 내 옵션 조합 중복 불가 (Composite Unique Constraint)
- 재고(stock) 음수 불가
- 재고 차감은 주문 생성 시 ProductItem 비관적 락으로 보장한다.
- 재고 복구는 주문 취소 시 DB 원자 update(stock = stock + qty)로 보장한다.
- 재고 0개:
    - status는 ON_SALE 유지
    - 구매자 화면에는 "일시 품절" 표시
- 옵션 판매 중단 시 status = STOP
- STOP 상태 옵션은 장바구니 담기 및 주문 불가
- 가격 변경은 진행 중 주문에 영향 없음
- 주문 생성 시 상품명 / 옵션명 / 가격 / 썸네일 URL 스냅샷 저장
- 장바구니 담기 시점이 아닌 주문 생성 시점에 최종 재고 검증 수행
- 재고 변경 경로:
    - 입고: POST /inbound (이력 기록: INBOUND)
    - 보정: PATCH /items (이력 기록: ADJUSTMENT)
    - 차감: 주문 트랜잭션 (현재 이력 미기록, 향후 OUTBOUND 확장 예정)
    - 복구: 주문 취소 트랜잭션 (DB 원자 update, 현재 이력 미기록, 향후 RESTORE 확장 예정)
- 옵션당 재고 상한: 99,999
- 모든 재고 변동(입고,보정)은 inventory_history에 기록

---

## 카테고리

- 카테고리 깊이:
    - 최대 3단계 (대 > 중 > 소)
- 같은 부모 아래 카테고리명 중복 불가 (저장 전 앞뒤 공백 trim 정규화 + (parent_id, name) DB 유니크 제약. 단 루트 카테고리는 parent_id가 NULL이라 MySQL 특성상 DB 유니크가 적용되지 않아 앱 레벨 검증으로 보장)
- 카테고리 생성/수정/삭제는 ADMIN 또는 SUPER_ADMIN만 가능
- 상위 카테고리 삭제 시 하위 전체 삭제
- 상품 매핑된 카테고리 삭제 불가
- 카테고리 조회:
    - Caffeine 캐시 TTL 1시간 → 확장 시 변경 고려
- 카테고리 생성/수정/삭제 시 캐시 무효화

---

## 검색

- 검색 대상:
    - APPROVED 상품만 검색 가능
- 검색 방식:
    - MySQL FULLTEXT(name, description)
- 검색어 최소 길이:
    - 검색어 1자도 검색 자체는 허용하되, 1자 검색어는 인기 검색어 집계에서만 제외한다 (FULLTEXT 검색은 2자 이상에서 정상 동작)
- 인기 검색어:
    - 노출은 TOP 10 (Redis ZSET 기반 실시간 집계)
    - ZSET엔 상위 50위까지 보관 (= 매일 00:00 초기화 + 시간가중치 재계산으로 순위가 출렁여, 50위까지 들고 있어야 노출 TOP 10이 안정적으로 유지됨)
    - 검색 시 ZINCRBY로 집계
    - 매일 00:00 초기화
    - 시간에 따라 가중치 다르게 적용하여(10%/30%/60%) 인기검색어 집계
- 비로그인 검색 허용
- 가격 필터:
    - 옵션 가격 중 하나라도 범위 포함 시 노출
- 정렬:
    - LATEST
    - PRICE_ASC
    - PRICE_DESC
    - POPULAR
    - 인기순(POPULAR) 정렬은 키워드·카테고리·가격 필터를 적용하지 않고 인기 랭킹 순서를 그대로 반환(가격 필터·키워드·카테고리 필터는 LATEST/PRICE 정렬에만 적용)
- 페이징:
    - size 10~20
- 상품 목록 캐시:
    - Redis TTL 5분 (자연 만료 기반 — 판매자 상품·옵션 수정 시 즉시 무효화하지 않음)
    - 인기순(POPULAR) 정렬은 캐시 대상에서 제외 (랭킹이 5분마다 출렁여 stale 방지)
- 상품 단건 캐시 (인기상품) :
    - Redis TTL 10분
    - 수정/삭제 시 캐시 무효화
- Redis 장애 시 DB 조회 fallback 수행 (인기검색어/인기상품 제외)
- 인기검색어/인기상품 fallback은 M3 고도화 단계에서 추가:
    - 2+3 조합: 인기상품 DB 스냅샷 + 인기검색어 search_history 집계
    - 또는 1번: Redis HA 구성 (Sentinel/Cluster)

---

## 조회수 / 인기 상품

- 조회수:
    - Redis INCR 기반 실시간 집계
    - 5분마다 Redis → MySQL 동기화
    - 주기적으로 초기화 필요 (일주일)
- 동일 사용자 중복 조회:
    - 로그인 사용자: 5분 내 동일 상품 재조회 제외
    - 비로그인 사용자: 중복 조회 모두 허용
- 인기 상품 점수:
    - 조회수 70% + 판매수 30%
- 인기 상품:
    - Redis ZSET 기반 TOP 20 관리
    - 5분마다 랭킹 갱신
    - Redis ZSET TTL 10분 (갱신 주기 5분의 안전 마진)

---

## 연관 상품

- 같은 카테고리 내 인기순(조회수 70% + 판매수 30%) TOP 10 추천
- 자기 자신 상품 제외
- 품절/비활성 상품 제외
- AI 도전 단계:
    - related_product.relation_weight 기반 추천 고도화 예정