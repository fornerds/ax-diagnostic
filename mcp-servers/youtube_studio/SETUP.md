# 키 발급 가이드 — 유튜브 스튜디오 (youtube_studio)

> 출처: `_workspace/youtube_studio/02_validation.md`(검증 기준일 2026-06-26) 및 README. 추측 없이 검증된 사실·공식 출처만 기재. 확정 못 한 항목은 "미확인"으로 남김.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 MCP 서버 (stdio). 공식 원격 MCP 커넥터 없음 — Google 계열 Anthropic 커넥터는 Gmail/Calendar/Drive만 존재, YouTube는 없음.
- **인증 방식**: OAuth 2.0 (Authorization Code + refresh token) 가 기본. 공개 데이터 read-only 보조용으로 API Key 선택 사용 가능.
- **핵심 제약 — Service Account 불가**: YouTube Data/Analytics API는 서비스 계정 인증을 지원하지 않는다(`NoLinkedYouTubeAccount`). 따라서 **사람이 한 번 OAuth 동의를 거쳐 refresh token을 사전 발급**받아 서버에 주입해야 한다. 순수 무인 자동화는 불가.
- **호출 방식**: 서버는 보관된 refresh token으로 access token을 갱신하고 `Authorization: Bearer <access_token>` 헤더로 호출.

## 발급 절차 (스텝)

> **콘솔 진입 URL은 공개이나, 발급된 키/클라이언트는 Google 계정·프로젝트별로 귀속되어 개별 직링크가 없다.** 아래는 "콘솔 진입 URL + 현행 UI 메뉴 경로 + 버튼명"으로 명시한다. (모든 콘솔 URL은 끝에 `?project=<PROJECT_ID>`를 붙이면 특정 프로젝트로 바로 진입 가능.)

1. **Google Cloud Console 프로젝트 생성**
   - 진입: https://console.cloud.google.com/
   - 상단 프로젝트 선택 드롭다운(헤더의 프로젝트명 영역) > **새 프로젝트(New Project)** > 이름 입력 > **만들기(Create)**. 이후 같은 드롭다운에서 해당 프로젝트 선택.

2. **API 활성화(Enable)**
   - 진입: **API 라이브러리** https://console.cloud.google.com/apis/library (또는 좌측 탐색 메뉴 ☰ > **API 및 서비스(APIs & Services)** > **라이브러리(Library)**).
   - 검색창에 `YouTube Data API v3` 입력 > 결과 항목 클릭 > **사용(ENABLE)** 버튼 클릭.
   - 분석 도구(`get_analytics_report`) 사용 시 같은 라이브러리에서 `YouTube Analytics API` 검색 후 동일하게 **사용(ENABLE)**.
   - (활성화된 API 확인: ☰ > API 및 서비스 > **사용 설정된 API 및 서비스(Enabled APIs & services)**.)

3. **OAuth 동의 화면 구성**
   - 진입: ☰ > **API 및 서비스(APIs & Services)** > **OAuth 동의 화면(OAuth consent screen)** (현행 UI에서는 **Google Auth Platform** 영역으로 통합됨). 콘솔 URL: https://console.cloud.google.com/auth/overview (브랜딩/대상은 https://console.cloud.google.com/auth/branding ).
   - 앱 이름·지원 이메일 등 앱 정보를 입력하고, **대상(Audience)**을 내부/외부 중 선택(개인 계정은 외부=External). 사용할 스코프를 등록(아래 "스코프 / 권한" 참조). 내부 소수 사용이면 테스트 사용자에 본인 계정 추가.

4. **OAuth 2.0 클라이언트 ID 발급**
   - 진입: ☰ > **API 및 서비스(APIs & Services)** > **사용자 인증정보(Credentials)** (콘솔 URL: https://console.cloud.google.com/apis/credentials ). 현행 UI의 클라이언트 직접 경로: **Google Auth Platform > 클라이언트(Clients)** https://console.cloud.google.com/auth/clients .
   - **사용자 인증정보 만들기(Create Credentials 또는 Create Client)** 클릭 > **OAuth 클라이언트 ID(OAuth client ID)** 선택.
   - **애플리케이션 유형(Application type)** 선택(초기 refresh token 발급 스크립트가 로컬/CLI면 **데스크톱 앱(Desktop app)**, 웹 콜백이면 **웹 애플리케이션(Web application)** + **승인된 리디렉션 URI(Authorized redirect URIs)** 등록) > **이름(Name)** 입력 > **만들기(Create)**.
   - 발급된 **클라이언트 ID/보안 비밀(client ID / client secret)**를 `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` 으로 사용(JSON 다운로드 가능).

5. **인가(consent)로 authorization code 획득**: 필요 scope를 포함해 `https://accounts.google.com/o/oauth2/v2/auth` 로 인가 요청. 이때 `access_type=offline` 및 `prompt=consent`를 지정해야 refresh token이 발급됨.

6. **토큰 교환**: 받은 authorization code를 `https://oauth2.googleapis.com/token` 에 POST → **refresh token** + access token 수령. refresh token을 `GOOGLE_REFRESH_TOKEN` 으로 보관.

7. **(선택) API Key 발급**: 공개 데이터 read-only 보조 호출이 필요하면 ☰ > API 및 서비스 > **사용자 인증정보(Credentials)** > **사용자 인증정보 만들기(Create Credentials)** > **API 키(API key)** 선택 → 발급된 키를 `YOUTUBE_API_KEY` 로 사용(프로덕션 전 **키 제한(Restrict key)** 권장).

> 출처: https://developers.google.com/youtube/v3/getting-started , https://developers.google.com/youtube/v3/guides/auth/server-side-web-apps , https://developers.google.com/youtube/v3/quickstart/python , https://developers.google.com/youtube/registering_an_application , https://docs.cloud.google.com/apis/docs/enable-disable-apis , https://developers.google.com/workspace/guides/create-credentials

## 전제조건

- **사업자 심사 / 발신번호 사전등록 / 유료 플랜: 없음.** 개인 Google 계정 + Cloud Console로 즉시 발급 가능 (사업자 심사 불필요).
  - 출처: https://developers.google.com/youtube/v3/getting-started
- **업로드 도구 사용 시 주의(본 MVP에는 업로드 도구 미포함)**: 2020-07-28 이후 생성된 미검증 프로젝트의 `videos.insert` 업로드는 영상이 private로 강제됨. 정상 공개하려면 audit/검증 통과 필요.
  - 출처: https://developers.google.com/youtube/v3/docs/videos/insert
- **외부 사용자 배포 시**: 민감/제한 스코프(`youtube`, `youtube.force-ssl` 등)를 외부 사용자에게 배포하려면 Google OAuth 앱 검증 필요. 미검증(테스트) 모드는 테스트 사용자 100명 제한 및 refresh token이 7일마다 만료될 수 있음. **본인/내부 소수 사용이면 테스트 모드로 충분.**

## 스코프 / 권한

도구 집합에 따라 필요한 스코프(OAuth 동의 화면에 등록):

| 용도 | 스코프 |
|------|--------|
| 읽기 전반(채널/영상/재생목록/댓글 조회) | `https://www.googleapis.com/auth/youtube.readonly` |
| 영상 메타 수정 · 재생목록 추가 | `https://www.googleapis.com/auth/youtube` |
| 댓글 작성 · 답글 · 모더레이션 | `https://www.googleapis.com/auth/youtube.force-ssl` |
| 분석 리포트 | `https://www.googleapis.com/auth/yt-analytics.readonly` |
| 수익 지표 포함 분석 | `https://www.googleapis.com/auth/yt-analytics-monetary.readonly` |

- **권장(검증 부담 최소화)**: read 중심 MVP는 `youtube.readonly` + `yt-analytics.readonly` 만으로 충분. 쓰기 도구(댓글/재생목록) 포함 시 `youtube.force-ssl` 까지.
- 출처: https://developers.google.com/identity/protocols/oauth2/scopes

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `GOOGLE_CLIENT_ID` | API 및 서비스 > 사용자 인증정보 > 사용자 인증정보 만들기 > OAuth 클라이언트 ID (4단계) | 필수 |
| `GOOGLE_CLIENT_SECRET` | 위와 동일 — OAuth 클라이언트 발급 시 함께 나오는 client secret (4단계) | 필수 |
| `GOOGLE_REFRESH_TOKEN` | 토큰 교환 단계(6단계)에서 수령한 refresh token | 필수 |
| `YOUTUBE_API_KEY` | API 및 서비스 > 사용자 인증정보 > 사용자 인증정보 만들기 > API 키 (7단계, 선택) | 선택 |
| `GOOGLE_OAUTH_REDIRECT_URI` | 초기 토큰 발급 스크립트용 redirect URI(OAuth 클라이언트에 등록한 값) | 선택(초기 토큰 발급 시) |

> 실 시크릿 값은 절대 커밋 금지. `.env.example` 복사 후 `.env` 에 채우고, `.mcp.json` 에는 `${VAR}` 환경변수 확장을 사용.

## 출처 (공식 URL)

- 시작하기 / 쿼터 / 사업자 심사 불필요: https://developers.google.com/youtube/v3/getting-started
- OAuth (server-side web apps, refresh token, access_type=offline): https://developers.google.com/youtube/v3/guides/auth/server-side-web-apps
- 인증 헤더 / Service Account 미지원: https://developers.google.com/youtube/v3/guides/authentication
- OAuth 스코프 목록: https://developers.google.com/identity/protocols/oauth2/scopes
- 미검증 프로젝트 업로드 잠금: https://developers.google.com/youtube/v3/docs/videos/insert
- Analytics 리포트 쿼리: https://developers.google.com/youtube/analytics/reference/reports/query
- API 활성화 절차(콘솔 메뉴/버튼): https://docs.cloud.google.com/apis/docs/enable-disable-apis
- OAuth 클라이언트/동의 화면 생성 절차(콘솔 메뉴/버튼): https://developers.google.com/workspace/guides/create-credentials
- 애플리케이션 등록(API 키/OAuth 클라이언트): https://developers.google.com/youtube/registering_an_application
- 빠른 시작(API 라이브러리·사용자 인증정보 패널): https://developers.google.com/youtube/v3/quickstart/python
- Cloud Console: https://console.cloud.google.com/
- API 라이브러리(직접): https://console.cloud.google.com/apis/library
- 사용자 인증정보(직접): https://console.cloud.google.com/apis/credentials
- OAuth 클라이언트 / 동의 화면(현행 Google Auth Platform): https://console.cloud.google.com/auth/clients , https://console.cloud.google.com/auth/overview

## 미확인 / 주의

- **OAuth 앱 검증 정책의 현재 소요기간 / 보안평가 요건**(민감 vs 제한 스코프 구분): 정책 페이지가 수시 변동하여 확정 못 함 → **미확인**. 키 발급 자체에는 영향 없음(발급 가능). 외부 배포 시 리드타임 안내용으로만 참고.
- **Cloud Console 메뉴 경로**: 발급 화면의 메뉴/버튼 경로는 공식 문서로 확정함(위 "발급 절차" 참조 — API 라이브러리 ENABLE, API 및 서비스 > 사용자 인증정보 > 사용자 인증정보 만들기 > OAuth 클라이언트 ID / API 키, Google Auth Platform의 OAuth 동의 화면). 다만 Google이 OAuth 동의 화면을 **Google Auth Platform**으로 통합·개편 중이라 일부 라벨(동의 화면 vs Auth Platform > 브랜딩/대상/클라이언트)이 계정에 따라 다르게 보일 수 있음 — 양쪽 경로/URL을 병기함. 개별 키·클라이언트는 계정·프로젝트별 귀속이라 **직링크 없음**, 콘솔 진입 URL + 메뉴 경로로 안내.
- **스튜디오 고유 기능 일부는 공개 API 부재**: 수익화 설정 변경, 커뮤니티 탭(게시물) 작성, 일부 상세 도달/노출 분석은 공개 API에 대응 엔드포인트가 없어 본 MCP에서 미커버. 이 가이드는 발급 가능한 OAuth/API Key 인증에 한함.
- **쿼터 구조**: 일반 일일 10,000 units와 별개로 search/upload는 별도 100회/일 버킷. 쿼터 리셋은 midnight PT(태평양 시간).
