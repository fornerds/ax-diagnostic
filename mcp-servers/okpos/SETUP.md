# 키 발급 가이드 — OKPOS(외식·소매 POS) (okpos)

> **주의: 아래 발급 경로(메뉴/버튼)는 미확인 — 정확하지 않을 수 있습니다.** 발급 콘솔이 로그인 게이트/계정별이라 외부에서 화면을 확인하지 못했습니다. 실제 UI와 다를 수 있으니 공식 콘솔에서 직접 확인하세요. (추적: `mcp-servers/_SETUP_GAPS.md`)

> 근거: `_workspace/okpos/02_validation.md`(검증일 2026-06-26, route=build/spec_ready) + `mcp-servers/okpos/README.md`.
> 모든 사실은 위 검증본의 출처/라이브 근거에 기반한다. 추측 금지 — 불명확한 발급 디테일은 "미확인"으로 남긴다.

## 연동 유형 / 인증 방식

- **연동 유형**: REST API(Swagger/OpenAPI 2.0, Springfox). 베이스 URL `https://dum.okpos.co.kr/api` — **모든 경로에 `/api` prefix 필수**(미포함 시 nginx 404, 라이브 실증).
- **인증 방식**: OAuth2 — **Resource Owner Password Credentials grant**.
  - `client_id` + `client_secret`(클라이언트 자격증명)과 `username` + `password`(계정)를 함께 사용해 access_token을 발급받는다.
  - 토큰 발급: `POST https://dum.okpos.co.kr/api/oauth/token?grant_type=password&client_id={ID}&client_secret={SECRET}&username={USER}&password={PW}` — 파라미터는 **query string**(명세 `in=query`), 응답 `application/json`.
  - 이후 모든 업무 호출에 헤더 `Authorization: {access_token}` 첨부(securityDefinition `myToken`, `in=header`). 토큰 prefix(`Bearer `) 표기가 명세에 없음 → 발급 응답의 `token_type`을 확인해 적용(가드).
  - 라이브 실증: 더미 자격증명으로 토큰 발급 시 `HTTP 401 {"error":"invalid_client","error_description":"Bad client credentials"}`(realm `OKPOS_OAUTH_SERVER`). 토큰 없이 보호 엔드포인트 호출 시 `HTTP 401 {"error":"unauthorized",...}`.

## 발급 절차 (스텝)

> **중요**: OKPOS는 **client_id/client_secret·계정(username/password)의 공개 셀프발급 경로가 없다**(라이브 401 `Bad client credentials`로 실증, 공식 사이트에 개발자 포털·오픈API 신청 페이지 부재 — 2026-06-27 재확인). 아래 절차의 신청 양식·심사 기준 등 구체 디테일은 **공개 문서가 없어 미확인**이며, 실제 경로는 OKPOS 직접 문의로 확인해야 한다.

1. **OKPOS와 가맹/제휴 계약** — 대상 매장이 OKPOS POS/ASP에 등록·동기화돼 있어야 API가 의미가 있다(이미 OKPOS를 쓰는 매장 전제).
2. **OKPOS에 외부 개발자용 자격증명 발급 가부 문의** — OKPOS가 직접 `client_id`/`client_secret` 및 계정(`username`/`password`)을 발급하는 구조로 강하게 추정. 셀프 발급 콘솔/포털은 확인되지 않음(미확인).
3. **발급받은 자격증명 4종 수령** — `client_id`, `client_secret`, `username`, `password`.
4. **토큰 발급 동작 확인** — 위 `POST /api/oauth/token` 호출이 200(access_token 반환)을 내는지 검증. 401 `invalid_client`면 자격증명 자체 문제(재발급해도 동일).

> 발급 신청 URL/양식/심사 절차·소요기간은 **미확인**(공개 문서 부재). 셀프발급 콘솔/개발자 포털이 존재하지 않으므로(직링크 없음·메뉴 경로 없음 — 2026-06-28 재확인) 발급은 **OKPOS 고객/제휴 채널로 직접 문의**해야 한다.
>
> **문의 경로(확인된 진입점)**: OKPOS 기업 사이트 `https://www.okpos.co.kr` → 고객센터(`/customer/...`)의 제휴/문의 메뉴. (해당 페이지는 JS 렌더링이라 본 환경에서 전화번호·이메일 실값을 직접 캡처하지 못함 → 구체 연락처·신청 양식은 **미확인**, 사이트 접속해 확인 필요. 추정 기재 금지.)

## 전제조건

- **가맹/제휴 계약**: OKPOS와의 계약이 사실상 선결(자격증명을 OKPOS가 발급하는 구조로 추정). — 단, 구체적 심사 기준은 미확인.
- **가맹점 사전 동기화**: 대상 매장이 OKPOS POS/ASP(okasp.okpos.co.kr)에 등록·동기화돼 있어야 동작.
- **사업자 심사·발신번호 사전등록 등**: 본 API 범위에서 해당 요건은 **확인되지 않음(없음/미확인)**. (검증본에 근거 없음)

## 스코프 / 권한

- 명세상 OAuth scope는 `scope=["global"]` 단일. **세분 스코프 문서 없음** → 실제 권한은 발급 계정에 종속(미확인).
- 도구별 권한 부족 시 `403`(발급 계정 권한 부족 신호)으로 응답될 수 있음.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `OKPOS_CLIENT_ID` | OAuth2 client_id — OKPOS가 발급 | 예 |
| `OKPOS_CLIENT_SECRET` | OAuth2 client_secret — OKPOS가 발급 | 예 |
| `OKPOS_USERNAME` | 계정 username — 가맹/제휴 계정 | 예 |
| `OKPOS_PASSWORD` | 계정 password — 가맹/제휴 계정 | 예 |
| `OKPOS_BASE_URL` | 베이스 URL 오버라이드(기본 `https://dum.okpos.co.kr/api`) | 아니오 |
| `OKPOS_ENABLE_DESTRUCTIVE` | 파괴적 도구 활성화(`true` + 호출 시 `confirm:true` 동시 필요) | 아니오 |

> 실제 `.env`는 커밋 금지(`.gitignore` 포함). `.mcp.json`에는 시크릿 실값 대신 `${VAR}` 환경변수 확장 사용.

## 출처 (공식 URL)

- Swagger UI(유효 진입점): `https://dum.okpos.co.kr/api/swagger-ui.html` (라이브 200)
- OpenAPI(Swagger 2.0) JSON: `https://dum.okpos.co.kr/api/v2/api-docs?group=okgo` (라이브 200; businesses/tpay/apiManager 동일 패턴)
- OAuth2 토큰 엔드포인트: `POST https://dum.okpos.co.kr/api/oauth/token`
- OKPOS 기업 사이트: `https://www.okpos.co.kr` (개발자 포털/오픈API 신청 페이지 부재 — 2026-06-28 재확인). 발급 문의는 고객센터(`/customer/...`)의 제휴/문의 메뉴 경유.
- OKPOS 사용 가이드(GitBook): `https://okpos.gitbook.io/okpos` (POS/키오스크/배달/네이버 연동 등 사용법만, **API 발급·client_id/secret·OAuth 문서 없음** — 2026-06-28 확인)
- 검색엔진이 `https://dum.okpos.co.kr/api/`를 "오케이포스 O2O API 서비스"로 색인하나 **실제 라이브는 HTTP 404**(2026-06-28 재확인) — 색인 제목은 과거 스냅샷일 뿐, 브라우징 가능한 포털 아님.

> 주의: `https://dum.okpos.co.kr/api/`(루트) 자체는 **HTTP 404**로 브라우징 가능한 포털이 아님. 유효 진입점은 `/api/swagger-ui.html`.

## 미확인 / 주의

- **발급 신청 경로**: 자격증명 공식 발급 절차·신청 양식·심사 기준·소요기간 — **공개 문서 부재(미확인)**. OKPOS 직접 문의 필요.
- **셀프발급 콘솔 URL**: 존재하지 않음(공개 셀프발급 경로 없음, 라이브 실증).
- **토큰 응답 상세**: `access_token`/`token_type`/`expires_in`/`refresh_token`/`scope` 필드, 만료시간, `Bearer` prefix 적용 여부, refresh_token grant 실제 동작 — **명세 미정의(미확인)**. 발급 계정 확보 후 실호출로 확정 필요.
- **레이트리밋**: 정책·헤더 미확인(라이브 응답에 rate-limit 헤더 없음).
- **권한 범위**: `global` 외 세분 스코프·계정별 권한 범위 미확인.
- **차단 리스크**: 자격증명 미확보 시 코드는 빌드·기동되나 모든 호출이 **영구 401**. 발급 불가 정책이면 본 커넥터는 런타임 동작 불가.
