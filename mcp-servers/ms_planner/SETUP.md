# 키 발급 가이드 — Microsoft Planner (ms_planner)

> 모든 항목은 `_workspace/ms_planner/02_validation.md`(검증본)·`README.md`의 출처 있는 사실만 반영했다. 발급 콘솔 위치·앱 등록 절차는 Microsoft Entra 공식 문서(2026-06-15 갱신)로 1회 확인해 보강했다. 추측한 값은 없으며, 불명확한 항목은 "미확인"에 남겼다.

## 연동 유형 / 인증 방식

- **연동 유형**: REST API 직접 호출(Microsoft Graph v1.0, 베이스 `https://graph.microsoft.com/v1.0`). Anthropic 공식 Microsoft 365 커넥터는 Planner 스코프를 제공하지 않으므로 사용 불가 → 자체 앱 등록 필요.
- **인증 방식**: Microsoft identity platform(Entra ID) OAuth2. 모든 Graph 요청에 `Authorization: Bearer {access_token}` 부착.
- **지원 흐름(2모드)**:
  - `delegated` — Authorization Code(사용자 로그인, refresh token 사용). `list_my_tasks`(`/me`) 동작.
  - `app` — Client Credentials(앱전용). scope는 `https://graph.microsoft.com/.default` 강제, **관리자 동의 필수**, `/me` 컨텍스트 없음(→ `list_user_tasks` 사용).
- **토큰 엔드포인트**: `POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token`

## 발급 절차 (스텝)

> 발급 콘솔: **Microsoft Entra admin center** (https://entra.microsoft.com)

1. **콘솔 접속**: https://entra.microsoft.com 에 최소 Application Developer 권한 계정으로 로그인. (테넌트가 여러 개면 상단 Settings에서 대상 테넌트로 전환.)
2. **앱 등록**: **Entra ID > App registrations > New registration**. 앱 이름 입력, Supported account types는 단일 테넌트 권장. **Register**.
3. **App ID / Tenant ID 확인**: 등록 후 **Overview** 페이지에서 **Application (client) ID** 와 **Directory (tenant) ID** 를 기록 → `MS_CLIENT_ID`, `MS_TENANT_ID`.
4. **client secret 생성**: **Certificates & secrets > Client secrets > New client secret** 로 시크릿 발급. **생성 직후 값(Value)을 즉시 복사**(이후 다시 볼 수 없음) → `MS_CLIENT_SECRET`. (인증서/페더레이션 자격증명으로 대체 가능.)
5. **API 권한 부여**: **API permissions > Add a permission > Microsoft Graph** 에서 아래 "스코프/권한" 표의 권한 추가.
   - `app` 모드: Application permissions 선택 → 권한 추가 후 **Grant admin consent**(관리자 동의) 필수.
   - `delegated` 모드: Delegated permissions 선택. 상위(`Group.*`) 권한은 관리자 동의 대상.
6. **(delegated 전용) Redirect URI 등록 + refresh token 발급**:
   - **Authentication** 에서 Redirect URI 등록 → `MS_REDIRECT_URI`.
   - 브라우저로 동의: `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize?client_id={client_id}&response_type=code&redirect_uri={redirect_uri}&response_mode=query&scope=Tasks.ReadWrite%20offline_access%20openid%20profile`
   - 리디렉트된 URL의 `code`를 토큰 엔드포인트에 교환(`grant_type=authorization_code`)하고, 응답의 `refresh_token`을 `MS_REFRESH_TOKEN`에 설정. (자세한 curl 예시는 README "인증 모드 > delegated" 참조.)

## 전제조건

1. **조직(work/school) 계정 전용.** 개인 Microsoft 계정은 모든 Planner API에서 "Not supported". 도입 기업은 Microsoft 365 Business/Enterprise + Entra 테넌트 보유 필수.
2. **관리자 동의.** `app`(Client Credentials) 모드 및 상위 권한(`Group.Read.All`/`Group.ReadWrite.All`)은 Entra Global Administrator 등의 관리자 동의가 선결 → 고객사 IT 관리자 협조 필요.
3. **프리미엄 플랜 데이터 미노출.** 새(프리미엄)/Project for the web 기반 플랜·작업은 Planner API로 접근 불가, basic 플랜만 노출(런타임 한계 — 안내 문구 필요).
4. 사업자 심사·발신번호 사전등록 등 별도 심사: **없음.**

## 스코프 / 권한

| 용도 | 위임(delegated) | 앱전용(app) | 비고 |
|------|-----------------|-------------|------|
| 읽기 전용 도구만 | `Tasks.Read` | `Tasks.Read.All` | |
| 쓰기 포함(권장 기본) | `Tasks.ReadWrite` | `Tasks.ReadWrite.All` | 13개 도구 전체 사용 |
| 그룹 단위 광범위 조회 | `Group.Read.All` / `Group.ReadWrite.All` | (해당 앱 권한) | 상위 권한 — 관리자 동의 대상 |

- `app` 모드는 개별 스코프 대신 `https://graph.microsoft.com/.default`로 토큰을 요청(부여된 Application 권한이 적용됨).
- 개인 Microsoft 계정은 위 모든 Planner 권한에서 "Not supported".

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `MS_TENANT_ID` | Entra app **Overview** 의 Directory (tenant) ID(GUID 또는 도메인) | 예 |
| `MS_CLIENT_ID` | Entra app **Overview** 의 Application (client) ID | 예 |
| `MS_CLIENT_SECRET` | **Certificates & secrets** 에서 발급한 client secret 값 | 예 |
| `MS_AUTH_MODE` | `app`(기본, Client Credentials) 또는 `delegated` | 아니오 |
| `MS_REFRESH_TOKEN` | delegated 흐름의 refresh token(authorize→token 교환으로 획득) | `delegated` 시 예 |
| `MS_REDIRECT_URI` | **Authentication** 에 등록한 Redirect URI | 아니오(delegated 발급 시 사용) |
| `MS_GRAPH_SCOPES` | 요청 스코프(delegated 전용; app은 `.default` 강제) | 아니오 |
| `MS_READ_ONLY` | `true` 면 쓰기 도구 비활성화 | 아니오 |
| `MS_GRAPH_BASE_URL` | Graph 베이스 오버라이드(국가 클라우드) | 아니오 |
| `MS_LOGIN_BASE_URL` | 토큰 authority 호스트 오버라이드 | 아니오 |

> 시크릿(`MS_CLIENT_SECRET`, `MS_REFRESH_TOKEN`)은 코드/리포지토리에 하드코딩 금지. `.env`(.gitignore) + `.env.example`로 키 이름만 노출.

## 출처 (공식 URL)

- 앱 등록(콘솔 위치·절차): https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app (Entra admin center https://entra.microsoft.com, updated 2026-06-15)
- 인증 개념(Bearer 토큰·MSAL 권장): https://learn.microsoft.com/en-us/graph/auth/auth-concepts
- Client Credentials / 토큰 엔드포인트: https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow
- Planner 개요(베이스 URL·ETag·basic 플랜 한정): https://learn.microsoft.com/en-us/graph/api/resources/planner-overview?view=graph-rest-1.0
- Planner 권한/엔드포인트(작업·플랜·버킷): https://learn.microsoft.com/en-us/graph/api/planneruser-list-tasks?view=graph-rest-1.0 , https://learn.microsoft.com/en-us/graph/api/planner-post-tasks?view=graph-rest-1.0
- Anthropic Microsoft 365 커넥터(Planner 스코프 미제공 확인): https://support.claude.com/en/articles/12542951-set-up-the-microsoft-365-connector

## 미확인 / 주의

- **원격 M365 커넥터 URL/전송**: 공개 문서에 명시 없음. route=build(자체 앱)이므로 무관.
- **페이지네이션 기본 크기 / `$top`·`$filter` 지원 범위**: Planner 문서에 미명시 → 구현은 `@odata.nextLink` 순회로 전수 수집, `$filter` 의존 금지.
- **변경 알림/Webhook(`/subscriptions`)**: Planner 대상 동작 미확인 → 계약 제외.
- **앱전용 모드 실사용 적합성**: 테넌트 정책 검토 필요. 1차 도입은 delegated(Authorization Code) 우선 권장.
- **국가 클라우드**: Global / USGov L4·L5 지원, 중국(21Vianet) 미지원. 비-Global은 `MS_GRAPH_BASE_URL`/`MS_LOGIN_BASE_URL` 오버라이드 필요.
- **client secret 만료**: 시크릿은 만료 기간이 있으므로 갱신 운영 필요(만료 시 인증 실패).
