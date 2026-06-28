# 키 발급 가이드 — ChatGPT (chatgpt)

OpenAI(ChatGPT) 공개 REST API(`https://api.openai.com/v1`)를 쓰기 위한 API 키 발급 가이드.
ChatGPT/OpenAI를 Claude 등 MCP 호스트에 붙이는 공식 원격 커넥터는 없으므로, OpenAI 대시보드에서 직접 API 키를 발급받아 로컬 MCP 서버에 주입한다(route = build).

## 연동 유형 / 인증 방식

- **연동 유형**: 운영자/사용자 API 키 직접 주입 (로컬 stdio MCP 서버). 엔드유저 OAuth 로그인 흐름 없음 — OpenAI API 본체는 OAuth 엔드유저 로그인을 제공하지 않으며, API 키를 환경변수로 주입받는 방식이 유일하다.
- **인증 방식**: API Key (Bearer). 모든 요청 헤더 `Authorization: Bearer ${OPENAI_API_KEY}`.
- **토큰 흐름**: 정적 시크릿. 토큰 교환·리프레시 없음.
- **선택 헤더**: 다중 조직/프로젝트 귀속 시 `OpenAI-Organization: ${OPENAI_ORG_ID}`, `OpenAI-Project: ${OPENAI_PROJECT_ID}`.

## 발급 절차 (스텝)

1. **로그인**: OpenAI 플랫폼 대시보드에 로그인한다. (https://platform.openai.com/login)
2. **결제 수단 등록**: 대시보드 Billing에서 결제 수단(카드)을 등록한다. 카드 등록만으로 키 발급·사용이 가능하며 별도 심사는 없다(self-serve).
3. **API 키 생성**: API Keys 페이지(https://platform.openai.com/api-keys)에서 "Create new secret key"로 키를 생성한다. 키 값(`sk-...`)은 생성 직후 1회만 표시되므로 즉시 복사해 안전하게 보관한다.
4. **(선택) 조직/프로젝트 확인**: 다중 조직/프로젝트를 쓴다면 대시보드에서 Organization ID / Project ID를 확인한다(선택 헤더용).
5. **환경변수 주입**: 발급한 키를 `OPENAI_API_KEY` 환경변수로 주입한다(예: `export OPENAI_API_KEY="sk-..."`). 공식 SDK는 이 env를 자동 인식한다.

## 전제조건

- **플랜/심사**: 없음. 카드 등록만으로 자가발급(self-serve), 별도 사업자 심사·실명·발신번호 사전등록 등 한국 특화 제약 없음(글로벌 self-serve).
- **레이트리밋 티어**: 누적 결제액에 따라 Free → Tier 1~5로 자동 승급(한도 상향). 초기에는 낮은 티어 한도가 적용된다.
- **GPT Image 모델 한정 전제조건**: `gpt-image-*` 모델을 사용하려면 개발자 콘솔에서 "API Organization Verification"(조직 인증)이 요구될 수 있다. 미인증 시 이미지 생성이 403으로 실패할 수 있다. 인증 부담이 없는 `dall-e-3`가 기본값이므로 일반 발급에는 추가 전제조건 없음.

## 스코프 / 권한

- 별도 OAuth 스코프 개념 없음. 발급한 키가 귀속된 프로젝트의 모델/엔드포인트 접근 권한을 그대로 사용한다.
- 본 MCP 서버의 12개 도구는 모두 **표준 프로젝트 API 키**로 호출 가능하다. Admin 전용 엔드포인트는 사용하지 않는다.
- 표준 키 vs Admin 키의 세부 권한 매트릭스는 공식 문서에서 확정되지 않았으나(미확인), 본 서버는 표준 키만 사용하므로 비차단.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `OPENAI_API_KEY` | platform.openai.com/api-keys 에서 "Create new secret key"로 발급한 `sk-...` 키 | 필수 |
| `OPENAI_ORG_ID` | 대시보드의 Organization ID (다중 조직 시 `OpenAI-Organization` 헤더) | 선택 |
| `OPENAI_PROJECT_ID` | 대시보드의 Project ID (프로젝트 귀속 시 `OpenAI-Project` 헤더) | 선택 |
| `OPENAI_DEFAULT_MODEL` | 직접 지정하는 기본 모델 ID(미설정 시 코드 기본값 `gpt-5.5`). 현행 ID는 `list_models` 도구로 조회 | 선택 |
| `OPENAI_BASE_URL` | 베이스 URL 오버라이드(프록시/Azure 등, 기본 `https://api.openai.com/v1`) | 선택 |

## 출처 (공식 URL)

- API 키 생성 페이지: https://platform.openai.com/api-keys (퀵스타트 "create an API key in the dashboard")
- 대시보드 로그인: https://platform.openai.com/login
- 퀵스타트(키 발급·env 자동 인식): https://developers.openai.com/api/docs/quickstart
- 인증 헤더(`Authorization: Bearer`, 선택 조직/프로젝트 헤더): https://developers.openai.com/api/reference/overview
- 레이트리밋 티어(Free + Tier 1~5, 누적 결제로 자동 승급): https://developers.openai.com/api/docs/guides/rate-limits
- GPT Image 조직 인증: https://developers.openai.com/api/docs/guides/image-generation

## 미확인 / 주의

- **결제/세금(한국 부가세)·데이터 사용/보존(DPA) 최신본**: 공식 결제·법무 페이지가 별도이며 02_validation에서 미확인으로 남음. 도구 동작과 무관하나 도입 전 컨설팅 후속 확인 권장.
- **키 스코프 세부 매트릭스(표준 키 vs Admin 키 경계)**: 대시보드 로그인 필요, 공식 문서로 확정 못 함(미확인). 본 서버는 표준 키만 사용하므로 비차단.
- **GPT Image 조직 인증 요구 여부**: 계정/지역에 따라 다를 수 있어 사전 확정 불가. 미인증 시 `generate_image`가 403으로 실패할 수 있다(에러를 그대로 노출 — 가짜 성공 없음).
- **키 1회 표시**: secret key는 생성 직후 1회만 노출된다. 분실 시 재발급해야 한다.
- **시크릿 취급**: 키를 코드/저장소에 하드코딩하지 말 것. `.env`로만 주입하고 `.gitignore`에 포함한다.
