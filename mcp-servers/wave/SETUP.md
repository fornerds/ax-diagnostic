# 키 발급 가이드 — Wave (wave)

> 근거: `_workspace/wave/02_validation.md`(검증일 2026-06-26) + `mcp-servers/wave/README.md`. 출처 있는 사실만 기재했고, 불명확한 항목은 "미확인"으로 남긴다.

## 연동 유형 / 인증 방식

- 연동 유형: 단일 GraphQL 엔드포인트 호출형. `POST https://gql.waveapps.com/graphql/public`, `Authorization: Bearer <token>`, `Content-Type: application/json`, body `{"query": ..., "variables": ...}`. (검증됨: Clients)
- 공식 원격 MCP 커넥터는 없음 → 로컬 MCP 서버를 빌드해 연결한다. (route=build)
- 인증 방식은 2가지 (검증됨: Authentication):
  - **1차(권장, 단일 사용자 MCP): Full Access Token (Bearer)** — 콘솔에서 직접 발급. redirect_uri/토큰교환 불필요. 토큰은 발급한 본인 권한과 동일. (공식 문구상 "development purposes or personal applications only" 용도)
  - **2차(멀티 사용자/배포 시): OAuth 2.0 Authorization Code** — client_id/secret + authorize/token 교환. 본 MCP 서버 1차 경로에서는 불필요(자리표시자 env만 존재).

## 발급 절차 (스텝)

Full Access Token 기준 (검증됨: Authentication / Manage Applications, Last updated 2026-04-01):

1. Wave 계정으로 로그인한다. 로그인 페이지: `https://my.waveapps.com/login/`
2. 개발자 포털(`https://developer.waveapps.com/hc/en-us`)에서 **좌측 네비게이션의 [Manage Applications]** 로 들어간다. (직링크가 워크스페이스별이 아니라 인증 게이트 뒤 콘솔이라, 공식 문서가 공개하는 진입점은 좌측 메뉴 [Manage Applications]이며 별도 딥링크 URL은 공개되지 않음. 안내 문서: https://developer.waveapps.com/hc/en-us/articles/360019762711-Manage-Applications)
3. 앱이 없으면 **[Create an application]**(또는 첫 진입 시 **[Get started]**) 버튼을 누른다 → **Name** 입력(Description·Redirect URI는 비워둬도 됨) → **Terms and Conditions** 동의 체크 → **[Create your Application]** 버튼으로 앱을 생성한다.
4. 생성된 **앱 이름을 클릭**해 앱 상세로 들어간 뒤 **[Create token]** 버튼을 눌러 Full Access Token을 발급받는다.
5. 발급된 토큰을 환경변수 `WAVE_FULL_ACCESS_TOKEN`에 보관한다(파일/코드에 하드코딩 금지).

> 정확한 메뉴 경로: **로그인(`my.waveapps.com/login`) → 개발자 포털 좌측 메뉴 [Manage Applications] → [Create an application] → (Name 입력·Terms 동의) [Create your Application] → 앱 이름 클릭 → [Create token]**

> 폐기: 앱 상세의 **[Revoke]** 버튼으로 토큰을 무효화할 수 있다. (검증됨: Authentication)

> OAuth 경로(2차)를 쓸 경우: authorize `GET https://api.waveapps.com/oauth2/authorize/`, token `POST https://api.waveapps.com/oauth2/token/`, revoke `POST https://api.waveapps.com/oauth2/token-revoke/`. auth code는 1회 교환·10분 만료. (검증됨: OAuth Guide)

## 전제조건

- **Wave 계정**: 필요(무료 가입). 별도 사업자 심사/제휴 승인 게이트 **없음** — 개인이 즉시 앱·Full Access Token 발급 가능. (검증됨: Authentication)
- **유료 구독 게이트(핵심)**: API로 접근하는 비즈니스에 **활성 Wave Pro 또는 Wave Advisor 구독**이 필요. 비활성이면 OAuth/토큰 갱신·사용 시 **HTTP 403** 발생. → 데모/실사용 전 이 조건을 충족해야 한다. (검증됨: Authentication / OAuth Guide)
- **국내(한국) 게이트 해당 없음**: 발신번호 사전등록·공동인증서·홈택스/전자세금계산서 연동과 무관(미국·캐나다 회계 서비스). (검증됨: 02_validation)
- **이메일 발송 전제**: `send_invoice`(invoiceSend)는 `Business.emailSendEnabled = true`인 비즈니스에서만 동작. (검증됨: API Reference)

## 스코프 / 권한

- **Full Access Token 경로**: 스코프 지정 불필요(본인 권한 전체). 개발/단일 사용자 MCP는 이 경로 권장.
- **OAuth 경로(사용 시)**: 스코프 패턴은 `resource:operation`이며 **write가 read를 포함하지 않는다**(별도 부여), `*`는 전체. (검증됨: OAuth Scopes)
  - 본 도구 목록 기준 최소 스코프 셋:
    `user:read business:read customer:* invoice:* invoice:send estimate:* estimate:send product:* account:read sales_tax:read vendor:read`
    (거래 생성 도구를 포함하면 `transaction:write` 추가.)
  - **함정 확정**: `transaction:read` 스코프는 **존재하지 않음**(transaction은 write/* 만). 거래내역 읽기 불가의 근거. (검증됨: OAuth Scopes + API Reference)

## 환경변수 매핑

| 변수명 | 값을 어디서 | 필수 |
|--------|------------|------|
| `WAVE_FULL_ACCESS_TOKEN` | `my.waveapps.com/login` 로그인 → 개발자 포털 좌측 [Manage Applications] → [Create an application] → 앱 이름 클릭 → [Create token] 으로 발급한 Full Access Token | 필수(1차 권장 경로) |
| `WAVE_BUSINESS_ID` | 대상 비즈니스의 opaque base64 ID. 미설정 시 도구 인자/토큰의 기본·단일 비즈니스 사용 | 선택(권장) |
| `WAVE_GRAPHQL_URL` | 엔드포인트 오버라이드. 기본값 `https://gql.waveapps.com/graphql/public` | 선택 |
| `WAVE_CLIENT_ID` | (OAuth 경로만) 앱 client id — Manage Applications의 앱 상세 | 선택 |
| `WAVE_CLIENT_SECRET` | (OAuth 경로만) 앱 client secret — 앱 상세 | 선택 |
| `WAVE_OAUTH_REFRESH_TOKEN` | (OAuth 경로만) 토큰 교환으로 받은 장수 refresh token | 선택 |

> 시크릿 하드코딩 금지. 실제 값은 `.env`(gitignore됨) 또는 `claude mcp add --env`로 주입.

## 출처 (공식 URL)

- Clients(엔드포인트·호출 방식): https://developer.waveapps.com/hc/en-us/articles/360018856171-Clients (Last updated 2019-03-07)
- Authentication(2방식·Full Access Token 발급·구독 게이트·Revoke): https://developer.waveapps.com/hc/en-us/articles/360018856751-Authentication (Last updated 2026-04-01)
- Manage Applications(앱 생성·Create token 버튼·메뉴 경로): https://developer.waveapps.com/hc/en-us/articles/360019762711-Manage-Applications
- 개발자 포털 진입점: https://developer.waveapps.com/hc/en-us / 로그인: https://my.waveapps.com/login/
- OAuth Guide(authorize/token/revoke·403 구독 게이트): https://developer.waveapps.com/hc/en-us/articles/360019493652-OAuth-Guide (Last updated 2026-04-01)
- OAuth Scopes(스코프 패턴·목록·transaction:read 부재): https://developer.waveapps.com/hc/en-us/articles/360032818132-OAuth-Scopes (Last updated 2026-04-01)
- API Reference(라이브 GraphQL 스키마·emailSendEnabled 전제): https://developer.waveapps.com/hc/en-us/articles/360019968212-API-Reference (Last updated 2026-05-26)

## 미확인 / 주의

- **Manage Applications 앱 콘솔 딥링크(URL)**: 발급은 인증 게이트 뒤 콘솔에서 이뤄지며, Wave 공식 문서는 **별도 딥링크 URL을 공개하지 않는다**. 공식이 제공하는 진입점은 개발자 포털(`https://developer.waveapps.com/hc/en-us`) **좌측 메뉴 [Manage Applications]** 이다 → **직링크 없음 + 메뉴 경로로 진입**(위 "발급 절차" 참조). 안내 문서: https://developer.waveapps.com/hc/en-us/articles/360019762711-Manage-Applications (검증됨: WebSearch 교차확인 2026-06)
- **레이트리밋 수치/429/Retry-After**: 공식 미공개. 서버는 지수 백오프(3회) + 429/5xx 재시도로 가드. (미확인)
- **`expires_in` 실제 초값**(OAuth 경로): 문서에 숫자 없음. 응답의 `expires_in`을 동적으로 사용(하드코딩 금지). Full Access Token 경로는 만료가 사실상 없음(콘솔에서 Revoke로 폐기). (미확인)
- **OAuth redirect_uri localhost/loopback 허용 여부**: 문서에 명시 없음 → 미확인. 단, Full Access Token 경로는 redirect_uri 불요라 영향 없음.
- **결제(Payments) 기능 지역 제한 정확 범위**: 미확인. 본 MCP는 회계 객체 CRUD 중심이라 범위 외.
- **Webhook 이벤트/서명검증**: 로그인 후에만 노출되어 미확인 → 본 MCP 범위에서 제외.
