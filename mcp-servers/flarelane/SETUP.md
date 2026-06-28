# 키 발급 가이드 — 플레어레인(FlareLane) (flarelane)

> **주의: 아래 발급 경로(메뉴/버튼)는 미확인 — 정확하지 않을 수 있습니다.** 발급 콘솔이 로그인 게이트/계정별이라 외부에서 화면을 확인하지 못했습니다. 실제 UI와 다를 수 있으니 공식 콘솔에서 직접 확인하세요. (추적: `mcp-servers/_SETUP_GAPS.md`)

> 근거: `_workspace/flarelane/02_validation.md` (검증일 2026-06-26) + FlareLane 공식 REST API 레퍼런스/가이드. 출처 없는 디테일은 "미확인"으로 표기한다.

## 연동 유형 / 인증 방식

- **연동 유형**: 공개 REST API 직접 호출(공식 MCP 커넥터·OAuth 로그인 흐름 없음). 우리 로컬 stdio MCP 서버가 `fetch`로 직접 호출한다.
- **인증 방식**: HTTP Bearer 토큰. 모든 엔드포인트 공통으로 `Authorization: Bearer ${FLARELANE_API_KEY}` 헤더 사용.
- **토큰 흐름**: 사전 발급된 **정적 프로젝트 단위 REST API 키**. OAuth/토큰 교환/리프레시 토큰 없음.
- **필수 식별자**: 모든 경로에 `PROJECT_ID`가 포함된다(`https://api.flarelane.com/v1/projects/{PROJECT_ID}/...`). API 키와 PROJECT_ID 두 값이 모두 필요하다.

## 발급 절차 (스텝)

> **직링크 안내**: API 키/PROJECT_ID 발급 화면은 **워크스페이스·프로젝트별로 URL이 달라 공통 직링크가 없다.** 콘솔 진입 직링크(`https://console.flarelane.com`)까지만 제공하고, 그 다음은 아래 메뉴 경로로 이동한다.

1. **콘솔 접속(직링크)**: `https://console.flarelane.com` 에 로그인한다(계정이 없으면 가입). 로그인하면 프로젝트 선택 화면으로 진입한다.
2. **프로젝트 선택/생성**: 대상 프로젝트를 연다. 이 프로젝트의 식별자가 `PROJECT_ID`다.
3. **PROJECT_ID 확인 (메뉴 경로 — 확인됨)**: 콘솔 로그인 → **프로젝트 선택** → 좌측/상단 **[Project](한국어 콘솔: [프로젝트])** 페이지. 공식 SDK 셋업 문서가 "You can find your project ID on the **[Project]** page in the console"라고 명시한다(= 프로젝트 페이지에서 PROJECT_ID 확인).
4. **REST API 키 확인/발급 (메뉴 경로 — 부분 확인)**: REST API 키는 **로그인한 콘솔에서 발급·조회**하며, 공식 REST API 레퍼런스의 인증 영역이 로그인 시 **"API Key" 표(Label / Last Used / Credentials 열)** 로 키를 노출한다("Log in to see your API keys"). 즉 **키는 워크스페이스 계정에 로그인한 상태에서 발급/확인하는 정적 키**다.
   - ⚠️ **콘솔 내 정확한 좌측 메뉴 > 버튼 경로(예: 설정 → API → [키 생성])는 공식 공개 문서에 명시되어 있지 않다(미확인).** 공식 "프로젝트 설정(Project Settings)" 문서 섹션이 다루는 항목은 *광고성 알림 가이드라인 · 멤버 초대 · 데이터 마스킹 · 2FA* 뿐이고, **API 키 발급 전용 안내 페이지는 존재하지 않는다**(EN/KO 문서 모두 확인). 따라서 키 발급은 "로그인한 콘솔 + REST API 레퍼런스의 인증 영역"에서 이루어지는 것으로 본다.
5. **환경변수로 주입**: 확인한 키와 PROJECT_ID를 `FLARELANE_API_KEY`, `FLARELANE_PROJECT_ID` 환경변수로 설정한다(`.env`는 커밋 금지).

> 두 필수 변수가 없으면 서버가 기동 시 "console.flarelane.com 프로젝트 설정에서 API 키와 PROJECT_ID를 발급해 환경변수로 설정하세요"라는 안내와 함께 종료한다. 잘못된 값은 런타임 **401**로 드러난다.

## 전제조건

키 발급 자체에 필요한 플랜/사업자 심사 등의 선행조건은 **공식 문서에 명시된 근거 없음(미확인)** — API 키는 셀프서비스로 보인다. 단, **채널별 발송 도구는 별도의 콘솔 선행설정이 필요**하다(키만으로는 200이 와도 실제 발송이 거부될 수 있음):

| 도구 | 즉시 동작(self-serve) | 발송 전 콘솔 선행설정 |
|------|----------------------|----------------------|
| `send_notification`, `track`, `view_notification_history`, `delete_users` | 가능 | 없음 |
| `send_sms`, `subscribe_sms`, `unsubscribe_sms` | 불가 | 발신번호 사전등록(전기통신사업법 제84조) · Billing 크레딧 충전 · **한국 내 발송만 지원** |
| `send_kakao_alimtalk` | 불가 | 카카오 비즈채널 전환(3~5영업일) · 콘솔 [Channels] 발신프로필 입력 · 템플릿 사전승인(2~3일) |
| `send_email` | 불가 | 콘솔 **템플릿** 생성(raw HTML 발송 불가) · 발신 이메일(senderEmail) 설정 |

## 스코프 / 권한

- 문서화된 스코프/권한 분리 체계 **없음(미확인)**. 읽기전용/발송전용 같은 키 구분 근거 없음.
- 구현은 **프로젝트 단위 단일 키가 전 작업 권한을 보유**한다는 전제로 동작한다.
- 키 만료/회전 정책도 **미문서화(미확인)** — 런타임 401을 "키/PROJECT_ID 무효 또는 만료"로 처리한다.

## 환경변수 매핑

| 변수명 | 값을 어디서 | 필수 |
|--------|------------|------|
| `FLARELANE_API_KEY` | console.flarelane.com 프로젝트 설정에서 발급한 REST API 키 (`Authorization: Bearer`에 사용) | 필수 |
| `FLARELANE_PROJECT_ID` | console.flarelane.com 프로젝트 식별자 (모든 엔드포인트 경로의 `{PROJECT_ID}`) | 필수 |
| `FLARELANE_BASE_URL` | 베이스 URL 오버라이드(기본 `https://api.flarelane.com/v1`) | 선택 |

## 출처 (공식 URL)

- 베이스 URL·인증 헤더·프로젝트 경로: https://flarelane-api-docs.readme.io/reference/send-notifications.md (외 10개 엔드포인트 페이지 동일)
- 인증 영역("Log in to see your API keys" + "API Key" 표 Label/Last Used/Credentials): https://flarelane-api-docs.readme.io/reference/track.md , https://flarelane-api-docs.readme.io/reference/authentication
- PROJECT_ID 위치("You can find your project ID on the [Project] page in the console"): https://flarelane.com/en/docs/sdk-integrations/android-sdk/ (공식 SDK 셋업, 2026-06-28 재확인)
- 프로젝트 설정 문서 섹션 항목(=광고성 알림 가이드라인/멤버 초대/데이터 마스킹/2FA — API 키 페이지 부재 근거): https://flarelane.com/ko/docs/ , https://flarelane.com/en/docs/ (사이드바, 2026-06-28 재확인)
- SMS 한국 내 발송·발신번호/크레딧 선행조건: https://flarelane.com/en/docs/channels/sms
- 카카오 알림톡 비즈채널/발신프로필/템플릿 승인 선행조건: https://flarelane.com/en/docs/channels/kakao-alimtalk
- 콘솔(진입 직링크): https://console.flarelane.com

## 미확인 / 주의

- **PROJECT_ID 위치 = 확인됨.** 콘솔 로그인 → 프로젝트 선택 → **[Project](프로젝트) 페이지**에서 확인(공식 SDK 셋업 문서 명시, 2026-06-28 재확인).
- **REST API 키 발급의 정확한 콘솔 좌측 메뉴 > 버튼 경로 = 미확인.** 키가 "로그인한 콘솔 + REST API 레퍼런스 인증 영역(API Key 표)"에서 발급/조회되는 점까지는 확인했으나, *설정 → API → [키 생성]* 같은 구체 클릭 경로는 공식 공개 문서에 없다. 공식 "프로젝트 설정" 문서 섹션은 광고성 알림 가이드라인·멤버 초대·데이터 마스킹·2FA만 다루며 **API 키 발급 전용 페이지가 없음**(EN/KO 사이드바 2026-06-28 재확인). 발급/조회 화면 직링크는 워크스페이스별이라 **공통 직링크 없음**.
- **키 스코프/권한 분리 = 미확인.** 단일 프로젝트 키 전제.
- **키 만료/회전 정책 = 미확인.** 401 응답으로 가드.
- **데이터 리전 옵션 = 미확인.** 단일 호스트 `api.flarelane.com`만 확인됨(단, SMS는 한국 내 발송만 지원 — 채널 단위 제약).
- 발송/구독/트래킹 = 100 req/sec, `delete_users` = 1 req/sec 레이트리밋. `delete_users`는 파괴적·복구 불가.
