# 키 발급 가이드 — 11번가 셀러오피스 (st11_seller)

11번가 OPEN API 키를 발급받아 이 MCP 서버에 연결하는 방법. 모든 사실은 `_workspace/st11_seller/02_validation.md`의 검증 항목(공식 OPEN API 센터 렌더 확인 + 구현체 교차검증)에서 가져왔다. 출처가 없는 발급 디테일은 "미확인"으로 표시한다.

## 연동 유형 / 인증 방식

- **연동 유형**: REST(HTTP) + API Key. 요청·응답 모두 **XML**(JSON 미제공), 응답 인코딩은 **EUC-KR(CP949)** 가능. OAuth/공식 SDK/GraphQL 없음.
- **인증 방식**: 발급된 **32자리 정적 API Key 1개**. 갱신/리프레시 개념 없음.
  - **일반(공개) API**: 쿼리 파라미터 `key={APIKEY}` (헤더 불필요).
  - **셀러 API**: 커스텀 HTTP 헤더 `openapikey: {APIKEY}` (`Authorization` 아님). 쓰기 호출은 추가로 `Content-Type: application/xml`, 조회는 `Accept: application/xml`.
- 같은 발급키를 일반/셀러 양쪽에 공용으로 쓴다(주입 위치만 다름).

## 발급 절차 (스텝)

1. **11번가 셀러오피스 가입 + 본인인증 + Seller 회원전환** 완료. (미인증 일반 키로는 상품/카테고리 **조회만** 되고 셀러 도구는 동작하지 않음.)
2. 셀러오피스 **"서비스등록·확인 > 서비스등록/확인"** 메뉴에서 Seller API 키를 발급(서비스 등록)한다.
3. **"Seller API 정보 수정"** 화면에서 이 MCP 서버를 실행하는 호스트의 **고정 출구 IP 주소를 등록**한다. (11번가 API 서버는 외부 접근을 차단하며, IP를 입력해야 Seller API Key가 승인된다.)
4. **승인 대기**: 키 발급/승인까지 **약 1시간** 소요(즉시 아님). 승인 후 호출 가능.
5. 발급된 32자리 키를 환경변수 `ELEVENST_OPENAPI_KEY`에 넣는다.

> 승인 ~1시간은 벤더 가이드 근거이며 공식 페이지에는 시간이 명시되어 있지 않다(02_validation #5).

## 전제조건

- **셀러 계정**: 셀러오피스 가입 + **본인인증 + Seller 회원전환** 필수. (일반/미인증 키로는 공개 조회만 가능.)
- **고정 출구 IP**: 호출 IP 화이트리스트 등록이 **사실상의 권한 게이트**. 미등록 IP는 전 호출 차단(에러 `005 accessDeny`류). **클라우드/동적 IP/노트북 환경에서는 운영이 어렵다** — 고정 IP(또는 NAT 게이트웨이/프록시)가 전제.
- **사업자 심사 / 발신번호 사전등록**: 위 본인인증·Seller 전환 외 별도 절차는 검증된 출처에 없음 → **없음**(02_validation 범위 내 기준).
- **플랜(유료 등급)**: 키 발급에 별도 유료 플랜이 필요하다는 검증된 근거 없음 → **없음**.

## 스코프 / 권한

- OAuth 스코프 개념 없음. 권한은 **계정 인증 단계 + IP 화이트리스트**로 결정된다.
- **2단계 권한**:
  - **일반(공개) API** — 상품·카테고리 **조회만** (`search_products`, `search_categories`). 셀러 권한 불필요.
  - **셀러 API** — 상품 등록부터 주문/배송/클레임 전체. 본인인증·Seller 전환 + IP 등록 필수.
- **셀러발(發) 주문취소 권한 없음**: 11번가는 셀러가 직접 주문을 취소하는 공개 REST API를 제공하지 않는다. 구매자 취소요청에 대한 **승인/거부**만 가능.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `ELEVENST_OPENAPI_KEY` | 셀러오피스 "서비스등록·확인"에서 발급한 32자리 OPEN API 키. 일반은 `key` 쿼리, 셀러는 `openapikey` 헤더로 주입 | 예 |
| `ELEVENST_GENERAL_BASE_URL` | 일반 API 베이스. 기본값 `https://openapi.11st.co.kr/openapi/OpenApiService.tmall` | 아니오(기본값) |
| `ELEVENST_SELLER_BASE_URL` | 셀러 API 베이스. 기본값 `https://api.11st.co.kr` | 아니오(기본값) |

## 출처 (공식 URL)

- 연동 유형(Http/XML 처리): https://openapi.11st.co.kr/openapi/OpenApiFrontMain.tmall
- 일반 API 인증(`key` 쿼리)·파라미터·에러표: https://openapi.11st.co.kr/openapi/OpenApiGuide.tmall
- 셀러 API 인증(`openapikey` 헤더)·키 발급 절차·IP 등록·권한 2단계: https://openapi.11st.co.kr/openapi/OpenApiOperationGuide.tmall?operationType=SERVICE_METHOD
- 사전구축 원격 커넥터 목록(11번가 미포함 → 로컬 빌드 근거): https://support.claude.com/en/articles/11176164-pre-built-web-connectors-using-remote-mcp

> 셀러 API의 리소스 경로·도구 스펙 자체는 셀러 로그인 게이트로 공식 캡처 불가하여 복수 공개 구현체 교차검증본을 사용한다(02_validation #9·#10 및 "셀러 API 경로 교차검증 출처"). 본 발급 가이드의 인증·발급 절차·IP 등록은 위 공식 페이지 렌더 확인 근거다.

## 미확인 / 주의

- **승인 소요시간(~1시간)**: 공식 페이지에 시간 명시 없음(벤더 가이드 근거). 실제 더 걸릴 수 있음.
- **계정당 API ID 최대 4개**: 벤더 가이드만 언급, 공식 근거 못 찾음 → **미확인**(운영 참고용).
- **일일 호출 한도**: 한도 존재는 확정(에러 `004 overedTraffic`), 단 **수치는 비공개** → 보수적 스로틀/백오프 필요.
- **IP 화이트리스트가 운영 최대 제약**: 고정 IP가 없으면 발급 후에도 호출이 전부 막힌다. 발급 전에 운영 호스트의 출구 IP를 먼저 확보할 것.
- **EUC-KR 디코딩**: 응답을 UTF-8로 가정하지 말 것(한글 깨짐). 발급과 무관하나 연결 검증 시 주의.
- **셀러 도구별 정확한 쿼리/바디 필드·필수여부·택배사/배송방법 코드표**: 셀러 계정 로그인 가이드(`OpenApiGuide.tmall?categoryNo=38~46&apiSpecType=1`)에서 최종 대조 필요 → 일부 **미확인**.
