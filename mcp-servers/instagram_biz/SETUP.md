# 키 발급 가이드 — Instagram Business (instagram_biz)

> 출처: `_workspace/instagram_biz/02_validation.md`(검증일 2026-06-26) 및 `README.md`. 아래는 출처가 확인된 사실만 정리한다. 기준 Graph API 버전 = **v25.0**.

## 연동 유형 / 인증 방식

- **연동 유형**: REST(Instagram Platform / Graph API, GraphQL 아님). 베이스 호스트 `https://graph.instagram.com/v25.0`.
- **인증 방식**: OAuth 2.0 Authorization Code — **Instagram Login 단일 구성**.
- **런타임 인증**: 모든 API 호출에 장기(60일) 사용자 액세스 토큰을 `access_token` 쿼리 파라미터로 부착.
- 출처: https://developers.facebook.com/docs/instagram-platform/content-publishing/ , https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/business-login/

## 발급 절차 (스텝)

**발급 위치 (콘솔)**: Meta 앱 대시보드(Meta App Dashboard).
- 콘솔 진입 URL: **https://developers.facebook.com/apps** (앱 목록 화면)
- **개별 직링크 없음** — App ID/Secret·토큰 생성 화면은 앱별로 동적 생성되므로(`developers.facebook.com/apps/<APP_ID>/...`) 고정 직링크가 존재하지 않는다. 아래 메뉴 경로로 진입한다.
- 공식 가이드: https://developers.facebook.com/docs/instagram-platform/create-an-instagram-app/

### A. 콘솔 발급 절차 (메뉴 > 버튼 클릭 경로 — 앱 생성 / 자격증명 위치)

1. **Meta 앱 생성**: https://developers.facebook.com/apps 접속 → 우측 상단 **`앱 만들기`(Create App)** 클릭. (기존 앱 대시보드 안이면 좌측 상단 드롭다운 → **`새 앱 만들기`(Create New App)**)
   - 사용 사례(use case) 선택 화면이 나오면 선택 없이 진행 가능(앱은 "사용 사례 없이" 생성 가능). 안내에 따라 앱 이름 입력 후 생성 완료.
   - 출처: https://developers.facebook.com/docs/development/create-an-app/
2. **Instagram 제품 추가**: 좌측 메뉴 **`제품 추가`(Add a Product / Add Product)** → 목록에서 **Instagram** 의 **`설정`(Set Up)** 클릭 → **`Instagram 비즈니스 로그인으로 API 설정`(API setup with Instagram Login)** 선택.
   - 출처: https://developers.facebook.com/docs/instagram-platform/create-an-instagram-app/
3. **App ID / App Secret 위치 (발급 화면)**: 좌측 메뉴 **`Instagram` > `Instagram 비즈니스 로그인으로 API 설정`(Instagram > API setup with Instagram Login)** 페이지. 이 화면에 **앱 이름·Instagram app ID·Instagram app secret** 이 표시된다.
   - 주의: Business Login 구성 앱은 **이 페이지의 Instagram app ID/secret** 를 사용한다(좌측 **`앱 설정` > `기본`**(App settings > Basic)에 표시되는 것은 *Facebook* app ID/secret 으로 별개). 본 커넥터는 **Instagram** app ID/secret 를 사용한다.
   - 이 화면에서 **`OAuth 리디렉션 URI`(redirect URI)** 등록과 스코프(권한) 구성도 함께 수행한다.
   - 출처: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/get-started/
4. **(콘솔 직접 토큰 발급 — 권장)**: 같은 페이지에서 대상 Instagram 계정 옆 **`토큰 생성`(Generate Token)** 버튼 클릭 → Instagram 인증 완료 → **장기(60일) 사용자 액세스 토큰** 이 바로 발급된다. 이 값을 `IG_ACCESS_TOKEN` 으로 사용하면 아래 B(OAuth 수동 교환)를 생략할 수 있다.
   - 출처: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/get-started/

### B. (선택) OAuth 수동 토큰 흐름 — 콘솔 `토큰 생성` 대신 직접 교환 시

위 A-4의 콘솔 `토큰 생성`을 쓰지 않고 자체 서버에서 토큰을 교환할 경우 아래 검증된 OAuth 흐름을 따른다(App ID/Secret 은 A-3 화면 값 사용).

2. **인가창 호출 → `code` 수신**:
   `GET https://www.instagram.com/oauth/authorize?client_id=<IG_APP_ID>&redirect_uri=<URI>&response_type=code&scope=<공백구분 스코프>`
   - 주의: 인가창 호스트는 `www.instagram.com`(단기교환 호스트 `api.instagram.com`과 다름).
   - 출처: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/business-login/
3. **단기 토큰(1시간) 교환**:
   `POST https://api.instagram.com/oauth/access_token` (client_id, client_secret, grant_type=authorization_code, redirect_uri, code) → 단기 토큰 + `user_id` 수신.
   - 출처: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/business-login/
4. **장기 토큰(60일) 교환**(서버사이드 전용, client_secret 포함):
   `GET https://graph.instagram.com/access_token?grant_type=ig_exchange_token&client_secret=<SECRET>&access_token=<단기토큰>`
   - 출처: https://developers.facebook.com/docs/instagram-platform/reference/access_token/
5. **(선택) 토큰 갱신**(24시간 경과 후∼만료 전, 갱신 시 60일 재연장):
   `GET https://graph.instagram.com/refresh_access_token?grant_type=ig_refresh_token&access_token=<장기토큰>`
   - 출처: https://developers.facebook.com/docs/instagram-platform/reference/refresh_access_token/
6. **환경변수 설정**: 단계 4의 **장기 토큰** → `IG_ACCESS_TOKEN`, 단계 3에서 받은 `user_id` → `IG_USER_ID`.

## 전제조건

1. **Meta 앱 + Instagram 제품 + Business Login 구성** 선행(위 1단계). 출처: business-login 문서.
2. **대상 계정은 Instagram 프로페셔널(비즈니스/크리에이터) 계정** 이어야 함. 개인 계정은 API 전면 불가. 출처: https://developers.facebook.com/docs/instagram-platform/reference/instagram-media/
3. **App Review(Advanced Access) + Business Verification(사업자 심사)**: 타인(고객) 계정 운영 시 각 `instagram_business_*` 스코프의 advanced access 검수 통과 필요(수일∼수주, 한국 사업자도 영문 글로벌 콘솔에서 진행).
   - **PoC/개발/테스트는 앱 role 사용자(본인 계정)에 한해 검수 없이 즉시 가능.** 외부 배포 전 검수 필수.
   - 출처: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/business-login/
4. **미디어 호스팅(발행 시)**: `image_url`/`video_url` 은 Meta가 직접 fetch 하는 **공개 접근 URL** 이어야 함(내부망/인증 URL 불가, S3 등 공개 스토리지 전제). 이미지는 **JPEG 한정**.
   - 출처: https://developers.facebook.com/docs/instagram-platform/content-publishing/
5. **발신번호 사전등록**: 해당 없음(SMS 커넥터 아님).

## 스코프 / 권한

| 스코프 | 용도(도구) | 출처 |
|--------|-----------|------|
| `instagram_business_basic` | 프로필·미디어·해시태그 읽기 | https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login |
| `instagram_business_content_publish` | 게시물 발행 | https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login |
| `instagram_business_manage_comments` | 댓글 읽기/작성/모더레이션 | https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login |
| `instagram_business_manage_insights` | 인사이트(계정·미디어) | https://developers.facebook.com/docs/instagram-platform/insights/ |

> 인가창의 `scope` 파라미터에 필요한 스코프를 **공백 구분**으로 나열한다.
> 폐기 주의: 구 스코프(`business_basic` 등 Instagram Login 구버전)는 **2025-01-27 폐기** → `instagram_business_*` 사용 필수.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `IG_ACCESS_TOKEN` | 발급 절차 4단계의 **장기(60일) 사용자 액세스 토큰** | 필수 |
| `IG_USER_ID` | 발급 절차 3단계 단기 토큰 응답의 `user_id`(Instagram 프로페셔널 계정 ID) | 필수 |
| `IG_API_VERSION` | Graph API 버전. 미설정 시 기본 `v25.0` | 선택 |
| `IG_APP_SECRET` | 앱 대시보드의 Instagram **app secret**(토큰 교환/갱신 `refresh_access_token` 도구 사용 시) | 선택 |

## 출처 (공식 URL)

- 콘솔 진입(앱 목록): https://developers.facebook.com/apps
- 앱 생성 절차(앱 만들기·사용 사례): https://developers.facebook.com/docs/development/create-an-app/
- Instagram 앱 생성·제품 추가·API setup with Instagram Login(앱 ID/secret 위치): https://developers.facebook.com/docs/instagram-platform/create-an-instagram-app/
- get-started(토큰 생성 버튼·App ID/secret 표시 화면): https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/get-started/
- 콘텐츠 발행 / 베이스 호스트 / JPEG·공개 URL 제약: https://developers.facebook.com/docs/instagram-platform/content-publishing/
- Business Login(앱·app id/secret·인가창·단기토큰): https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/business-login/
- 장기 토큰 교환: https://developers.facebook.com/docs/instagram-platform/reference/access_token/
- 토큰 갱신: https://developers.facebook.com/docs/instagram-platform/reference/refresh_access_token/
- 스코프 개요: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login
- 인사이트 스코프: https://developers.facebook.com/docs/instagram-platform/insights/
- Graph API 버전(v25.0, 2026-02-18): https://developers.facebook.com/docs/graph-api/changelog/
- 프로페셔널 계정 요건: https://developers.facebook.com/docs/instagram-platform/reference/instagram-media/

## 미확인 / 주의

- **콘솔 클릭 경로**: 공식 문서로 메뉴 라벨·버튼명 확정 완료(위 "발급 절차 A" 참조). App ID/Secret 은 `Instagram > Instagram 비즈니스 로그인으로 API 설정` 화면에 표시되며, 같은 화면의 `토큰 생성`(Generate Token) 버튼으로 장기 토큰을 콘솔에서 직접 발급 가능. 콘솔이 앱별로 동적 생성되어 **개별 직링크는 없음**(진입 URL = developers.facebook.com/apps + 위 메뉴 경로).
- **레이트리밋**: "시간당 200콜"은 낡은 값. 현재 모델 = `24시간 = 4800 × Impressions`(임프레션 기반). 저활동 계정은 한도가 작아 빨리 소진될 수 있음. 출처: https://developers.facebook.com/docs/graph-api/overview/rate-limiting/
- **발행 한도**: 100/24h(캐러셀=1개로 계산). 출처: content-publishing 문서.
- **토큰 만료**: 장기 토큰도 60일 만료. v1 서버는 env 토큰 사용 모델로 자동 갱신하지 않음 — 주기적으로 `refresh_access_token` 도구로 갱신하거나 재발급 필요.
- **DM/메시징, 멘션(mentioned_media)**: 본 커넥터 v1 제외(권한·검수 부담 / Instagram Login 가용성 미확인).
