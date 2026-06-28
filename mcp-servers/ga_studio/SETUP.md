# 키 발급 가이드 — 구글 데이터 스튜디오 / Data Studio (ga_studio)

> 검증 기준: `_workspace/ga_studio/02_validation.md` (검증일 2026-06-26) + `README.md`.
> 모든 사실은 공식 출처 URL 기준. 추측 없음. 끝까지 불명확한 건 "미확인"으로 남김.

## 연동 유형 / 인증 방식

- **연동 유형:** REST(JSON) API — 베이스 URL `https://datastudio.googleapis.com/v1` (버전 v1).
- **인증 방식:** Google OAuth 2.0 (사용자 동의 기반). 호출 시 `Authorization: Bearer <ACCESS_TOKEN>` 헤더 사용.
  - API key 불가 — 데이터 엔드포인트는 OAuth2를 강제한다(라이브 401 응답 "Expected OAuth 2 access token"으로 확인됨).
- **공식 MCP 커넥터 없음** → 로컬 REST 래핑 MCP가 유일한 공식 경로.
- **토큰 운영 방식(이 서버):** 사전에 발급한 OAuth `client_id` / `client_secret` / `refresh_token`을 환경변수로 받아, 서버 기동 시 access token을 갱신하는 방식이 기본.

## 발급 절차 (스텝)

> 발급 위치: **Google API Console** (`https://console.cloud.google.com/`) — APIs & Services.

1. **조직 계정으로 로그인.** 호출 사용자가 **Google Workspace 또는 Cloud Identity 조직 소속**이어야 한다(아래 전제조건 참조). 개인 무료 Gmail은 불가.
2. **프로젝트 선택/생성.** Google API Console에서 작업할 프로젝트를 선택하거나 새로 만든다.
3. **Data Studio API 사용 설정(Enable).** APIs & Services > Library 에서 "Data Studio API"를 찾아 프로젝트에 **Enable** 한다.
4. **OAuth 동의화면(Consent screen) 구성.** APIs & Services > OAuth consent screen 에서 동의화면을 설정하고, 필요한 스코프(아래 "스코프/권한")를 등록한다. 사용자 식별이 필요하면 `userinfo.profile` 포함.
5. **OAuth 클라이언트 생성.** APIs & Services > Credentials > Create credentials > OAuth client ID 에서 클라이언트(데스크톱 또는 웹) 를 생성 → **client_id / client_secret** 확보.
6. **사용자 동의로 토큰 획득.** OAuth 동의 흐름을 `access_type=offline`로 수행해 **refresh_token**(및 초기 access_token)을 확보한다.
   - 토큰 엔드포인트: `https://oauth2.googleapis.com/token` (refresh_token grant로 access token 갱신).
   - 쓰기 도구를 쓰려면 **동의 단계에서 non-readonly 스코프(`.../auth/datastudio`)로 동의**받아야 한다(스코프는 동의 시점에 고정).
7. **(필요 시) 관리자 앱 승인.** 조직 정책에 따라 관리자(어드민)의 앱 승인이 선행될 수 있다.
8. **환경변수 주입.** 확보한 client_id / client_secret / refresh_token 을 아래 "환경변수 매핑"대로 주입한다. 코드/`.mcp.json`에 실값 하드코딩 금지.

## 전제조건

- **조직 계정 전용 (차단 요인).** 호출 사용자는 **Google Workspace 또는 Cloud Identity 조직 소속**이어야 한다. 개인 무료 Gmail로는 API 호출 자체가 불가(401/403).
  - 원문: "The API is only available to users that belong to an organization with Google Workspace or Cloud Identity."
- **Data Studio API "사용 설정(Enable)"** 이 프로젝트에 되어 있어야 한다.
- **OAuth 클라이언트 + 사용자 동의(refresh_token 확보)** 가 선행되어야 한다.
- **관리자 앱 승인** 이 조직 정책에 따라 필요할 수 있다.
- 사업자 심사 / 발신번호 사전등록 / 유료 플랜 결제(개별 신청) 등 별도 심사 절차: **해당 없음** (단, 위 "조직 계정" 자체가 사실상 유료 Workspace/Cloud Identity 소속을 요구).

## 스코프 / 권한

| 용도 | 스코프 |
|------|--------|
| 읽기 (`search_assets`, `get_asset_permissions`) | `https://www.googleapis.com/auth/datastudio.readonly` (권장) 또는 `https://www.googleapis.com/auth/datastudio` |
| 쓰기 (`add_asset_members`, `revoke_asset_members`, `update_asset_permissions`) | `https://www.googleapis.com/auth/datastudio` (readonly 불가) |
| 사용자 식별(선택) | `https://www.googleapis.com/auth/userinfo.profile` |

> 쓰기 도구는 동의 단계에서 non-readonly(`.../auth/datastudio`) 스코프로 동의받은 refresh_token이 필요하다. 또한 호출자에게 해당 자산의 관리 권한이 있어야 한다.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `GOOGLE_OAUTH_CLIENT_ID` | API Console > Credentials 에서 생성한 OAuth 클라이언트의 client_id | 예 |
| `GOOGLE_OAUTH_CLIENT_SECRET` | 동일 OAuth 클라이언트의 client_secret | 예 |
| `GOOGLE_OAUTH_REFRESH_TOKEN` | 사용자 동의(`access_type=offline`)로 발급한 refresh_token | 예 |
| `GOOGLE_OAUTH_ACCESS_TOKEN` | (선택) 초기 access token. 없으면 refresh_token으로 발급 | 아니오 |
| `ENABLE_WRITE_TOOLS` | 쓰기 3종 노출 여부(`true`/`false`, 기본 false=읽기 전용) | 아니오 |

> 시크릿은 env로만 주입한다. `.env.example` 참고, 실제 `.env`는 `.gitignore` 처리됨.

## 출처 (공식 URL)

- Data Studio API 가이드 (인증·스코프·조직 한정 제약·베이스URL): https://developers.google.com/data-studio/integrate/api
- API 레퍼런스 (Base URI, 메서드): https://developers.google.com/data-studio/integrate/api/reference
- 타입/스키마 (Role/Member/AssetType enum): https://developers.google.com/data-studio/integrate/api/reference/types
- 토큰 엔드포인트(OAuth2): `https://oauth2.googleapis.com/token`
- Google API Console: `https://console.cloud.google.com/` (APIs & Services — Library / OAuth consent screen / Credentials)
- 공식 MCP 커넥터 부재 확인(Looker MCP는 별개 제품): https://docs.cloud.google.com/looker/docs/mcp

## 미확인 / 주의

- **[미확인] 전용 레이트리밋/쿼터 수치(QPS·일일한도).** 공식 수치 문서 부재(2회 탐색 후에도 없음). 코드에서 `429`/`RESOURCE_EXHAUSTED` 감지 시 지수 백오프(`Retry-After` 존중)로 흡수. 하드코딩 한도 금지. 프로젝트별 쿼터는 Cloud Console > APIs & Services > Data Studio API > Quotas 에서 확인.
- **[주의 — 적용 한계] 조직 계정 전용.** 401/403 발생 시 "Workspace/Cloud Identity 조직 계정 + 관리자 앱 승인 + 적절한 스코프가 필요합니다" 안내 필수.
- **[주의 — 가치 한계] 자산 관리 전용 API.** 리포트 생성/편집 불가, 차트·필터·차원 등 내부 데이터 조회 불가. "대시보드 수치를 자연어로 질의"는 이 API로 구현 불가. 제공 가치는 (a) 자산 목록·메타 검색, (b) 공유/권한 조회·관리뿐.
- **[주의 — 제품 혼동]** 검색 상위의 Looker(엔터프라이즈) MCP / MCP Toolbox는 **다른 제품**이다. 본 커넥터는 무료 Data Studio/Looker Studio의 공개 REST API만 래핑.
- **[주의 — 파괴적 작업]** `update_asset_permissions`(PATCH)는 부분 갱신이 아니라 **전체 권한 맵 치환**이다. `revoke_asset_members`는 멤버 일괄 제거. OWNER는 어떤 경로로도 변경/추가/제거 불가. 쓰기 도구는 기본 비활성(`ENABLE_WRITE_TOOLS=true`일 때만 노출).
