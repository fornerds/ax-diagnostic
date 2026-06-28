# 키 발급 가이드 — 메타 비즈니스 스위트 (meta_biz)

> 출처 기준: `_workspace/meta_biz/02_validation.md`(검증일 2026-06-26), `mcp-servers/meta_biz/README.md`.
> 원칙: 추측 없이 출처 있는 사실만 기재. 발급 디테일이 공식 문서로 확인되지 않은 부분은 "미확인"으로 남김.

## 연동 유형 / 인증 방식

- **연동 유형**: OAuth 2.0 기반 REST(Graph API). 본 MCP 서버는 OAuth 웹플로우를 **내장하지 않으며**, 외부에서 1회 발급한 **Page access token**을 환경변수로 주입받아 호출한다.
- **인증 모델**: **Facebook Login for Business**(단일 모델 채택). 베이스 호스트 `https://graph.facebook.com/v25.0/`, 인증은 Page access token.
- **서버가 받는 시크릿**: 사전 발급된 **Page access token**(`META_PAGE_ACCESS_TOKEN`). App ID/Secret은 토큰 갱신·`appsecret_proof` 생성용으로 **선택**.

## 발급 절차 (스텝)

토큰 발급 OAuth 흐름은 서버 외부에서 1회 수행한다(검증된 흐름).

> **발급 페이지 직링크**
> - 앱 대시보드(앱 생성/관리): **`https://developers.facebook.com/apps`** (= "내 앱(My Apps)" 진입점, 클릭 시 바로 앱 목록/생성 화면)
> - Graph API 탐색기(Page access token 즉시 발급 콘솔): **`https://developers.facebook.com/tools/explorer`**
> - 직링크 주의: 위 두 URL은 계정/앱에 종속되지 않는 고정 콘솔 URL이라 직접 사용 가능. 단 개별 앱 상세(App ID·Secret) 화면은 `https://developers.facebook.com/apps/<APP-ID>/settings/basic/` 형태로 **앱별(워크스페이스별) URL**이므로 범용 직링크는 없음 → 아래 메뉴 경로로 이동.

1. **Meta 앱 등록** — 앱 대시보드(`https://developers.facebook.com/apps`)에서 개발자 계정으로 로그인 → **[앱 만들기(Create App)]** 버튼 클릭 → 유스케이스 선택 시 페이지/로그인 관련 유스케이스를 고른다 → 앱 이름 입력 후 생성. 생성 시 **App ID**와 **App Secret**이 발급된다.
   - **App ID / App Secret 확인 경로**: 앱 대시보드 → 좌측 사이드바 **설정(Settings) → 기본 설정(Basic)** → App ID는 바로 표시, App Secret은 옆의 **[표시(Show)]** 버튼 클릭(비밀번호 재확인 후 노출). 직링크 형태: `https://developers.facebook.com/apps/<APP-ID>/settings/basic/`.
2. **사용자 인가** — Facebook Login for Business로 `client_id`(App ID) + `redirect_uri` + `scope`(쉼표 구분, 작업별 필요 권한은 아래 "스코프/권한" 참조)로 인가 요청 → 리다이렉트로 `?code=` 수신.
3. **code → User access token 교환**.
4. **User token → long-lived user token 교환** — 무만료 Page 토큰의 전제. 이 단계를 누락하면 단기 토큰이 되어 1~2시간 내 만료됨.
5. **Page access token 획득** — `GET /{user-id}/accounts` 호출로 대상 페이지의 **Page access token**을 받는다. long-lived user 토큰에서 파생해야 무만료 Page 토큰이 된다.
   - **콘솔 대안(수동 발급)**: Graph API 탐색기(`https://developers.facebook.com/tools/explorer`) 접속 → 우측 상단 **[사용자 또는 페이지(User or Page)]** 드롭다운에서 대상 페이지 선택 → 필요한 권한(scope) 체크 후 **[액세스 토큰 생성(Generate Access Token)]** → 발급된 Page access token 복사. (주의: 탐색기 기본 발급분은 단기 토큰일 수 있으므로 무만료가 필요하면 long-lived 교환 절차를 거칠 것.)
6. 획득한 Page token을 `META_PAGE_ACCESS_TOKEN`에 주입(서버 실행 환경의 `.env` 또는 `${VAR}` 확장).

> 주의: `GET /{user-id}/accounts`는 기본적으로 **단기(short-lived) Page 토큰**을 반환한다. "만료 없음"은 **long-lived user 토큰에서 파생한 Page 토큰**에 한해 성립한다(검증 항목, 출처: pages-api/getting-started).

## 전제조건

운영(실서비스) 전 선결 사항이 존재한다.

1. **App Review + Advanced Access + Business Verification**(실서비스 전제). `instagram_content_publish`, `pages_messaging`, `instagram_manage_messages`, `pages_manage_metadata` 등 핵심 권한은 App Review 대상이며, Advanced Access는 **Business Verification(사업자등록증 등) 필수**.
   - **개발/시연 단계**: 앱에 role을 가진 **자기 소유 페이지/IG 자산**으로 **Standard Access**에서 검증 가능(심사 전).
2. **인스타그램 = 프로페셔널 계정(비즈니스/크리에이터) + 페이스북 페이지 연결** 전제. 개인 계정 불가.
3. **IG 메시지/대화 엔드포인트는 "인증된 비즈니스 소유 앱"에서만 동작** → Business Verification 선결.

## 스코프 / 권한 (작업별)

- 페이지 게시/조회: `pages_manage_posts`, `pages_read_engagement`, `pages_manage_engagement`
- 통합 인박스(읽기 / 메신저 응답): `pages_messaging`, `pages_manage_metadata`, `pages_read_engagement`
- IG 발행(FB Login): `instagram_basic`, `instagram_content_publish`, `pages_read_engagement`
- IG DM 읽기(FB Login): `instagram_basic`, `instagram_manage_messages`, `pages_manage_metadata` (+ 비즈니스 인증 앱)

> scope는 인가 요청 시 **쉼표(,)로 구분**한다(검증됨).

## 환경변수 매핑

| 변수명 | 값을 어디서 | 필수 |
|--------|-------------|------|
| `META_PAGE_ACCESS_TOKEN` | 위 발급 절차 5단계(`GET /{user-id}/accounts` 결과). long-lived user 토큰에서 파생 권장 | 필수 |
| `META_IG_USER_ID` | 인스타그램 비즈니스 계정 ID(IG 발행/한도/상태 도구 기본값). 도구 인자로도 전달 가능 | 선택 |
| `META_DEFAULT_PAGE_ID` | 기본 페이지 ID. 미지정 시 도구 인자로 페이지 ID 필수 | 선택 |
| `GRAPH_API_VERSION` | API 버전 고정(기본 `v25.0`) | 선택(기본값 제공) |
| `META_APP_ID` | 앱 대시보드(`developers.facebook.com/apps`)에서 발급된 App ID. 토큰 갱신/디버그용 | 선택 |
| `META_APP_SECRET` | 앱 대시보드의 App Secret. `appsecret_proof` 생성용 | 선택 |

> 시크릿은 코드/`.mcp.json`에 실값으로 쓰지 말 것. `.env`(gitignore) 또는 `${VAR}` 확장 사용.

## 출처 (공식 URL)

- Pages API 시작(토큰 흐름, `/{user-id}/accounts`): https://developers.facebook.com/docs/pages-api/getting-started/
- Graph API 버전 changelog(v25.0 = 2026-02-18): https://developers.facebook.com/docs/graph-api/changelog/
- Instagram Platform 개요(OAuth 인가→code→token): https://developers.facebook.com/docs/instagram-platform/overview/
- Instagram 콘텐츠 발행(권한·한도): https://developers.facebook.com/docs/instagram-platform/content-publishing/
- 권한(App Review 게이트): https://developers.facebook.com/docs/permissions/
- Instagram App Review / Business Verification: https://developers.facebook.com/docs/instagram-platform/app-review/
- 앱 생성(App ID/Secret 발급): https://developers.facebook.com/docs/development/create-an-app/
- 앱 대시보드 콘솔(앱 생성/관리 직링크): https://developers.facebook.com/apps
- 기본 설정(App ID/Secret 위치 = Settings → Basic, Show 버튼): https://developers.facebook.com/docs/development/create-an-app/app-dashboard/basic-settings/
- Graph API 탐색기(Page access token 콘솔 발급 직링크): https://developers.facebook.com/tools/explorer
- Graph API 탐색기 가이드(User/Page 드롭다운으로 토큰 발급): https://developers.facebook.com/docs/graph-api/guides/explorer/
- (참고·범위 외) 공식 Ads MCP 커넥터 최신 URL: https://www.facebook.com/business/help/1456422242197840

## 미확인 / 주의

- ~~**앱 대시보드 내 App ID/Secret 정확한 위치**~~: **해소됨**(2026-06-28 공식 문서 재확인). 경로 = 앱 대시보드 → 좌측 사이드바 **설정(Settings) → 기본 설정(Basic)** → App ID 표시 + App Secret 옆 **[표시(Show)]** 버튼. 앱별 직링크 `https://developers.facebook.com/apps/<APP-ID>/settings/basic/`. (출처: docs/development/create-an-app/app-dashboard/basic-settings/)
- **IG DM *발송*은 호스트가 다름**: `https://graph.instagram.com/v25.0/{ig-id}/messages`(Bearer)로 `graph.facebook.com`과 호스트 상이 → 본 서버 1차 범위 제외(IG DM은 **읽기만** 지원). 발송이 필요하면 별도 IG Login 흐름·토큰 필요.
- **광고(Ads) 발급은 본 서버 범위 외**: 공식 Ads MCP 커넥터(Business OAuth, 별도 앱심사 불필요)로 연결. 원격 URL은 2차 출처 간 불일치(`mcp.facebook.com/ads` vs `mcp.meta.com/ads/<business-id>`)로 **미확인** → Meta 비즈니스 도움말 1456422242197840에서 최신 URL 확인.
- **토큰 수명**: long-lived user 토큰 교환 단계를 누락하면 Page 토큰이 1~2시간 내 만료. 발급 절차 4단계 필수.
- **버전 고정**: Meta는 버전 주기 종료 정책이 있어 `GRAPH_API_VERSION`으로 고정(현행 `v25.0`).
