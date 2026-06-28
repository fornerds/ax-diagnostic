# 키 발급 가이드 — 아임웹 (imweb)

> 출처: `_workspace/imweb/02_validation.md` (doc-validator, verdict: spec_ready) + `mcp-servers/imweb/README.md` + 공식 개발자 문서 1회 재확인(아래 "출처").
> 대상 계약: **레거시 v2 API** (`https://api.imweb.me/v2`), **읽기 전용**.
> 핵심 원칙: 출처 있는 사실만 기재. 발급 위치(관리자 콘솔 메뉴 경로·버튼명)는 공식 ticker 페이지에서 verbatim 확인됨. 발급 직링크는 사이트별 콘솔이라 공통 고정 URL 부재(아래 "발급 절차" 참조).

---

## 연동 유형 / 인증 방식

- **연동 유형:** 공식 원격 MCP 커넥터 없음 → 로컬 stdio MCP 서버. (출처: Anthropic 커넥터 디렉터리에 아임웹 미등재)
- **인증 방식:** 사이트별 **API Key + Secret Key** → 액세스 토큰 교환. **OAuth 아님**(OAuth는 신규 `openapi.imweb.me` 세대로, 본 커넥터 미사용).
- **토큰 교환:** 서버가 런타임에 자동 수행 — `GET https://api.imweb.me/v2/auth?key={API_KEY}&secret={SECRET_KEY}`
  - 응답: `{"msg":"SUCCESS","code":200,"access_token":"{ACCESS_TOKEN}"}`
  - 공식 정본은 GET + 쿼리스트링. 일부 공식 예시는 JSON body(`-d '{"key","secret"}'`)를 쓰므로 서버는 GET 우선, 실패 시 POST body fallback.
- **API 호출 헤더(함정):** 레거시 v2는 호출 시 **`access-token: {ACCESS_TOKEN}`** 헤더 사용(`Authorization: Bearer` 아님) + `Content-Type: application/json`.
- 사용자는 액세스 토큰을 직접 주입할 필요 없음 — **API Key/Secret만** 환경변수로 설정하면 됨.

---

## 발급 절차 (스텝)

레거시 v2 키는 **사이트 관리자가 자가발급**합니다(별도 OAuth 동의/제휴 심사 없음).

> **발급 직링크:** 없음 — API Key/Secret Key는 **사이트(워크스페이스)별 관리자 콘솔**에서 발급하므로 모든 사이트에 공통으로 적용되는 고정 발급 URL이 존재하지 않습니다. 아래 메뉴 경로로 본인 사이트 관리자 화면에서 발급합니다. (출처: 공식 ticker 페이지 — 발급 URL 미기재, 메뉴 경로만 명시)

1. 아임웹(`https://imweb.me`)에 **관리자(사이트 소유자) 권한**으로 로그인.
2. 상단 **내 사이트** → 발급할 사이트의 **[관리]** 버튼 클릭 → 사이트 관리 페이지 진입.
3. 좌측 메뉴 **환경설정** → **외부 서비스 연동 (API)** 클릭.
4. **[API key 발급받기]** 버튼 클릭 → 발급된 **API Key** 와 **Secret Key** 확인.
   - 전체 경로: `내 사이트 > 관리 > 사이트 관리페이지 > 환경설정 > 외부 서비스 연동 (API) > [API key 발급받기]` (출처: 공식 ticker 페이지, verbatim)
5. 발급된 두 값을 각각 환경변수 `IMWEB_API_KEY`, `IMWEB_SECRET_KEY` 로 설정.
6. (액세스 토큰은 서버가 호출 시 `GET /v2/auth`로 자동 교환하므로 별도 발급 불필요.)

> 주의: API Key/Secret Key를 **삭제 또는 재생성**하면 기존 연동이 끊길 수 있으니, 운영 중이면 재연결이 필요합니다.

> 신규 OAuth 앱(`openapi.imweb.me`)은 아임웹 **심사 + 제휴계약**이 운영 전제이며 "특정 사이트만을 위한 서비스는 승인 안 됨" 제약이 있습니다. 단일 사이트 내부 자동화에는 레거시 v2 자가발급이 현실적입니다. (출처: 검증 스펙 R2)

---

## 전제조건

- **권한:** 사이트 **관리자(소유자) 권한** 필요(자가발급).
- **요금제:** **미확인** — 무료/유료 요금제별 API 사용 가능 여부가 공식 문서에 명시되어 있지 않음. 운영 전 본인 사이트에서 키 발급 가능 여부 확인 필요. (출처: 검증 스펙 R3, #28)
- **사업자 심사 / 발신번호 사전등록:** 레거시 v2 자가발급 키에는 **해당 없음**(읽기 전용 커넥터, SMS/발신 기능 없음).
- **제휴계약:** 레거시 v2 자가발급 키에는 **불필요**(신규 OAuth 앱에만 해당).

---

## 스코프 / 권한

- 레거시 v2에는 OAuth식 **스코프 동의 절차가 없음**. 발급된 키는 해당 사이트의 API 접근 권한을 가짐.
- (제휴사 전용) 헤더 `ACCESS-AFFILIATE: {제휴사 업체코드}` — **제휴사만** 입력. 일반 자가발급 키는 미사용. (출처: token.md)
- 본 커넥터는 **읽기 전용 9종**만 노출(상품/주문/회원/매장정보/카테고리/리뷰 조회). 쓰기·상태변경(배송/취소/반품/교환, 회원 생성·수정, 쿠폰 지급, 상품 등록·삭제)은 파괴적이라 의도적 제외. (출처: 검증 스펙 R4)

---

## 환경변수 매핑

| 변수명 | 값을 어디서 | 필수 |
|--------|-------------|------|
| `IMWEB_API_KEY` | 사이트 관리 > 환경설정 > 외부 서비스 연동(API)에서 발급한 **API Key** | 필수 |
| `IMWEB_SECRET_KEY` | 동일 화면의 **Secret Key** | 필수 |
| `IMWEB_ACCESS_AFFILIATE` | 제휴사 업체코드(제휴사 계정만). 일반 키는 비움 | 선택 |
| `IMWEB_BASE_URL` | 베이스 URL 오버라이드. 기본 `https://api.imweb.me/v2` | 선택 |

> 시크릿은 코드/`.mcp.json`에 실값을 쓰지 말고 환경변수로 주입. `.mcp.json`은 `${VAR}` 확장 사용.

---

## 출처 (공식 URL)

- **키 발급 메뉴 경로(공식, verbatim):** `내 사이트 > 관리 > 사이트 관리페이지 > 환경설정 > 외부 서비스 연동 (API) > [API key 발급받기]` — https://old-developers.imweb.me/getstarted/ticker  *(2026-06 재확인 — 메뉴 경로·버튼명 명시. 발급 직링크 URL은 사이트별 콘솔이라 부재.)*
- 토큰 발급 엔드포인트(`GET https://api.imweb.me/v2/auth`, key·secret 인증): https://old-developers.imweb.me/getstarted/token.md  *(2026-06 재확인 — 엔드포인트 명시, 단 발급 콘솔 위치는 미기재)*
- 호출 헤더(`access-token: {TOKEN}`) verbatim 예시: https://old-developers.imweb.me/getstarted/getexamples
- 제휴사 헤더(`ACCESS-AFFILIATE`): https://old-developers.imweb.me/getstarted/token.md
- 응답 envelope(`{msg, code, data, version, request_count}`) / 레이트리밋: https://old-developers.imweb.me/getstarted/getexamples · https://old-developers.imweb.me/llms-full.txt
- 신규 OAuth 세대 / 운영 전제(참고, 본 커넥터 미사용): https://developers-docs.imweb.me/llms-full.txt
- 원격 MCP 커넥터 부재 확인: https://support.claude.com/en/articles/11176164-pre-built-web-connectors-using-remote-mcp

---

## 미확인 / 주의

- **발급 콘솔 메뉴 경로(확인됨):** `내 사이트 > 관리 > 사이트 관리페이지 > 환경설정 > 외부 서비스 연동 (API) > [API key 발급받기]` — 공식 ticker 페이지(https://old-developers.imweb.me/getstarted/ticker)에서 메뉴 경로·버튼명 verbatim 확인됨(2026-06). **발급 직링크 URL은 사이트별 관리자 콘솔이라 공통 고정 URL 부재**(이는 정상 — 본인 사이트 관리 화면에서 발급). 검증 스펙 R3의 "미확인" 항목 중 메뉴 경로 부분은 해소됨.
- **요금제 게이팅(미확인):** API 사용 가능 요금제 조건이 공식 문서에 없음. 무료 요금제 가능 여부 등은 운영 전 확인.
- **토큰 수명(미확인):** 레거시 문서에 액세스 토큰 만료시간 미명시. 서버는 인증오류(HTTP 401/403 또는 응답 `code` 401/403) 감지 시 `GET /v2/auth`로 자동 재발급 후 1회 재시도하는 가드를 둠.
- **레이트리밋:** 목록 조회(products/orders/members)는 **초당 1회**, 그 외 초당 5회. 서버가 목록 도구에 1초 throttle + 429 시 지수 백오프(최대 3회) 적용. (출처: llms-full.txt)
- **신규 OpenAPI(보류):** `openapi.imweb.me`(OAuth2)의 개별 리소스 경로·스코프 문자열·호출 헤더(Bearer 여부)는 공식 레퍼런스(Scalar)에서 verbatim 확인 불가 → 본 커넥터 미포함. 확장 시 재검증 필요. (검증 스펙 R1)
