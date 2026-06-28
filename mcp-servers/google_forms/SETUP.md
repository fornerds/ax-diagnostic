# 키 발급 가이드 — 구글폼 (google_forms)

> 출처: `_workspace/google_forms/02_validation.md`(검증일 2026-06-26), `mcp-servers/google_forms/README.md`
> 원칙: 검증된 사실만 기재. 미확인 항목은 "미확인 / 주의"에 명시.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 MCP 서버(stdio)에서 Google Forms API v1(REST) 직접 호출. 베이스 URL `https://forms.googleapis.com`, 버전 v1(stable).
- **인증 방식**: OAuth 2.0 Authorization Code Grant(사용자 동의) + Refresh Token.
  - API Key 단독으로는 사용 불가(사용자 동의 + 리프레시 토큰 필요).
  - MCP 서버는 `client_id` + `client_secret` + `refresh_token`으로 access token을 자동 갱신하여 호출한다(`401` 시 refresh 후 1회 재시도).
  - 출처: https://developers.google.com/workspace/forms/api/quickstart/python

## 발급 절차 (스텝)

> 콘솔 UI 주의: 구글은 기존 "OAuth consent screen" / "Credentials(사용자 인증 정보)" 페이지를 **Google Auth platform**(Menu → Google Auth platform → Branding / Clients)으로 통합했다. 아래 경로는 현행(2026-06) 공식 문서 기준.

1. **GCP Console 접속 후 프로젝트 선택**: https://console.cloud.google.com/ → 상단 프로젝트 선택기에서 대상 프로젝트 선택(없으면 생성).
2. **Google Forms API 사용 설정(Enable)**:
   - **직링크(원클릭 활성화)**: https://console.cloud.google.com/flows/enableapi?apiid=forms.googleapis.com → 프로젝트 확인 후 **[사용]/[Enable]** 클릭.
   - 또는 메뉴 경로: **API 및 서비스(APIs & Services) → 라이브러리(Library)** (직링크 https://console.cloud.google.com/apis/library) → "Google Forms API" 검색 → **[사용]/[Enable]**.
3. **OAuth 동의화면(브랜딩) 구성** — 메뉴: **Google Auth platform → 브랜딩(Branding)** (직링크 https://console.cloud.google.com/auth/branding):
   - 앱 이름 / 사용자 지원 이메일 / 연락처 이메일 입력 → 사용자 데이터 정책 동의 → **[만들기/Create]**.
   - **대상(Audience)**: 자사 Workspace 내부용이면 **Internal** 선택 → Google 앱 심사(verification) 불필요.
   - 외부 사용자 대상이면 Audience를 External로 두고 **Google Auth platform → 대상(Audience)** 화면에서 **테스트 사용자(Test users) 등록**, 또는 정식 배포 시 **민감 스코프 verification** 필요.
4. **OAuth 클라이언트 ID 발급 — 유형 "데스크톱 앱(Desktop app)"** — 메뉴: **Google Auth platform → 클라이언트(Clients)** (직링크 https://console.cloud.google.com/auth/clients):
   - **[클라이언트 만들기 / Create Client]** 클릭 → **애플리케이션 유형(Application type) → 데스크톱 앱(Desktop app)** 선택 → 이름 입력 → **[만들기/Create]**.
   - 생성 후 다이얼로그에서 **JSON 다운로드(Download JSON)** → `credentials.json`로 저장. 여기서 `client_id` / `client_secret` 확보.
5. **최초 1회 동의로 refresh token 획득**: authorization-code 플로우를 1회 수행해 위 스코프에 동의하고 **refresh token**을 캡처한다(콘솔 단일 버튼으로는 refresh token이 발급되지 않음 — 동의 플로우 1회 실행 필요).
   - 방법 예: [Forms API Python 퀵스타트](https://developers.google.com/workspace/forms/api/quickstart/python)(`InstalledAppFlow`로 동의 → `token.json`에 refresh_token 저장), 또는 본인 클라이언트로 [OAuth 2.0 Playground](https://developers.google.com/oauthplayground/) 사용(우측 톱니바퀴 → "Use your own OAuth credentials"에 client_id/secret 입력 → 스코프 동의 → Exchange authorization code for tokens → refresh_token 복사).
6. **환경변수 주입**: `client_id` / `client_secret` / `refresh_token`을 환경변수로 설정(아래 매핑 표). 하드코딩 금지.

## 전제조건

- **유료 플랜**: 별도 필요 없음 — API 쿼터는 현재 기본 무료(쿼터 초과분 과금은 "2026년 후반 예정"). 출처: https://developers.google.com/workspace/forms/api/limits
- **사업자 심사 / 발신번호 사전등록 등**: 없음(해당 없는 한국식 통신 규제 절차 없음).
- **OAuth 앱 심사(verification)**: `forms.body`는 **민감(sensitive) 스코프**. 외부 일반 사용자 대상 배포 시에만 Google 앱 심사 필요(개인정보처리방침·도메인 소유확인(Search Console)·데모영상(YouTube 비공개)·정당화, 최대 10일). **Internal 사용자 유형 / 테스트 사용자 / 개인용은 면제** → 즉시 사용 가능. 이는 코드 문제가 아니라 운영/배포 전제. 출처: https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification
- **게시 전제(2026-06-30 발효)**: 2026-06-30 **이후** API로 생성한 폼은 **기본 미게시(unpublished)** 상태라 응답을 받지 못한다. `create_form` 직후 `set_publish_settings`(is_published=true, is_accepting_responses=true) 호출이 필요. 출처: https://developers.google.com/workspace/forms/api/guides/publish-form

## 스코프 / 권한

- **권장 단일 세트(최소 권한)**: `https://www.googleapis.com/auth/forms.body` + `https://www.googleapis.com/auth/forms.responses.readonly`
  - 생성·수정·게시·구조조회·응답조회를 모두 커버.
- 작업별 정확한 스코프(출처: 각 메서드 REST 레퍼런스):
  - 생성/일괄수정/게시(forms.create / batchUpdate / setPublishSettings): `forms.body` (또는 `drive` / `drive.file`)
  - 폼 구조 조회(forms.get): `forms.body.readonly`(쓰기 병행 시 `forms.body`로 통합 가능; 그 외 `drive` / `drive.file` / `drive.readonly`)
  - 응답 조회(responses.get / list): `forms.responses.readonly`(또는 `drive` / `drive.file`)
- `drive` 계열은 더 광범위하므로 불필요 시 사용하지 않음(최소 권한).

## 환경변수 매핑

| 변수명 | 값을 어디서 | 필수 |
|--------|-------------|------|
| `GOOGLE_CLIENT_ID` | GCP Console에서 발급한 데스크톱 OAuth 클라이언트 ID | 필수 |
| `GOOGLE_CLIENT_SECRET` | 동일 OAuth 클라이언트의 Secret | 필수 |
| `GOOGLE_REFRESH_TOKEN` | 최초 1회 동의(authorization-code 플로우)로 캡처한 리프레시 토큰 | 필수 |
| `GOOGLE_FORMS_SCOPES` | (선택) 공백 구분 스코프 오버라이드. 기본값: `https://www.googleapis.com/auth/forms.body https://www.googleapis.com/auth/forms.responses.readonly` | 선택 |

> 모든 시크릿은 env 주입. 하드코딩 금지. `.env.example`에는 키 이름만 둔다.

## 출처 (공식 URL)

- Forms API 퀵스타트(인증 절차·발급 메뉴 경로): https://developers.google.com/workspace/forms/api/quickstart/python
- OAuth 클라이언트/자격증명 생성 가이드(메뉴 경로·버튼명): https://developers.google.com/workspace/guides/create-credentials
- Forms API 원클릭 활성화 직링크: https://console.cloud.google.com/flows/enableapi?apiid=forms.googleapis.com
- API 라이브러리: https://console.cloud.google.com/apis/library
- OAuth 동의화면(브랜딩): https://console.cloud.google.com/auth/branding
- OAuth 클라이언트(Clients): https://console.cloud.google.com/auth/clients
- Forms API REST 레퍼런스(베이스 URL·버전): https://developers.google.com/workspace/forms/api/reference/rest
- forms.create(스코프): https://developers.google.com/workspace/forms/api/reference/rest/v1/forms/create
- forms.get(스코프): https://developers.google.com/workspace/forms/api/reference/rest/v1/forms/get
- forms.batchUpdate(스코프): https://developers.google.com/workspace/forms/api/reference/rest/v1/forms/batchUpdate
- forms.setPublishSettings(스코프): https://developers.google.com/workspace/forms/api/reference/rest/v1/forms/setPublishSettings
- responses.list / get(스코프): https://developers.google.com/workspace/forms/api/reference/rest/v1/forms.responses/list
- 게시 모델 / 2026-06-30 기본 미게시: https://developers.google.com/workspace/forms/api/guides/publish-form
- OAuth 스코프 목록(민감 여부): https://developers.google.com/identity/protocols/oauth2/scopes
- 민감 스코프 verification: https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification
- 쿼터·과금: https://developers.google.com/workspace/forms/api/limits
- 인증·인가 트러블슈팅(401/403): https://developers.google.com/workspace/forms/api/troubleshoot-authentication-authorization

## 미확인 / 주의

- **refresh token: 콘솔 단일 화면 없음(확인됨)**: 클라이언트 ID 발급(메뉴 경로·버튼명)까지는 공식 문서로 확정. refresh token은 **콘솔 단일 버튼으로는 발급되지 않으며** authorization-code 동의 플로우 1회 실행이 필요(퀵스타트의 `InstalledAppFlow` 또는 OAuth Playground "Use your own OAuth credentials"). "발급 절차" 5단계에 구체 경로 반영함. (이는 누락이 아니라 OAuth 사양상 정상 — 직링크 부재가 정상)
- **Forms 전용 세부 에러코드표 미확인**: 공식 문서에서 케이스별 에러코드표 미발견. 표준 Google 포맷(`{error:{code,message,status}}`) + 429 지수 백오프로 처리하고, 401(토큰)·403(스코프/미게시)·404(없는 ID) 메시지를 그대로 전달.
- **forms.watches(알림) 제외**: Cloud Pub/Sub 토픽 + 서비스계정 publish 권한 + 콜백 수신처가 필요해 stdio MCP 단독으로는 수신 불가 → 초기 계약에서 제외(별도 인프라 전제).
- **2026-06-30 게시 변경 시점 주의**: 이 날짜 이후 생성 폼은 기본 미게시. 응답이 안 모이면 `set_publish_settings`로 게시했는지 먼저 확인.
