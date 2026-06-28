# 키 발급 가이드 — TikTok for Business (tiktok_biz)

> 검증 출처: `_workspace/tiktok_biz/02_validation.md`(2026-06-26, doc-validator) + `mcp-servers/tiktok_biz/README.md`.
> 출처 URL 없는 사실은 적지 않는다. 불명확한 발급 디테일은 "미확인"으로 남긴다.

## 연동 유형 / 인증 방식
- **연동 유형**: TikTok for Business **Marketing API (v1.3)** — REST API. 공식 원격 MCP 커넥터 없음 → 로컬 빌드.
- **인증**: OAuth 2.0 (3-legged, 광고주 인가). 모든 API 요청 헤더에 **`Access-Token: <access_token>`** (하이픈 표기) 전달.
- **토큰 성격**: Marketing API 광고주 토큰은 **장기 토큰 — 만료 없음, refresh 불필요**. 폐기는 `/oauth2/revoke_token/`.
  - 출처: https://business-api.tiktok.com/portal/docs/obtain-a-long-term-access-token/v1.3 ("A long-term access token... The token does not expire.")
  - 주의: SDK 주석의 "creator access token valid 24 hours"는 Login Kit/크리에이터 토큰용이며 **Marketing API에는 적용되지 않는다.**
- **베이스 URL / 버전**: `https://business-api.tiktok.com/open_api/v1.3` (버전 경로 고정, v1.3 현행).
  - 출처: https://business-api.tiktok.com/portal/docs/marketing-api-authentication/v1.3

## 발급 위치 (콘솔 진입 / 메뉴 경로)
- **개발자 포털(콘솔) 진입 URL**: https://business-api.tiktok.com/portal — "TikTok For Business Developers" 포털. (앱 목록 직접 진입: https://business-api.tiktok.com/portal/apps)
  - 주의: business-api.tiktok.com 의 **Marketing API 전용 포털**이다. developers.tiktok.com(Login Kit/Display API, client_key/client_secret)과 **별개 플랫폼**이며 이 커넥터에는 해당 없음.
- **App ID / Secret 발급 위치**: 포털 로그인 후 **My Apps > App Detail > Basic Information**.
  - 출처(직접 인용): 인증 문서가 "To find the App ID, navigate to **My Apps > App Detail > Basic Information**" / "To find the Secret, navigate to **My Apps > App Detail > Basic Information**" 라고 명시. https://business-api.tiktok.com/portal/docs/marketing-api-authentication/v1.3
- **광고주 인가 URL 발급 위치**: **My Apps > (해당 앱 클릭) > Basic Information** 섹션의 **Advertiser authorization URL**.
  - 출처: https://business-api.tiktok.com/portal/docs/marketing-api-authorization/v1.3 ("navigate to My Apps, and click the app... In the Basic Information section, find the Advertiser authorization URL").
- **직링크 없음**: App ID/Secret·인가 URL은 **앱·계정별**이며 로그인·앱 선택을 거쳐야 하므로 개별 직링크가 없다. 콘솔 진입 URL(위) + 위 메뉴 경로로 도달한다.

## 발급 절차 (스텝)
공식 Get Started 5단계(계정 → 개발자 등록 → 앱 생성 → 인가 → 인증/토큰)를 따른다.
출처(워크플로): https://business-api.tiktok.com/portal/docs/marketing-api-get-started/v1.3

1. **TikTok For Business 계정 생성.**
2. **개발자 등록**(개발자 프로필 승인 필요 — 미승인 상태로 앱 생성 시 오류). 출처: https://ads.tiktok.com/marketing_api/docs?id=1738855176671234 (Register as a developer)
3. **개발자 앱 생성** — 포털(https://business-api.tiktok.com/portal) 로그인 후:
   - **Create an App** 버튼 클릭.
   - **App name**(개발자 앱 이름 — 모바일 앱 이름 아님) 입력.
   - **App description**(Intended Uses + Developer App Access Controls) 입력.
   - **Advertiser redirect URL** 입력 — **최대 10개 등록 가능(localhost 포함)**. 로컬 MCP 셋업이면 `http://localhost/...` 가능. redirect URL을 새로 추가하면 새 광고주 인가 URL을 생성할 수 있다.
   - **Scope of permission**에서 필요한 **권한 스코프를 선택**한다(아래 스코프 섹션). 선택한 스코프가 광고주 인가 화면에 표시된다.
   - **Confirm** 클릭 → 심사 제출(검토 **2~3 영업일**). 상태 확인: 우상단 버튼 → **My Apps > Verification Status**. 승인 후 인가 코드로 장기 토큰 획득 가능.
   - 추가 앱: **My Apps > Create App**(개발자당 최대 5개).
   - 출처: https://business-api.tiktok.com/portal/docs/create-a-developer-app/v1.3 (Create a developer app — Steps)
4. **광고주 인가**: **My Apps > (앱 클릭) > Basic Information**의 **Advertiser authorization URL**을 광고주에게 전달 → 광고주가 브라우저에서 승인.
   - 손으로 authorize URL을 조립하지 않는다 — 포털이 만든 URL을 광고주에게 전달한다.
   - 광고주 화면 흐름: 권한 목록 검토 → **Platform Service Agreement** 동의 → **Confirm** → **Send Code**(광고 계정 이메일로 인증코드 발송) → 코드 입력 → **Confirm**.
     - 같은 앱으로 이전 인가한 계정은 **48시간 내 재인가 시 추가 인증 불필요**(다른 앱 인가 시에는 재인증 필요).
   - 승인 후 redirect URL에 `auth_code`(쿼리; **유효 1시간·1회용**, `code`·`state`도 함께) 수신.
   - 출처: https://business-api.tiktok.com/portal/docs/marketing-api-authorization/v1.3 (Authorization — Steps)
5. **토큰 교환** (`POST /open_api/v1.3/oauth2/access_token/`, **Content-Type: application/json 전용**):
   ```bash
   curl -X POST https://business-api.tiktok.com/open_api/v1.3/oauth2/access_token/ \
     -H "Content-Type: application/json" \
     -d '{"app_id":"<APP_ID>","secret":"<APP_SECRET>","auth_code":"<AUTH_CODE>"}'
   ```
   - 응답: `data.access_token`(만료 없음), `data.advertiser_ids[]`(string[]), `data.scope[]`(number[] — 스코프 ID).
   - 출처: https://business-api.tiktok.com/portal/docs/marketing-api-authentication/v1.3 ; SDK `AuthenticationApi.js`
6. **저장**: `access_token`을 `TIKTOK_ACCESS_TOKEN` 환경변수로 저장. **refresh 로직 불필요.**
   - 토큰으로 접근 가능한 광고주 확인: `GET /open_api/v1.3/oauth2/advertiser/get/` (required: app_id, secret, Access_Token). 커넥터의 `list_advertisers` 도구가 이를 호출한다.

## 전제조건
- **TikTok For Business 계정 + 개발자 등록 + 개발자 앱**이 선행(위 절차 1~3). 이게 없으면 키 발급 자체가 불가.
- **쓰기 기능(캠페인/광고 생성·수정, 소재 업로드)은 앱 심사(app review) 승인 전제.** 조회·리포트만 쓰면 심사 의존도 낮음.
  - 출처: 02_validation 항목 34, 누락·리스크 §2.
- 발신번호 사전등록 등 통신 관련 전제: **해당 없음**(이 커넥터는 광고 Marketing API).
- 사업자 심사 절차 명세: **미확인**(아래 "미확인/주의" 참조).

## 스코프 / 권한
앱 생성 시 선택하며 광고주 인가 화면에 표시된다. (출처: https://ads.tiktok.com/marketing_api/docs?id=1753986142651394 Permission scope)

| ID | 권한 | 작업 분류 |
|----|------|----------|
| 1 | Ad Account Management | 조회/리포트 기본 |
| 2 | Ads Management | 캠페인/광고 조회·상태변경·생성/수정(쓰기) |
| 3 | Audience Management | 오디언스 조회 |
| 4 | Reporting | 리포트 |
| 5 | Measurement | — |
| 6 | Creative Management | 소재(쓰기) |
| 7 | App Management | — |
| 8 | Pixel Management | — |
| 9 | DPA Catalog Management | — |
| 10000 | Reach & Frequency | — |

- **조회/리포트 시작 권장**: scope 1·3·4 (+ 캠페인/광고 조회는 Ads 하위).
- **상태변경·생성/수정**: scope 2(Ads). **소재**: scope 6(Creative). 쓰기는 앱 심사 승인 전제.

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `TIKTOK_ACCESS_TOKEN` | 토큰 교환(절차 4) 응답의 `data.access_token`(장기 토큰, 만료 없음). 모든 API 요청의 `Access-Token` 헤더값 | 필수 |
| `TIKTOK_ADVERTISER_ID` | 토큰 교환 응답 `data.advertiser_ids[]` 중 대상 계정 ID. 도구 호출 시 미지정 기본값(편의) | 선택 |
| `TIKTOK_APP_ID` | 포털 **My Apps > App Detail > Basic Information**의 App ID. 토큰 교환/`list_advertisers` 셋업용 | 셋업 시 필요(런타임 조회 도구엔 미사용) |
| `TIKTOK_APP_SECRET` | 포털 **My Apps > App Detail > Basic Information**의 Secret. 토큰 교환/`list_advertisers` 셋업용 | 셋업 시 필요 |
| `TIKTOK_BASE_URL` | 베이스 URL 오버라이드(기본 `https://business-api.tiktok.com/open_api/v1.3` 사용 권장) | 선택 |

> 시크릿 실값을 `.mcp.json`에 쓰지 말 것 — `${VAR}` 확장 사용. 실값은 `.env`(gitignore) 또는 MCP 설정 `env`로 주입.

## 출처 (공식 URL)
- 개발자 포털(콘솔) 진입: https://business-api.tiktok.com/portal (앱 목록: https://business-api.tiktok.com/portal/apps)
- Get Started 워크플로(5단계): https://business-api.tiktok.com/portal/docs/marketing-api-get-started/v1.3
- 개발자 등록: https://ads.tiktok.com/marketing_api/docs?id=1738855176671234
- 개발자 앱 생성(Create an App·스코프·redirect·Confirm·심사): https://business-api.tiktok.com/portal/docs/create-a-developer-app/v1.3
- App ID/Secret 위치(My Apps > App Detail > Basic Information) + 인증/토큰 교환/에러 엔벨로프: https://business-api.tiktok.com/portal/docs/marketing-api-authentication/v1.3
- 장기 토큰(만료 없음): https://business-api.tiktok.com/portal/docs/obtain-a-long-term-access-token/v1.3
- 광고주 인가 절차(Advertiser authorization URL·메뉴 경로): https://business-api.tiktok.com/portal/docs/marketing-api-authorization/v1.3
- 권한 스코프 표: https://ads.tiktok.com/marketing_api/docs?id=1753986142651394
- v1.2 vs v1.3 비교(v1.3 현행): https://business-api.tiktok.com/portal/docs?id=1781891256601602
- 공식 SDK(경로/스키마 출처): https://github.com/tiktok/tiktok-business-api-sdk (`js_sdk/src/api/*`)

## 미확인 / 주의
- **앱 심사(쓰기 기능) SLA·심사 기준·basic/advanced 등급 승인 절차**: 비공개 — **미확인.** 쓰기 도구는 권한 오류 `code`로 가드한다(Display/Content Posting 심사와는 별개).
- **레이트리밋 정확 수치(QPS·일일 한도)**: 공개 문서에 수치 없음 — **미확인.** 응답 헤더 `X-Tt-Ads-Throttle` 관찰 + 지수 백오프로 처리. 수치 하드코딩 금지.
- **엔드포인트별 page_size 최대값**: PageInfo 모델은 확인(page/page_size 기본 1/10)되나 엔드포인트별 상한은 비공개 — **미확인.** 응답 `page_info` 기반 순회로 처리.
- **에러 판정 주의**: 앱레벨 오류도 HTTP 200으로 온다. 성공 판정은 HTTP status가 아니라 **응답 본문 `code === 0`** 기준.
- **advertiser_id 주의**: 1개 토큰이 여러 `advertiser_ids`를 커버 → 토큰만으로 대상 계정이 결정되지 않음. 도구 호출 시 `advertiser_id`(또는 `TIKTOK_ADVERTISER_ID`) 지정 필요. 먼저 `list_advertisers`로 확인.
