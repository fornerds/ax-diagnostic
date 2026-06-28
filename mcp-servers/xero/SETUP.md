# 키 발급 가이드 — Xero (xero)

> 근거: `_workspace/xero/02_validation.md`(검증일 2026-06-26, doc-validator) 및 `mcp-servers/xero/README.md`. 출처 URL이 있는 사실만 기재. 발급 절차의 화면 단위 디테일 일부는 공식 문서가 JS 렌더로 본문 확정이 어려워 "미확인"으로 표기.

## 연동 유형 / 인증 방식
- **연동 유형:** OAuth 2.0 + REST(Accounting API). 공식 호스팅형 원격 커넥터는 없음 → 로컬/STDIO 방식.
- **인증 방식: OAuth 2.0, 두 경로 중 택1.**
  - **경로 A — Custom Connections (client_credentials, M2M).** 공식 1순위. 서버가 토큰 자동 발급·캐시. `XERO_CLIENT_ID` / `XERO_CLIENT_SECRET` 사용.
  - **경로 B — Authorization Code Flow 토큰 직접 주입.** 멀티 조직 / 한국 등 경로 A 불가 조직용. 외부에서 발급한 access token을 `XERO_CLIENT_BEARER_TOKEN`에 주입(설정 시 경로 A보다 우선).
- 출처: https://developer.xero.com/documentation/api/accounting/overview , https://developer.xero.com/documentation/guides/oauth2/custom-connections/

## 발급 절차 (스텝)

### 공통 — 개발자 앱 등록
1. Xero 개발자 포털 My Apps 관리 페이지로 직접 이동한다: **직링크 → https://developer.xero.com/app/manage** (Xero 계정 미로그인 시 로그인 후 같은 화면으로 진입).
   - 메뉴 경로: 개발자 포털(https://developer.xero.com) 로그인 → 상단 **My Apps** 탭.
2. **My Apps** 화면 우측 상단의 **[New app]** 버튼을 클릭한다. (`Add a new app` 창이 열림)
   - **integration type(연동 유형)을 선택**한다 — 여기서 발급 경로가 갈린다:
     - **경로 A(Custom Connections)** → integration type = **[Custom connection]** 선택. (아래 §경로 A 4A로 이어짐)
     - **경로 B(Authorization Code Flow)** → integration type = **[Web app]** 선택. (아래 §경로 B 4B로 이어짐)
   - 공통 입력: **App name**(데이터 공유 시 사용자에게 표시됨), **Company or application URL**(`https://` 접두 필수). Web app(경로 B)은 추가로 **Redirect URI**가 최소 1개 필요(절대 URI, 와일드카드 불가, 앱당 최대 50개; 로컬 테스트는 `http://localhost/` 사용 가능 — `http://127.0.0.1`은 불가).
   - 저장하면 앱 상세 페이지에서 **Client ID**가 제공되고, **Client Secret**은 [Generate a secret](또는 동등 버튼)으로 발급한다. Secret은 1회 노출이므로 즉시 저장.
   - 출처: https://developer.xero.com/documentation/guides/oauth2/custom-connections/ , https://developer.xero.com/documentation/guides/oauth2/auth-flow/ (My Apps → New app → integration type 선택 → name/URL/redirect URI 입력 → Client ID/Secret 발급).
3. 개발/PoC는 **Demo Company(무료)** 사용을 권장한다.
   - 출처: https://developer.xero.com/documentation/guides/oauth2/custom-connections/ ("connected to the Xero Demo Company for free for development").

### 경로 A — Custom Connections (AU/NZ/UK/US 조직)
4A. (위 2단계에서 integration type = **Custom connection** 선택 상태에서) 필요한 **API scopes를 선택**하고, 연결을 **승인할 사용자(authorising user)**를 지정한다. 지정한 사용자에게 승인 링크 이메일이 발송되며, 그 사용자가 링크에서 조직을 선택해 승인한다. 승인 완료 시 알림 이메일이 온다. 이때 **유료 월구독**이 등록된다(연결당 과금, Demo Company는 무료).
   - 승인 완료 후 앱 상세 페이지에서 **Client ID**가 노출되고 **Client Secret**을 생성할 수 있다. (My Apps에서 client_id/client_secret 재확인 가능)
   - 출처: https://developer.xero.com/documentation/guides/oauth2/custom-connections/ ("Log in to My Apps and click New App ... select Custom connection ... select the API scopes ... and who will authorise the connection ... client id will be available on the app details page and you can generate the client secret").
5A. 발급된 `Client ID` / `Client Secret`을 환경변수 `XERO_CLIENT_ID` / `XERO_CLIENT_SECRET`에 주입한다.
   - 토큰 자동 발급: `POST https://identity.xero.com/connect/token`, `grant_type=client_credentials`, 헤더 `Authorization: Basic base64(client_id:client_secret)`.
   - 출처: https://developer.xero.com/documentation/guides/oauth2/custom-connections/

### 경로 B — Authorization Code Flow 토큰 주입 (한국 등 그 외 조직 / 멀티 조직)
4B. 인가 요청: `GET https://login.xero.com/identity/connect/authorize?response_type=code&client_id=...&redirect_uri=...&scope=openid profile email accounting.transactions accounting.contacts accounting.settings accounting.reports.read offline_access&state=...`
5B. 토큰 교환: `POST https://identity.xero.com/connect/token` + 헤더 `Authorization: Basic base64(client_id:client_secret)`, body `grant_type=authorization_code&code=...&redirect_uri=...`.
6B. 응답의 `access_token`을 `XERO_CLIENT_BEARER_TOKEN`에 넣는다. (`offline_access` 요청 시 refresh token 발급 → `grant_type=refresh_token`로 갱신.)
   - **운영 주의:** access token은 보통 30분 만료. 본 자체 빌드 서버는 경로 B 토큰을 자동 갱신하지 않으므로 재주입/갱신 절차를 별도 설계해야 한다.
   - 출처: https://developer.xero.com/documentation/guides/oauth2/auth-flow/

### 공통 — 테넌트(조직) 선택
7. `GET https://api.xero.com/connections` (헤더 `Authorization: Bearer {access_token}`)로 연결된 조직 목록을 얻고 `tenantId`를 확인한다. 이후 모든 Accounting 호출에 `Xero-tenant-id: {tenantId}` 헤더가 필요하다.
   - 서버 동작: `XERO_TENANT_ID`가 있으면 그 조직으로 고정, 없으면 첫 조직 자동 선택·캐시.
   - 출처: https://developer.xero.com/documentation/guides/oauth2/tenants

## 전제조건
- **경로 A(Custom Connections) 전용 제약:**
  - **대상 조직이 AU·NZ·UK·US 소재여야 함.** 한국 등 그 외 조직은 경로 A 불가 → 경로 B 사용. (출처: https://developer.xero.com/documentation/guides/oauth2/custom-connections/)
  - **유료 월구독 필요**(연결당): $10/m AUD(inc GST), $10/m NZD(ex GST), £5/m GBP(ex VAT), $5/m USD(ex tax). **단일 조직.** Demo Company는 무료. (출처: 위 + https://developer.xero.com/pricing)
- **경로 B:** 브라우저 동의 1회 + 토큰 갱신 운영 부담.
- **사업자 심사 / 발신번호 사전등록:** 해당 없음(없음). (단 다수 고객 대상 SaaS 배포 시 Xero App Store 인증이 별도 필요 — 단일/소수 조직 도입엔 비해당. 출처: 02_validation §C)
- **granular scopes:** 2026-03-02 이후 생성한 앱은 자동 granular 스코프 체계. 지금 신규 발급하는 Client는 granular 명칭으로 스코프를 구성해야 함. (출처: https://developer.xero.com/faq/granular-scopes , https://developer.xero.com/documentation/guides/oauth2/scopes/)

## 스코프 / 권한
- 필수 기본: `openid profile email` (+ 경로 B는 `offline_access`).
- 작업별(granular 체계):
  - 송장·은행거래: `accounting.transactions` / `accounting.invoices`
  - 연락처: `accounting.contacts`
  - 계정·세율·추적: `accounting.settings` / `accounting.settings.read`
  - 보고서: `accounting.reports.read`
  - 수동분개 조회: `accounting.journals.read`
  - 급여: `payroll.employees` / `payroll.timesheets`
  - 연결관리: `app.connections`
- 서버 기본 스코프(미지정 시): `accounting.transactions accounting.contacts accounting.settings accounting.reports.read accounting.journals.read`. `XERO_SCOPES`로 조정.
- **정확한 granular 명칭은 발급 시 scopes 문서로 확정**(미확인 가드). 출처: https://developer.xero.com/documentation/guides/oauth2/scopes/

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `XERO_CLIENT_ID` | 개발자 포털 앱 등록 시 발급된 Client ID | 경로 A 필수 |
| `XERO_CLIENT_SECRET` | 개발자 포털 앱 등록 시 발급된 Client Secret | 경로 A 필수 |
| `XERO_CLIENT_BEARER_TOKEN` | Authorization Code Flow로 얻은 access token(설정 시 경로 A보다 우선) | 경로 B 필수 |
| `XERO_SCOPES` | client_credentials 요청 스코프(공백 구분). 미지정 시 기본 세트 | 선택 |
| `XERO_TENANT_ID` | `GET /connections` 응답의 조직 tenantId(고정 시). 미지정 시 첫 연결 자동 선택 | 선택 |

> 시크릿은 환경변수로만 주입. `.env`는 커밋 금지(`.gitignore` 포함), 커밋 대상은 `.env.example`뿐.

## 출처 (공식 URL)
- 개발자 포털: https://developer.xero.com
- My Apps 관리(앱 등록 직링크): https://developer.xero.com/app/manage
- Accounting API 개요: https://developer.xero.com/documentation/api/accounting/overview
- OAuth2 Auth Flow: https://developer.xero.com/documentation/guides/oauth2/auth-flow/
- Custom Connections: https://developer.xero.com/documentation/guides/oauth2/custom-connections/
- 테넌트/연결: https://developer.xero.com/documentation/guides/oauth2/tenants , https://developer.xero.com/documentation/best-practices/managing-connections/connections/
- 스코프: https://developer.xero.com/documentation/guides/oauth2/scopes/ , granular: https://developer.xero.com/faq/granular-scopes
- 가격: https://developer.xero.com/pricing
- 공식 MCP 서버(1순위 재사용 권고): https://github.com/XeroAPI/xero-mcp-server

## 미확인 / 주의
- **앱 등록 화면 절차(메뉴 경로·버튼명·입력 필드)는 공식 도움말 본문 기준으로 보강 완료**(2026-06-28 재검증): My Apps(https://developer.xero.com/app/manage) → **[New app]** → integration type(Custom connection / Web app) 선택 → name·URL(·Redirect URI) 입력 → Client ID 노출 + Client Secret 생성. developer.xero.com 본문은 여전히 JS 렌더로 WebFetch 타임아웃되나, 공식 페이지 텍스트 스니펫(custom-connections / auth-flow 가이드)으로 버튼명·필드명을 확인함. 단 **Client Secret 생성 버튼의 정확한 라벨(예: "Generate a secret")과 일부 화면 세부는 발급 시점 UI에서 최종 확인 권장.**
- **한국 조직은 권장 경로(Custom Connections=경로 A)를 쓸 수 없다.** 경로 B(토큰 주입) 또는 AU/NZ/UK/US 조직/Demo Company 필요.
- **Xero는 한국 회계/세무(전자세금계산서·부가세 양식·원화 현지화·국내 은행피드) 미지원** → 한국 기업 주 장부로 부적합, 글로벌 법인/해외 자회사 용도로 한정.
- **분당 레이트리밋 정확 수치 미확인** → 상수 하드코딩 금지(헤더 `X-MinLimit-Remaining`/`Retry-After`로 처리). 일일 한도는 티어 의존(Starter 1,000 / Core+ 5,000/일/connection).
- **Payroll 도구 14개는 본 자체 빌드에서 비활성**(국가별 베이스 경로 미확인). Payroll까지 쓰려면 공식 패키지(`@xeroapi/xero-mcp-server`) 재사용 경로 사용.
