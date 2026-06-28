# 키 발급 가이드 — 쿠팡 윙(WING) (coupang_wing)

> 출처: `_workspace/coupang_wing/02_validation.md` (검증일 2026-06-26) 및 `README.md`.
> 원칙: 검증된 출처가 있는 사실만 기재. 불명확한 항목은 "미확인"으로 남김.

## 연동 유형 / 인증 방식

- **연동 유형**: REST API + **HMAC(API Key)** 서명 방식. OAuth/토큰 교환·GraphQL 아님. 별도 scope 개념 없음.
- **베이스 URL**: `https://api-gateway.coupang.com`
- **발급 결과물**: `vendorId`(업체코드), `AccessKey`, `SecretKey` 3종.
- **인증 헤더**(모든 요청):
  - `Authorization: CEA algorithm=HmacSHA256, access-key={ACCESS_KEY}, signed-date={signedDate}, signature={signature}`
  - `X-Requested-By: {VENDOR_ID}` (업체코드, 필수)
  - `Content-Type: application/json;charset=UTF-8` (POST/PUT 시)
- 서명은 요청마다 호출 시점 UTC(`yyMMdd'T'HHmmss'Z'`)로 생성. (서명 알고리즘 상세는 README 참조)

출처: https://developers.coupangcorp.com/hc/en-us/articles/360033461914-Creating-HMAC-Signature

## 발급 절차 (스텝)

1. **WING 셀러 어드민 로그인** (사업자 인증 완료된 판매자 계정).
2. **판매자정보 > 추가판매정보 > OPEN API** 메뉴로 이동.
3. **OPEN API 키 발급(셀프서비스)** 진행 → `vendorId`(업체코드), `AccessKey`, `SecretKey`를 확인.
4. 발급된 3종 값을 MCP 서버 환경변수(`COUPANG_VENDOR_ID`, `COUPANG_ACCESS_KEY`, `COUPANG_SECRET_KEY`)에 주입.

출처(키 발급 위치·셀프서비스): https://developers.coupangcorp.com/hc/en-us/sections/20299721603481-Issue-API-Key
보조 출처(메뉴 경로): https://winselling.co.kr/guide/winshop/open_api/setting_coopang

## 전제조건

- **사업자 인증(사업자등록) 완료한 WING 판매자만** 키 발급 가능. 개인/미인증 계정은 발급 불가.
- 발신번호 사전등록 등 별도 통신 전제조건: **없음**(해당 API는 SMS/알림톡류 아님).
- 별도 유료 플랜 요구: 검증 출처에 명시 없음 → **미확인**(WING 셀러 자격 자체가 전제).

출처: `02_validation.md` "키 발급 전제" 섹션 (https://winselling.co.kr/guide/winshop/open_api/setting_coopang)

## 스코프 / 권한

- HMAC 방식으로 **별도 OAuth scope/권한 선택 개념 없음**. 발급된 키는 해당 vendorId의 Open API 전반에 사용.
- 도메인별 경로 prefix가 나뉨(주문/반품/CS/매출=`openapi`, 상품=`seller_api`, 정산=`marketplace_openapi`) — 키 자체에 도메인 제한은 없고 경로로 구분.

출처: https://developers.coupangcorp.com/hc/en-us/articles/360033461914-Creating-HMAC-Signature, `02_validation.md` 인증/공통 표

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `COUPANG_VENDOR_ID` | WING > 판매자정보 > 추가판매정보 > OPEN API 의 **업체코드(vendorId)** | 예 |
| `COUPANG_ACCESS_KEY` | 동일 화면의 **AccessKey** | 예 |
| `COUPANG_SECRET_KEY` | 동일 화면의 **SecretKey**(서명용) | 예 |
| `COUPANG_BASE_URL` | 기본 `https://api-gateway.coupang.com` (오버라이드용) | 아니오 |

> 시크릿 실값을 파일에 쓰지 말고 `.mcp.json`에서는 `${VAR}` 환경변수 확장을 사용.

## 출처 (공식 URL)

- HMAC 서명 / 인증 헤더: https://developers.coupangcorp.com/hc/en-us/articles/360033461914-Creating-HMAC-Signature
- API Key 발급(섹션): https://developers.coupangcorp.com/hc/en-us/sections/20299721603481-Issue-API-Key
- Node.js 예제(서명 코드): https://developers.coupangcorp.com/hc/en-us/articles/360042793752-Node-js-Examples
- 개발자 포털(한국 셀러 기준): https://developers.coupangcorp.com/hc/en-us
- (보조) WING OPEN API 설정 가이드: https://winselling.co.kr/guide/winshop/open_api/setting_coopang

## 미확인 / 주의

- **발급 즉시 사용 불가 가능**: 발급 후 사용까지 **최대 24시간+ 지연** 가능. 키 **유효기간 약 6개월**(만료 시 재발급 필요). (보조 출처: winselling.co.kr / 카페24 재발급 https://support.cafe24.com/hc/ko/articles/29472295175449)
- **공식 포털 직접 접근 제약**: 개발자 포털이 Cloudflare 봇 차단으로 직접 fetch/headless 접근 불가했음. 발급 위치·절차는 공식 인용 스니펫 + 보조 가이드 교차검증으로 확정했으며, **콘솔 내 정확한 버튼 라벨·세부 화면 단계는 셀러 어드민 버전에 따라 다를 수 있음**(상기 메뉴 경로 기준).
- **유료 플랜 요건**: 검증 출처에 명시 없음 → 미확인.
- **국제화 분기 주의**: `partner-developers.coupangcorp.com`(쿠팡 파트너스=제휴마케팅) 및 Taiwan Developers는 **다른 대상**. 한국 WING 셀러는 `developers.coupangcorp.com` + `api-gateway.coupang.com` 기준.
- **clock skew**: 서버 시간(UTC)과 어긋나면 401 → 서명은 항상 호출 시점 UTC로 생성. 허용 오차 정확값은 미확인.
- **레이트리밋**: vendorId당 초당 5회 미만 유지(초과 시 429). 키 발급과 무관하나 운영 시 주의.
