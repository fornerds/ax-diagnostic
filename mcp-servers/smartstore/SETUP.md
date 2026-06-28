# 키 발급 가이드 — 네이버 스마트스토어 / 커머스API (smartstore)

> 출처 기준: `_workspace/smartstore/02_validation.md`(검증일 2026-06-26, 커머스API 공식 기술문서 v2.81.0) + 발급 절차/콘솔 위치는 공식 docs가 fetch 차단되어 1회 웹 확인으로 보강(web_checked).

## 연동 유형 / 인증 방식
- **연동 유형**: 로컬 MCP 서버(stdio). 공식 Remote MCP/Claude 커넥터는 **없음**(재확인) → 직접 HTTP 호출.
  - 출처: https://apicenter.commerce.naver.com/docs/ai-use-guide
- **인증 방식**: OAuth 2.0 **Client Credentials Grant + 전자서명(`client_secret_sign`, bcrypt)**. **OAuth scope 스펙 없음.**
  - 출처: https://apicenter.commerce.naver.com/docs/auth
- **베이스 URL**: `https://api.commerce.naver.com/external` (경로 버전 v1/v2 혼재).
  - 출처: https://apicenter.commerce.naver.com/docs/restful-api
- **토큰 엔드포인트**: `POST /v1/oauth2/token` (`Content-Type: application/x-www-form-urlencoded`). 토큰 유효 **3시간(10,800초)**.
  - 출처: https://apicenter.commerce.naver.com/docs/commerce-api/current/exchange-sellers-auth , https://apicenter.commerce.naver.com/docs/commerce-api/current/o-auth-2-0

## 발급 절차 (스텝)
> **발급 위치 = 커머스API센터** `https://apicenter.commerce.naver.com`.
> **발급 페이지 직링크: 없음.** 공식 도메인(`apicenter.commerce.naver.com`)이 일반 fetch/크롤을 차단하고, 애플리케이션 발급 화면은 로그인 세션이 있어야 진입 가능한 어드민 영역이라 "클릭하면 바로 발급 화면으로 가는" 안정적 직링크를 공개 출처로 확정할 수 없다. → **아래 메뉴 경로 기준으로 진입**한다. 메뉴 경로/버튼명은 공개 가이드 다수로 교차 확인(web_checked, 2026-06 재확인). 콘솔 UI 라벨은 변동될 수 있으니 화면 흐름도 함께 본다.

**메뉴 경로(요약)**: 커머스API센터 로그인 → **애플리케이션 > 내 스토어 애플리케이션** → **[애플리케이션 등록]** → **[등록하기]** → (정보 입력 + API 호출 IP + API 그룹) → **[등록]** → 상세 화면에서 애플리케이션 ID/시크릿 확인.

1. **커머스API센터 접속·로그인**: `https://apicenter.commerce.naver.com` 접속 → 네이버 커머스/판매자 계정으로 로그인.
2. **가입하기**(최초 1회): 개발업체 계정명·장애대응 연락처 입력, 약관 동의 후 [가입하기] → [확인].
3. **애플리케이션 목록 진입**: 상단/좌측 **[애플리케이션] > [내 스토어 애플리케이션]** 메뉴로 이동.
4. **애플리케이션 등록**: **[애플리케이션 등록]** → **[등록하기]** 선택.
5. **상세 정보 입력**: 애플리케이션 정보 입력 → **API 호출 IP** 입력(서버 고정 공인 IP, **최대 3개**; 미입력 시 인증 불가) → **API 그룹** 필요 항목 추가(이 커넥터: 상품·주문·문의·정산) → **[등록]**.
6. **키 확인**: 등록 완료 후 애플리케이션 상세에서 **애플리케이션 ID(`client_id`)** 확인, **애플리케이션 시크릿(`client_secret`)** 은 **[보기]** 클릭으로 노출해 복사.
   - `client_secret`은 `$2a$10$...` 형식의 **완전한 bcrypt salt** 값이다(전자서명 시 bcrypt salt로 그대로 사용).
   - **인증/IP 추가**는 애플리케이션 상세 페이지 우측 상단 **[인증]** 버튼에서 진행한다.

## 전제조건
- **발급 권한자(통합매니저)**: 커머스API센터의 애플리케이션은 **통합매니저** 권한자만 발급/관리할 수 있다. 고객사가 통합매니저 권한을 보유했는지 **발급 전 확인 필요**. (web_checked)
  - 주의: 통합매니저가 변경되면 기존 통합매니저가 등록한 애플리케이션이 모두 삭제될 수 있다. (web_checked)
- **스토어당 애플리케이션 1개**: 스토어별 애플리케이션은 최대 1개 등록 가능. (web_checked / 공식 docs 본문 미명시 → 센터에서 재확인 권장)
- **호출 IP 사전등록(필수)**: MCP 서버가 도는 머신의 **고정 공인 IP**를 애플리케이션에 등록해야 한다. 미등록 IP는 게이트웨이가 `403 GW.IP_NOT_ALLOWED`로 차단(동적 IP·서버리스·재택 환경 함정). 등록은 애플리케이션 상세 우측 상단 **[인증]** 버튼 → IP 입력 → [추가] → [저장]. **IP 최대 3개**(공식 docs 본문 미명시이나 공개 가이드 다수 일치, web_checked) → 센터 화면에서 최종 확인.
  - 출처: https://apicenter.commerce.naver.com/docs/trouble-shooting , https://sidesaram.com/entry/네이버-커머스API센터-–-스토어-애플리케이션-인증-및-API호출-IP-추가-가이드
- **TLS 1.2 이상**: 필수(공식). Node 기본 환경 충족, 구형 런타임 주의.
  - 출처: 02_validation.md(누락·리스크 A.3)
- **플랜/사업자 심사**: 자사 스토어(`type=SELF`) 사용에 대한 별도 유료 플랜·사업자 추가 심사 요건은 검증 출처에 **명시 없음**.
- **발신번호 사전등록**: 해당 없음(이 커넥터는 SMS/알림톡 발송 기능을 포함하지 않음) → **없음**.
- **대행(`type=SELLER`) 사용 시**: 타 판매자 리소스 접근은 제휴/심사 전제가 있을 수 있음(미확정) → 1차는 `SELF` 권장.
  - 출처: https://apicenter.commerce.naver.com/docs/commerce-api/current/exchange-sellers-auth

## 스코프 / 권한
- **OAuth scope 없음**: 커머스API는 `scopes` 스펙을 제공하지 않는다. 코드에 scope 파라미터 없음.
  - 출처: https://apicenter.commerce.naver.com/docs/auth , https://apicenter.commerce.naver.com/docs/restriction
- **권한 부여 = API 그룹 선택**: 애플리케이션 등록 시 선택한 **API 그룹** 단위로 권한이 부여된다. 전체 9개 그룹(인증·상품·주문·정산·문의·물류(N배송)·커머스솔루션·판매자정보·API데이터솔루션) 중, 이 커넥터 대상은 **상품·주문·문의·정산**. 등록 시 해당 그룹을 추가해야 한다.
  - 출처: https://apicenter.commerce.naver.com/llms/llms.txt
- **인증 타입**: `type=SELF`(자사 스토어, 기본) 또는 `type=SELLER`(대행, `account_id` 필수).
  - 출처: https://apicenter.commerce.naver.com/docs/commerce-api/current/exchange-sellers-auth

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `NAVER_COMMERCE_CLIENT_ID` | 커머스API센터 → 애플리케이션 → **애플리케이션 ID** | 필수 |
| `NAVER_COMMERCE_CLIENT_SECRET` | 커머스API센터 → 애플리케이션 → **시크릿**(`$2a$10$...` bcrypt salt 형식) | 필수 |
| `NAVER_COMMERCE_AUTH_TYPE` | 사용 모드 직접 지정: `SELF`(기본) 또는 `SELLER`(대행) | 선택(기본 `SELF`) |
| `NAVER_COMMERCE_ACCOUNT_ID` | `SELLER`일 때 대상 판매자 ID/UID | 조건부(`SELLER` 시 필수) |
| `NAVER_COMMERCE_BASE_URL` | 베이스 URL 오버라이드(기본 `https://api.commerce.naver.com/external`) | 선택 |

> 시크릿은 코드/`.mcp.json`에 실값으로 쓰지 말고 환경변수로 주입한다(`.env`는 `.gitignore` 처리).

## 출처 (공식 URL)
- 커머스API센터(발급 콘솔): https://apicenter.commerce.naver.com — 로그인 후 **애플리케이션 > 내 스토어 애플리케이션**에서 발급(공개 직링크 없음, 메뉴 경로로 진입)
- 발급 메뉴 경로/버튼명·IP 등록(web_checked, 공개 가이드): https://sidesaram.com/entry/네이버-커머스API센터-–-스토어-애플리케이션-인증-및-API호출-IP-추가-가이드 , https://wikidocs.net/312069
- 인증/전자서명: https://apicenter.commerce.naver.com/docs/auth
- 토큰 교환(operation): https://apicenter.commerce.naver.com/docs/commerce-api/current/exchange-sellers-auth
- OAuth 2.0(토큰 유효시간): https://apicenter.commerce.naver.com/docs/commerce-api/current/o-auth-2-0
- 베이스 URL/RESTful: https://apicenter.commerce.naver.com/docs/restful-api
- 권한/scope/레이트리밋: https://apicenter.commerce.naver.com/docs/restriction
- 트러블슈팅(IP 차단): https://apicenter.commerce.naver.com/docs/trouble-shooting
- AI 활용 가이드(Remote MCP 없음 근거): https://apicenter.commerce.naver.com/docs/ai-use-guide
- endpoint 카탈로그(API 그룹/버전): https://apicenter.commerce.naver.com/llms/llms.txt

## 미확인 / 주의
- **발급 화면 직링크**: 공식 도메인(`apicenter.commerce.naver.com`)이 일반 fetch를 차단하고 발급 화면이 로그인 세션 기반 어드민이라, "클릭 즉시 발급 화면" 직링크는 공개 출처로 확정 불가 → **직링크 없음**으로 명시(메뉴 경로로 진입). 메뉴 경로/버튼명(애플리케이션 > 내 스토어 애플리케이션 → 등록)은 공개 가이드 다수로 교차 확인(web_checked). 화면 라벨/순서는 변동 가능 — 실제 콘솔 화면 기준으로 진행.
- **통합매니저 자격·IP 최대 개수·스토어당 앱 1개**: 공식 docs 본문에서 직접 확정하지 못함(커머스API센터 어드민/안내 영역, 웹 보강). 발급 전 고객사/센터에서 재확인 필요.
- **전자서명 Base64 인코딩**: 공식 코드예시가 자체 모순(Node/Python=표준 Base64, Java=URL-safe). 구현은 **표준 Base64**(Node 예시 기준). 401 반복 시 인코딩부터 의심.
  - 출처: https://apicenter.commerce.naver.com/docs/auth
- **timestamp 5분 유효**: 토큰 요청 직전 생성. 서버 시계 오차 크면 실패 → NTP 동기 권장.
- **레이트리밋 수치 비공개·동적**: 하드코딩 금지. `429 GW.RATE_LIMIT`/`GW.QUOTA_LIMIT` 시 응답 헤더(`GNCP-GW-RateLimit-*`) 기반 백오프.
  - 출처: https://apicenter.commerce.naver.com/docs/restriction
- **공식 SDK 미발견**: npm/pip 공식 SDK 확인 안 됨 → 직접 HTTP 호출.
  - 출처: https://github.com/commerce-api-naver/commerce-api
- **유료 플랜/사업자 추가 심사 요건**: `SELF` 사용에 대한 별도 요건은 검증 출처에 명시 없음(미확인).
