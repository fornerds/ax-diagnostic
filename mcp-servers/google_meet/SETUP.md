# 키 발급 가이드 — Google Meet (google_meet)

> 출처: `_workspace/google_meet/02_validation.md`(검증일 2026-06-26) 및 `mcp-servers/google_meet/README.md`. 모든 사실은 공식 Google 개발자 문서 출처 URL 1:1 대조본에서 가져왔다. 추측 금지.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 MCP 서버 빌드(route=build). 공식 Meet 전용 원격 MCP 커넥터는 없음(공식 원격 MCP는 Gmail/Drive/Calendar/Chat/People 5개만 제공, Meet 미포함).
- **베이스 URL / 버전**: `https://meet.googleapis.com`, 버전 v2.
- **인증 방식**: OAuth 2.0 **사용자 자격증명(user authentication) 전용**. **API 키 불가**(런타임에 무조건 실패). 단독 서비스계정도 불가 — 서비스계정은 domain-wide delegation으로 사용자 위임 시에만 가능.
- **토큰 흐름**: 설치형(데스크톱) OAuth 권장. `client_id`/`client_secret` + 사용자 동의 → authorization code → `refresh_token` 확보. MCP 서버는 저장된 `refresh_token`으로 런타임에 `access_token`을 자동 갱신하여 `Authorization: Bearer <access_token>` 헤더로 호출.

## 발급 위치 (콘솔 진입 URL)

> 발급은 **Google Cloud Console / Google Auth platform**에서 한다. OAuth 클라이언트는 **프로젝트별로 생성**되므로 발급 자체에 대한 개별 직링크는 없다(아래는 해당 프로젝트를 선택한 상태에서 진입하는 콘솔 화면 URL). 콘솔 우상단에서 대상 프로젝트를 먼저 선택할 것.

| 화면 | 진입 URL | 메뉴 경로 |
|------|----------|-----------|
| Meet API 사용 설정 | https://console.cloud.google.com/apis/library/meet.googleapis.com | 메뉴 > **API 및 서비스(APIs & Services)** > **라이브러리(Library)** > "Google Meet API" 검색 |
| OAuth 동의화면(브랜딩) | https://console.cloud.google.com/auth/branding | 메뉴 > **Google Auth Platform** > **브랜딩(Branding)** |
| 대상(테스트 사용자) | https://console.cloud.google.com/auth/audience | 메뉴 > **Google Auth Platform** > **대상(Audience)** |
| 데이터 액세스(스코프) | https://console.cloud.google.com/auth/scopes | 메뉴 > **Google Auth Platform** > **데이터 액세스(Data Access)** |
| OAuth 클라이언트 ID 발급 | https://console.cloud.google.com/auth/clients | 메뉴 > **Google Auth Platform** > **클라이언트(Clients)** |

> 참고: 구버전 메뉴 "API 및 서비스 > 사용자 인증정보(Credentials) > 사용자 인증정보 만들기 > OAuth 클라이언트 ID"는 현재 **Google Auth Platform > 클라이언트** 화면으로 이동되었다(동일 기능, 명칭 변경). 단서로 받은 구 경로도 같은 결과로 연결된다.

## 발급 절차 (스텝)

1. **Google Cloud 프로젝트에서 Meet API 사용 설정**
   - 진입: https://console.cloud.google.com/apis/library/meet.googleapis.com (또는 메뉴 > **API 및 서비스** > **라이브러리**에서 "Google Meet API" 검색)
   - 해당 API 페이지에서 **[사용(Enable)]** 버튼 클릭.
   - 사용 설정하지 않으면 API 호출 시 `403`.
2. **OAuth 동의화면(Google Auth Platform > 브랜딩) 구성**
   - 진입: https://console.cloud.google.com/auth/branding (메뉴 > **Google Auth Platform** > **브랜딩**)
   - **앱 이름(App name)** 입력 → **사용자 지원 이메일(User support email)** 선택 → **[다음(Next)]**.
   - **대상(Audience)** 단계에서 사용자 유형 **내부(Internal)** 또는 **외부(External)** 선택 → **[다음]**.
   - **연락처 정보(Contact Information)**: 알림 받을 **이메일 주소** 입력 → **[다음]**.
   - **Google API 서비스: 사용자 데이터 정책** 동의 체크 → **[계속(Continue)]** → **[만들기(Create)]**.
   - (외부 앱·테스트 모드) 테스트 사용자 추가: **대상(Audience)** 화면(https://console.cloud.google.com/auth/audience) > **테스트 사용자(Test users)** > **[사용자 추가(Add users)]** > 이메일 입력 > **[저장(Save)]** (≤100명).
   - (외부 앱) 스코프 등록: **데이터 액세스(Data Access)** 화면(https://console.cloud.google.com/auth/scopes) > **[스코프 추가 또는 삭제(Add or Remove Scopes)]** > 아래 "스코프 / 권한"의 스코프 선택 > **[저장]**. 민감/제한 스코프 포함 시 프로덕션 배포에는 Google OAuth 앱 검증 필요(아래 "전제조건" 참조).
3. **OAuth 클라이언트 ID 발급(데스크톱 앱 권장, 웹 앱도 가능)**
   - 진입: https://console.cloud.google.com/auth/clients (메뉴 > **Google Auth Platform** > **클라이언트**)
   - **[클라이언트 만들기(Create Client)]** 클릭.
   - **애플리케이션 유형(Application type)** 선택:
     - **데스크톱 앱(Desktop app)**: 이름 입력 → **[만들기(Create)]**. (설치형 OAuth 권장)
     - **웹 애플리케이션(Web application)**: 이름 입력 → **승인된 리디렉션 URI(Authorized redirect URIs)**에 콜백 URL 추가(자체 OAuth 플로우 사용 시) → **[만들기]**.
   - 발급 결과로 `client_id`와 `client_secret`을 받는다(데스크톱/웹 앱 모두 클라이언트 시크릿 JSON 다운로드 가능).
4. **사용자 동의 → refresh_token 확보**
   - 위 스코프로 사용자 동의 플로우를 실행해 authorization code를 받고, 표준 Google OAuth 토큰 교환으로 `refresh_token`을 확보한다.
   - 발급 절차 예시: Google OAuth Playground 또는 자체 OAuth 플로우(README 명시). 본 서버는 발급된 `refresh_token`으로 런타임에 `access_token`을 자동 갱신한다.
   - (콘솔 클릭 단위 세부 화면은 표준 Google OAuth 흐름을 따른다 — 아래 "미확인 / 주의" 참조.)
5. **환경변수에 주입** (아래 "환경변수 매핑" 표)

## 전제조건

- **한국식 게이트 없음**: 제휴 심사·사업자 심사·발신번호 사전등록·공동인증서 등 **해당 없음**(글로벌 Google API).
- **OAuth 앱 검증**: 민감/제한 스코프를 프로덕션 배포할 경우 Google OAuth 앱 검증 대상. 단, **테스트 모드(테스트 사용자 ≤100명)는 검증 없이 즉시 사용 가능**.
- **보안평가(security assessment)**: `drive.readonly`(Restricted)는 데이터 저장·전송 시 보안평가 대상 → 외부 배포 시 수일~수주 소요 가능. (본 1차 구현은 Drive 확장 제외 권고이므로 일반적으로 불필요.)
- **Workspace 에디션(데이터 가용성 전제)**: 녹화/전사/스마트노트 데이터는 관리자가 기능을 켜고 + 해당 Workspace 에디션이 그 기능을 포함(Business Standard+ 등) + 회의에서 실제 생성된 경우에만 존재. 이는 키 발급 게이트가 아니라 런타임 데이터 가용성 전제. 데이터 없으면 list가 빈 배열 반환(정상).

## 스코프 / 권한

| 스코프 | 민감도 | 용도 |
|--------|--------|------|
| `https://www.googleapis.com/auth/meetings.space.created` | 민감(Sensitive) | 가장 포괄적 — 스페이스 생성/수정/조회·conferenceRecords·transcripts·participants·smartNotes 읽기 |
| `https://www.googleapis.com/auth/meetings.space.readonly` | 민감(Sensitive) | 읽기 전용만 필요할 때(smartNotes 읽기도 이 스코프로 충분) |
| `https://www.googleapis.com/auth/meetings.space.settings` | 비민감(기본 검증) | 스페이스 설정만 수정할 때 |
| `https://www.googleapis.com/auth/drive.readonly` | 제한(Restricted) | (선택, 본 서버 미사용) Drive 녹화/전사 파일 메타·다운로드 확장 시 — 보안평가 대상 |
| `https://www.googleapis.com/auth/drive.meet.readonly` | 제한(Restricted) | (선택, 본 서버 미사용) 동일 — 1차 구현 제외 권고 |

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `GOOGLE_OAUTH_CLIENT_ID` | OAuth 클라이언트 ID 발급 결과(스텝 3) | 예 |
| `GOOGLE_OAUTH_CLIENT_SECRET` | OAuth 클라이언트 시크릿 발급 결과(스텝 3) | 예 |
| `GOOGLE_OAUTH_REFRESH_TOKEN` | 사용자 동의 토큰 교환 결과(스텝 4) — access_token 갱신용 | 예(토큰 파일과 택1) |
| `GOOGLE_OAUTH_TOKEN_PATH` | refresh_token 대신 토큰 JSON 파일 경로 | 선택(위와 택1) |

> 시크릿 하드코딩 절대 금지. `.env.example`에는 키 이름만 두고 값은 비운다. 실제 시크릿은 커밋 금지.

## 출처 (공식 URL)

- Meet API 사용 설정(콘솔): https://console.cloud.google.com/apis/library/meet.googleapis.com
- API 사용 설정 절차(메뉴: API 및 서비스 > 라이브러리 > 사용): https://developers.google.com/workspace/guides/enable-apis
- 사용자 인증정보 만들기(OAuth 클라이언트 ID·API 키·서비스계정 발급 화면 경로): https://developers.google.com/workspace/guides/create-credentials
- OAuth 동의화면(브랜딩·대상·데이터 액세스) 구성 절차: https://developers.google.com/workspace/guides/configure-oauth-consent
- 인증·인가 가이드(API 키 불가, 스코프 민감도): https://developers.google.com/workspace/meet/api/guides/authenticate-authorize
- REST API v2 레퍼런스: https://developers.google.com/workspace/meet/api/reference/rest/v2
- 공식 원격 MCP 서버 목록(Meet 미포함 근거): https://developers.google.com/workspace/guides/configure-mcp-servers
- 레이트리밋: https://developers.google.com/workspace/meet/api/guides/limits

## 미확인 / 주의

- **개인 Gmail에서 spaces.create 차단 여부**: 미확인. 공식 API 문서에 개인 Gmail 절대 차단 명시 없음. 하드 제약으로 단정하지 않으며, 401/403 발생 시 "Workspace 계정/기능 확인" 안내로 가드.
- **refresh_token 발급의 정확한 단계별 화면 절차**: 본 검증본은 "표준 Google OAuth" 및 OAuth Playground/자체 플로우 사용까지만 확인. 콘솔 클릭 단위 세부 화면은 본 문서 범위 밖(표준 Google OAuth 흐름을 따른다).
- **에러 바디 정확 스키마(google.rpc.Status)**: 미확인. HTTP 상태코드(401/403/404/429)는 표준 확인됨. 구현은 상태코드 기반 분기 + `error.message` 패스스루로 처리(스키마 단정 금지).
- **레이트리밋(참고, 발급과 별개)**: Read 6,000/min/project(600/user), Write 1,000/min/project(100/user), `spaces.create` 100/min/project(10/user). 초과 시 `429`.
- 자동 녹화/전사/스마트노트 옵션은 해당 Workspace 에디션에 기능이 포함되고 관리자가 활성화한 경우에만 실제 동작한다.
