# 키 발급 가이드 — BambooHR (bamboohr)

## 연동 유형 / 인증 방식
- **연동 유형:** REST API (HTTPS, JSON). 공식 원격 MCP 커넥터 없음 → 로컬 서버 빌드.
- **인증 방식:** **API Key + HTTP Basic Auth.** 발급한 API 키를 username으로, 임의 문자열 `x`를 password로 전송한다.
  - 헤더: `Authorization: Basic base64("{API_KEY}:x")` (코드에서 자동 처리)
- **토큰 흐름:** 없음 — 정적 API 키이며 만료/갱신 절차가 없다. OAuth 2.0/OIDC는 멀티테넌트용으로 존재하나 본 커넥터는 미채택.
- **베이스 URL:** `https://{BAMBOOHR_COMPANY_DOMAIN}.bamboohr.com/api/v1` (정식 형태 단일 고정. 레거시 게이트웨이 `api.bamboohr.com/.../gateway.php`는 사용 금지 — 공식 현행 문서 미수록.)

## 발급 절차 (스텝)
1. BambooHR에 로그인한다 (`https://{회사도메인}.bamboohr.com`). 충분한 권한(관리자급)이 있어야 API Keys 메뉴가 노출된다.
2. **API Keys 페이지로 이동**한다. 진입 경로는 UI 버전/권한에 따라 둘 중 하나다:
   - **경로 A (공식 문서 명시):** 화면 **좌하단의 본인 이름** 클릭 → 사용자 컨텍스트 메뉴에서 **API Keys** 선택.
   - **경로 B (Account 메뉴 / 신형 UI):** 홈에서 우상단 **Account**(계정 아이콘) 클릭 → **API Keys** 선택. (= Settings > Account > API Keys)
   - > **직링크(딥링크) 없음:** BambooHR은 API Keys 페이지로 바로 가는 공식 URL을 공개하지 않는다. 페이지 진입은 위 UI 메뉴 경로로만 가능(공식 문서·Okta/벤더 가이드 모두 메뉴 경로만 안내, 딥링크 미수록).
3. API Keys 페이지에서 **[Add New Key]** 버튼을 클릭한다.
4. **API Key Name** 필드에 식별용 이름을 입력하고 **[Generate Key]** 버튼을 클릭한다.
5. 표시되는 키 값을 **[Copy Key]**로 복사해 둔 뒤 **[Done]**을 누른다. 복사한 값이 `BAMBOOHR_API_KEY`다. (키는 생성 직후에만 전체 노출되므로 즉시 저장할 것.)
6. 회사 서브도메인(예: `acme.bamboohr.com`의 `acme`)을 `BAMBOOHR_COMPANY_DOMAIN`으로 사용한다.

> 주의: API 키의 **데이터 접근 범위 = 키를 생성한 사용자의 BambooHR 권한 레벨**과 동일하다. 따라서 과대권한 계정으로 키를 만들면 전사 인사데이터가 노출될 수 있으므로, **최소권한 전용 서비스 계정**으로 발급할 것.

## 전제조건
- **사업자 심사 / 앱 심사:** 불필요 (셀프서비스 발급).
- **발신번호 사전등록 등:** 해당 없음.
- **플랜/관리자 조건:** API Keys 메뉴는 충분한 권한이 있어야 노출된다. 플랜·관리자 설정에 따라 메뉴가 안 보일 수 있으며, 노출에 필요한 정확한 플랜/권한 조건은 공식 문서에 **명문 미확인**(아래 "미확인 / 주의" 참조).

## 스코프 / 권한
- 별도의 스코프 문자열이 없다(Basic Auth).
- 실질 접근 범위 = **키를 생성한 사용자의 권한 레벨** 전체. 권한이 키에 그대로 상속된다.
- 권장: 필요한 데이터만 볼 수 있는 **최소권한 전용 서비스 계정**으로 키를 발급.

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `BAMBOOHR_API_KEY` | 좌하단 이름 → API Keys에서 생성한 키. Basic Auth username으로 사용(password는 코드에서 `x` 고정) | 예 |
| `BAMBOOHR_COMPANY_DOMAIN` | 회사 서브도메인(예: `acme.bamboohr.com` → `acme`). 베이스 URL 조립에 사용 | 예 |

> 시크릿은 하드코딩 금지. `.env`(gitignore 처리)에 넣고 `.env.example`만 커밋한다.

## 출처 (공식 URL)
- 시작하기 / 키 발급 UI 경로(좌하단 이름 → API Keys) / 인증: https://documentation.bamboohr.com/docs/getting-started
- 키 발급 버튼명(Add New Key → Generate Key → Copy Key → Done) 및 Account 메뉴 경로 교차확인: https://support.okta.com/help/s/article/how-to-generate-a-bamboohr-api-key
- API 상세(HTTPS·UTF-8·에러 헤더·레이트리밋 503): https://documentation.bamboohr.com/docs/api-details
- 공식 SDK(현행 PHP만 유지보수): https://documentation.bamboohr.com/docs/sdks
- 커넥터 디렉터리(원격 MCP 미등재 확인): https://claude.com/docs/connectors/directory

## 미확인 / 주의
- **(미확인) API Keys 메뉴 노출 조건:** 셀프서비스 발급이 가능한 정확한 플랜/관리자 권한 조건은 공식 문서에 명문화되어 있지 않다. 메뉴가 보이지 않으면 관리자 권한/플랜을 확인할 것.
- **레이트리밋:** 초과 시 응답은 **HTTP 503**(429 아님)이며, `Retry-After` 헤더 제공이 보장되지 않는다. 공식 수치(req/min)는 비공개 → 지수 백오프로 대응(가정 금지).
- **에러 사유:** 응답 헤더 `X-BambooHR-Error-Message`에서 추출. 텍스트는 수시 변경될 수 있어 분기 로직 의존 금지.
- **응답 포맷:** 엔드포인트별 기본값이 다르므로(신형 JSON, 구형 XML) 모든 요청에 `Accept: application/json`을 명시한다.
- **구 `oidcLogin`("User API Key login flow"):** 2025.04.14부터 폐기 진행 — 신규 사용 금지.
