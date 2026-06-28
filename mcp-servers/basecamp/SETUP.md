# 키 발급 가이드 — Basecamp (basecamp)

> 근거: `_workspace/basecamp/02_validation.md`(검증일 2026-06-26, 공식 저장소 `basecamp/bc3-api` 원문 1:1 대조) 및 `mcp-servers/basecamp/README.md`. 추측 없이 출처가 있는 사실만 기재한다.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 stdio MCP 서버(공식 1차 remote 커넥터 없음, 공개 REST + OAuth2로 직접 연동).
- **인증 방식**: OAuth 2.0 Authorization Code 흐름, Bearer 토큰. Basic 인증 미지원(폐지됨).
- **OAuth 도메인**: `https://launchpad.37signals.com` (authorize / token / authorization.json).
- **API 베이스 URL**: `https://3.basecampapi.com/{account_id}/` — `account_id`는 `authorization.json`으로 동적 결정(하드코딩 금지). HTTPS 전용, 버전 프리픽스 없음.

## 발급 절차 (스텝)

1. **OAuth 앱 등록**: https://launchpad.37signals.com/integrations 접속 → 통합(integration) 앱 등록 → `client_id`, `client_secret`, `redirect_uri` 발급.
2. **Authorize(브라우저)**: 사용자를
   `GET https://launchpad.37signals.com/authorization/new?response_type=code&client_id={CLIENT_ID}&redirect_uri={REDIRECT_URI}`
   로 보내 승인 → `redirect_uri?code=...`로 인가 코드 수신.
3. **토큰 교환(backchannel)**:
   `POST https://launchpad.37signals.com/authorization/token`
   body: `grant_type=authorization_code`, `client_id`, `client_secret`, `redirect_uri`, `code`
   → 응답 `{ access_token, token_type: "Bearer", expires_in: 1209600, refresh_token }` (만료 2주).
4. **베이스 URL 부트스트랩**:
   `GET https://launchpad.37signals.com/authorization.json` (`Authorization: Bearer {token}`)
   → `accounts[]`에서 `product == "bc3"`인 `href`를 API 베이스 URL로 채택.
5. **토큰 갱신(만료 2주마다)**:
   `POST https://launchpad.37signals.com/authorization/token`
   body: `grant_type=refresh_token`, `refresh_token`, `client_id`, `client_secret`.

> legacy 파라미터(`type=web_server`/`type=refresh`)도 여전히 동작하나 권장형은 위 표준 파라미터(`response_type=code` / `grant_type=...`)다.

## 전제조건

- **유일한 전제**: 유효한 Basecamp 계정. 누구나 Launchpad에서 OAuth 앱을 등록할 수 있다.
- **한국식 게이트 없음**(검증됨): 사업자 심사·발신번호 사전등록·실명인증 등 한국 특화 게이트 **없음**.
- **별도 플랜 요건**: bc3-api 저장소 내 명문 없음 → **없음(저장소 기준)**.
- 참고: 상용/멀티테넌트 재배포 약관 제한은 API 저장소에 명문이 없어 **미확인**(아래 미확인 참조).

## 스코프 / 권한

- **세분 scope 없음**(검증됨). OAuth 요청에 scope 파라미터를 지정하지 않는다.
- 접근 범위는 **인증한 사용자의 Basecamp 권한**에 종속(전체 위임 모델).
- **User-Agent 필수**: 모든 API 요청에 앱명 + 연락처(이메일/URL)를 담은 User-Agent를 붙여야 한다. 누락 시 모든 요청 400. 예: `BasecampMCP (you@example.com)`.
- **권한 주의**: webhook 계열 호출(`list_webhooks`/`create_webhook`/`delete_webhook`)은 client(외부 고객) 토큰으로는 403 — 일반 멤버 토큰 필요.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `BASECAMP_CLIENT_ID` | Launchpad 앱 등록(https://launchpad.37signals.com/integrations) | 자동 갱신 모드에서 필수 |
| `BASECAMP_CLIENT_SECRET` | 동일(앱 등록 시 발급) | 자동 갱신 모드에서 필수 |
| `BASECAMP_REDIRECT_URI` | 동일(앱 등록값과 일치해야 함) | 토큰 발급 시 필요 |
| `BASECAMP_ACCESS_TOKEN` | 토큰 교환 응답의 `access_token` | 토큰 직접주입 모드에서 필수 |
| `BASECAMP_REFRESH_TOKEN` | 토큰 교환 응답의 `refresh_token` | 선택(자동 갱신 시 권장) |
| `BASECAMP_ACCOUNT_ID` | `authorization.json`의 `product=="bc3"` 계정 id(베이스URL 고정 오버라이드) | 선택(기본은 동적 결정) |
| `BASECAMP_USER_AGENT` | 직접 지정(앱명 + 연락처). 예 `BasecampMCP (you@example.com)` | 필수 |

최소 동작 조합:
- 토큰 직접주입 모드: `BASECAMP_ACCESS_TOKEN` + `BASECAMP_USER_AGENT`
- 자동 갱신 모드: `BASECAMP_REFRESH_TOKEN` + `BASECAMP_CLIENT_ID` + `BASECAMP_CLIENT_SECRET` + `BASECAMP_USER_AGENT`

## 출처 (공식 URL)

- API 문서(저장소): https://github.com/basecamp/bc3-api
- 인증 문서: https://github.com/basecamp/bc3-api/blob/master/sections/authentication.md
- OAuth 앱 등록(콘솔): https://launchpad.37signals.com/integrations
- Authorize 엔드포인트: https://launchpad.37signals.com/authorization/new
- Token 엔드포인트: https://launchpad.37signals.com/authorization/token
- 계정/베이스URL 부트스트랩: https://launchpad.37signals.com/authorization.json

## 미확인 / 주의

- **상용/재배포 약관 제한 — 미확인**: bc3-api 저장소에 멀티테넌트 임베드 제한 명문 없음. 약관 본문(별도 문서)은 검증 범위 밖 → 상용 배포 전 37signals API/서비스 약관 별도 검토 권고(법무).
- **공식 언어별 SDK — 미확인**: 37signals 공식 SDK는 확인 불가. 표준 HTTP 클라이언트로 구현(구현 비차단).
- **데이터 거주지(미국) — 미확인/운영 판단**: 서버/데이터 리전 해외(미국) 추정. 저장소 내 한국 리전 명문 없음.
- **레이트리밋 동적**: "50req/10s/IP"는 한 종류일 뿐, GET/POST별·동적 한도 존재. 한도 숫자 하드코딩 금지, 429 + `Retry-After` 동적 준수. IP 단위라 공유 서버에서 쉽게 포화.
- **토큰 만료 2주**: 장기 운용 시 `refresh_token` 기반 자동 갱신 권장.
- **세대명 표기 불일치**: 문서가 자신을 "Basecamp 5"로도 칭하나 식별자·베이스URL은 불변(`bc3` / `3.basecampapi.com`). 마케팅 세대명 기준 분기 금지.
