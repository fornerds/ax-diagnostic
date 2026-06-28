# 키 발급 가이드 — Tableau (tableau)

> 근거: `_workspace/tableau/02_validation.md`(공식 소스 1:1 대조 검증본) + `README.md`.
> 발급 UI 절차(PAT)는 공식 Tableau 도움말 1회 보강(web_checked). 추측 없음 — 출처 있는 사실만 기재.

## 연동 유형 / 인증 방식
- **연동 유형**: "배포형 build" — 공식 [`@tableau/mcp-server`](https://www.npmjs.com/package/@tableau/mcp-server)(검증 `2.18.1`, Apache-2.0, "Tableau Supported")를 배포·인증 구성·툴 노출 범위만 설정. 도구 신규 코딩 없음.
- **백엔드**: Tableau Cloud(완전 지원) 또는 Tableau Server(OAuth 사용 시 2025.3+, 그 외 모든 지원 버전; Pulse 불가).
- **인증 방식(AUTH 값, 4종)**: `pat`(기본·권장 시작점) / `uat`(User Attribute Token) / `direct-trust`(Connected App JWT) / `oauth`. 부가 플래그 `ENABLE_PASSTHROUGH_AUTH`(passthrough).
  - **단일 사용자/로컬**: `stdio` + `pat` (가장 단순, 권장 1차).
  - **다중 사용자/원격(HTTP)**: 공식 서버가 OAuth 사실상 강제 → `oauth` 또는 `direct-trust`. (같은 PAT 동시 로그인은 기존 세션 종료 → 다중 클라이언트에 PAT 금지.)

## 발급 절차 (스텝) — 기본 PAT(Personal Access Token)
> **발급 직링크: 없음.** PAT 생성 화면은 단독 URL이 없고 본인 계정 설정 페이지 안에서만 열린다(사이트/워크스페이스별 URL). 따라서 아래 **메뉴 경로**로 진입한다. (공식 도움말 확인: 직링크 미제공)
> **진입 경로(요약):** 사이트 로그인 → 우상단 프로필 이미지/이니셜 → **My Account Settings** → **Personal Access Tokens** 섹션 → [**Create Token**] 버튼.

공식 PAT 생성 UI(Tableau Cloud / Tableau Server):
1. Tableau Cloud/Server 사이트에 로그인. (사이트 진입 URL은 본인 사이트, 예 `https://<your-site>.online.tableau.com`)
2. 우상단 프로필 이미지/이니셜 클릭 → **My Account Settings**(내 계정 설정).
3. 같은 페이지에서 **Personal Access Tokens** 섹션으로 스크롤/이동.
4. **Token Name**(토큰 이름) 입력 → **Create Token**(토큰 생성) 버튼 클릭. → 이 이름이 `PAT_NAME`.
5. 표시되는 대화상자에서 **Copy Secret**(시크릿 복사) → 안전한 곳에 저장. → 이 값이 `PAT_VALUE`. (이후 재확인 불가, 비밀번호처럼 취급.)
6. **Close** 클릭.
7. 발급한 `PAT_NAME`/`PAT_VALUE`를 환경변수로 주입(코드/`.mcp.json`에 실값 금지, `${VAR}` 확장 사용).

출처: https://help.tableau.com/current/pro/desktop/en-us/useracct.htm (메뉴 경로·버튼명·"직링크 없음" 공식 도움말로 재확인, web_checked 2026-06-28)

> `uat` / `direct-trust(Connected App)` / `oauth(EAS)`의 관리콘솔 발급 단계 세부 UI는 **미확인**(아래 "미확인/주의" 참조). 1차 구현은 PAT 사용 권장.

## 전제조건
1. **유료 Tableau Cloud 또는 Tableau Server 사이트** — MCP 서버는 래퍼일 뿐, 백엔드 사이트 없이는 무의미. (출처: 02_validation §누락·리스크 1)
2. **PAT 발급 가능 여부는 사이트 관리자 설정에 종속** — 사이트에서 PAT 생성이 비활성화돼 있으면 발급 불가. (출처: 02_validation §누락·리스크 2)
3. **사용자에게 API Access 권한** + 사이트에 **VDS / Metadata API 활성화** — `query-datasource`(VizQL Data Service) 및 메타데이터 조회 동작 전제. Tableau Server는 관리자가 직접 켜야 할 수 있음. (출처: https://tableau.github.io/tableau-mcp/docs/configuration/tableau-config)
4. **Pulse 툴은 Tableau Cloud 전용** — Tableau Server 대상이면 `EXCLUDE_PULSE=true`(또는 `EXCLUDE_TOOLS`에 `pulse` 그룹)로 7개 Pulse 툴 제외 필수. (출처: https://tableau.github.io/tableau-mcp/docs/enterprise/tableau-server)
5. **런타임 Node.js 22.7.5+** — 공식 서버 요구(`engines.node >=22.7.5`). (출처: https://github.com/tableau/tableau-mcp/blob/main/package.json)
6. **HTTP 원격 배포 시 OAuth 강제** — `TRANSPORT=http`이면 `OAUTH_ISSUER` 필수(없으면 기동 실패; Tableau Server는 2025.3+). 자가호스팅 HTTP를 원격 Claude가 쓰려면 공개 인터넷 도달 필요(사내망/VPN 뒤면 로컬 stdio 사용). (출처: 02_validation §누락·리스크 5–6)
- 발신번호 사전등록·사업자 심사 등: **없음**(해당 없음).

## 스코프 / 권한
- **API Access 권한**(사용자): VDS / `query-datasource` 동작 전제. 없으면 query-datasource 실패.
- **사이트 설정**: VDS / Metadata API 활성화, Pulse는 Cloud에서 활성화.
- **site administrator 역할**: 삭제 툴(`delete-*`)·관리자 툴(`list-users`, admin-insights 4종)에 필요. 비관리자 호출 시 "requires site administrator permissions" 반환. admin-insights는 추가로 `ADMIN_TOOLS_ENABLED=true` 필요.
- **노출 툴 범위(계약 기본 EXCLUDE 권고)**: `delete-datasource`, `delete-workbook`, `delete-extract-refresh-task`, `update-cloud-extract-refresh-task`, `get-oauth-token`, `revoke-access-token`, `reset-consent`, admin-insights 4종(`query-admin-insights-*`, `get-stale-content-report`). Tableau Server 대상이면 `pulse` 그룹 전체 추가 제외.
  - 별도 OAuth 스코프 발급 절차는 PAT 경로에 없음(REST sign-in으로 credentials token 자동 교환).
- 출처: https://github.com/tableau/tableau-mcp/blob/main/src/tools/web/adminGate.ts , https://tableau.github.io/tableau-mcp/docs/configuration/tableau-config

## 환경변수 매핑
> 기본 권장 구성(로컬 stdio + PAT)의 핵심값. 전체 목록은 02_validation §환경변수 목록 / `.env.example` 참조.

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `SERVER` | Tableau 사이트 URL (예 `https://your-site.online.tableau.com`) | 필수(AUTH≠oauth) |
| `SITE_NAME` | 사이트 콘텐츠 URL (Cloud 기본 사이트는 빈값 가능) | 조건부(기본 `''`) |
| `TRANSPORT` | `stdio`(기본) 또는 `http` | 선택 |
| `AUTH` | `pat`/`uat`/`direct-trust`/`oauth` (미설정 시 stdio→pat, http→oauth 조건) | 선택 |
| `PAT_NAME` | My Account Settings → Personal Access Tokens 에서 입력한 토큰 이름 | 필수(AUTH=pat) |
| `PAT_VALUE` | PAT 생성 시 **Copy Secret** 으로 1회 복사한 시크릿 값 | 필수(AUTH=pat) |
| `CONNECTED_APP_CLIENT_ID` / `CONNECTED_APP_SECRET_ID` / `CONNECTED_APP_SECRET_VALUE` / `JWT_SUB_CLAIM` | Tableau 관리콘솔의 Connected App(Direct Trust) 등록 정보 | 필수(AUTH=direct-trust) |
| `OAUTH_ISSUER` (+ embedded authz 시 `OAUTH_REDIRECT_URI`, `OAUTH_JWE_PRIVATE_KEY`/`_PATH`) | OAuth/EAS(External Authorization Server) 구성값 | 필수(AUTH=oauth / TRANSPORT=http) |
| `UAT_TENANT_ID` / `UAT_ISSUER` / `UAT_PRIVATE_KEY`(또는 `_PATH`) / `UAT_KEY_ID` / `UAT_USERNAME_CLAIM` | User Attribute Token 구성값 | 필수(AUTH=uat) |
| `INCLUDE_TOOLS` / `EXCLUDE_TOOLS` | 노출 툴(개별/그룹) 포함·제외 | 선택(권장) |
| `EXCLUDE_PULSE` | `true` 시 pulse 그룹 제외 (Server 대상 필수) | 선택 |
| `PORT` | HTTP 포트 (기본 3927) | 선택 |
| `ADMIN_TOOLS_ENABLED` | admin-insights 툴 활성화(`true`) | 선택 |

> 시크릿(`PAT_VALUE`, `CONNECTED_APP_SECRET_VALUE`, 개인키 등)은 코드/`.mcp.json`에 실값 금지 → 환경변수·`${VAR}` 확장으로 주입.

## 출처 (공식 URL)
- PAT 발급 UI 절차: https://help.tableau.com/current/pro/desktop/en-us/useracct.htm
- PAT 보안(만료·MFA 필수): https://help.tableau.com/current/server/en-us/security_personal_access_tokens.htm
- PAT 인증(동시 로그인 시 세션 종료): https://tableau.github.io/tableau-mcp/docs/configuration/mcp-config/authentication/pat
- 인증/환경변수(소스): https://github.com/tableau/tableau-mcp/blob/main/src/config.ts
- Tableau 사이트 구성(VDS/Metadata/API Access): https://tableau.github.io/tableau-mcp/docs/configuration/tableau-config
- Tableau Server(OAuth 2025.3+, Pulse 불가): https://tableau.github.io/tableau-mcp/docs/enterprise/tableau-server
- 패키지/실행: https://www.npmjs.com/package/@tableau/mcp-server , https://github.com/tableau/tableau-mcp/blob/main/README.md
- Tableau × Claude 가이드: https://www.tableau.com/developer/learning/tableau-mcp-server-claude

## 미확인 / 주의
- **`uat` / `direct-trust(Connected App)` / `oauth(EAS)` 의 관리콘솔 발급 단계 세부 UI** — 환경변수명·서버 검증 로직은 소스로 확정되었으나, Tableau 관리콘솔에서 Connected App/EAS 키를 발급하는 화면별 클릭 절차는 본 검증 범위 밖(미확인). 1차 구현은 PAT 사용, uat/oauth/direct-trust는 "선택 인증 모드"로 두고 실제 배포 시 사이트 관리자가 콘솔에서 구성.
- **PAT 만료**: Tableau Server 기본 1년, Tableau Cloud는 사이트 설정에 종속. 추가로 **15일 미사용 시 만료**(02_validation 인용). 만료 토큰은 계정 페이지에서 자동 제거.
- **PAT 개수 제한**: Tableau Server 최대 10개, Tableau Cloud 최대 104개. (출처: https://help.tableau.com/current/pro/desktop/en-us/useracct.htm)
- **MFA 활성 시**: Tableau 인증 + MFA면 REST API 인증에 PAT 필수(자격증명 불가).
- **PAT 발급 자체가 사이트에서 비활성화**되어 있을 수 있음(관리자 설정 종속) → 발급 불가 시 관리자 확인 필요.
