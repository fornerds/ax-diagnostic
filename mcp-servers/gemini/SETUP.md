# 키 발급 가이드 — 제미나이 (gemini)

> 대상: Google **Gemini API (Developer)** — 개발자용 Gemini API.
> 소비자 Gemini 앱/웹은 공개 API가 없어 대상이 아니다.
> 검증 출처: `_workspace/gemini/02_validation.md` (검증일 2026-06-26, 기준 ai.google.dev 공식 문서).

## 연동 유형 / 인증 방식

- **연동 유형:** API Key (단일 키). OAuth 토큰 교환 없음.
- **인증 방식:** HTTP 헤더 `x-goog-api-key: <API_KEY>` 로 전달.
  - 구방식 `?key=<KEY>` 쿼리 파라미터는 현행 공식 문서에서 안내하지 않으므로 의존 금지. 헤더 단일 방식으로 사용한다.
- **토큰 흐름:** 사전 토큰 교환 없음. AI Studio에서 발급한 키를 그대로 헤더에 실어 호출한다. 키는 GCP 프로젝트(과금/쿼터)에 바인딩된다.
- 출처: https://ai.google.dev/gemini-api/docs/api-key

## 발급 절차 (스텝)

1. Google 계정으로 **Google AI Studio**의 API 키 페이지에 접속한다: https://aistudio.google.com/apikey
2. "Create API key"(API 키 만들기)를 눌러 키를 생성한다. 신규 키는 즉시 발급되며, 심사/승인 대기가 없다.
3. 생성된 키 문자열을 복사해 안전하게 보관한다(키는 GCP 프로젝트에 바인딩되며, 과금·쿼터가 이 프로젝트에 귀속).
4. 발급한 키를 환경변수 `GEMINI_API_KEY` 로 설정한다(아래 "환경변수 매핑" 참조).

> (유료 전환) 무료 티어에서 유료 티어로의 승급은 별도 신청이 아니라 **GCP 프로젝트에 결제(빌링) 연결**로 자동 처리된다. Tier 1=빌링 연결 / Tier 2=결제 $100 누적+첫 결제 후 3일 / Tier 3=$1,000 누적+30일. (출처: https://ai.google.dev/gemini-api/docs/rate-limits)

## 전제조건

- **사업자/제휴 심사:** 없음. Google 계정 + AI Studio 키만 있으면 한국에서 즉시 사용 가능(검증됨).
- **발신번호 사전등록 등 도메인 특화 전제:** 없음(해당 없음).
- **지역 지원:** 한국(South Korea)은 정식 지원 지역에 포함된다. (출처: https://ai.google.dev/gemini-api/docs/available-regions)
- **유료 티어 전환:** 별도 심사 없이 GCP 프로젝트 빌링 연결로 자동 승급.

## 스코프 / 권한

- **런타임 필수 스코프:** 없음 — API Key 자체가 인증 수단이다(OAuth 스코프 개념 없음).
- **키 생성 측 권한(발급자 IAM):** 키를 만드는 주체는 `apikeys.keys.create` 권한이 필요하다(발급 시점 한정, 런타임 호출과는 무관).
- 출처: https://ai.google.dev/gemini-api/docs/api-key

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `GEMINI_API_KEY` | AI Studio(https://aistudio.google.com/apikey)에서 발급한 API 키. `x-goog-api-key` 헤더로 전송됨 | 필수 |
| `GOOGLE_API_KEY` | 위 미설정 시 fallback으로 사용(공식 라이브러리 호환). 동일한 AI Studio 발급 키 | 선택 |
| `GEMINI_DEFAULT_MODEL` | generate/countTokens 기본 모델 ID 오버라이드. 미설정 시 `gemini-2.5-flash`. 가용 모델은 `list_models`로 동적 확인 | 선택 |
| `GEMINI_EMBED_MODEL` | 임베딩 기본 모델 ID 오버라이드. 미설정 시 `gemini-embedding-001` | 선택 |

> 둘 다 설정 시 공식 라이브러리 우선순위는 `GOOGLE_API_KEY`이나, 본 MCP 서버는 `GEMINI_API_KEY`를 1차로 읽고 둘 다 지원한다.
> 키는 코드에 하드코딩하지 말고 환경변수로만 주입한다. `.env`는 커밋하지 않는다.

## 출처 (공식 URL)

- 키 발급/인증: https://ai.google.dev/gemini-api/docs/api-key
- 키 발급 콘솔(AI Studio): https://aistudio.google.com/apikey
- 지원 지역(한국 포함): https://ai.google.dev/gemini-api/docs/available-regions
- 레이트리밋/티어: https://ai.google.dev/gemini-api/docs/rate-limits
- 약관(데이터 사용 정책): https://ai.google.dev/gemini-api/terms

## 미확인 / 주의

- **모델 ID 변동성:** Gemini 모델명은 변동이 잦다(2.0→2.5→3.x). 기본값이 깨지면 `list_models`로 현재 가용 모델을 확인하고 `GEMINI_DEFAULT_MODEL`/`GEMINI_EMBED_MODEL`로 주입하라.
- **레이트리밋은 프로젝트 단위:** 쿼터는 API 키가 아니라 **GCP 프로젝트 단위**로 공유된다. 같은 프로젝트의 여러 키가 쿼터를 나눠 쓴다. 무료 티어는 RPD가 낮아 일 단위 소진에 유의.
- **모델별 정확 RPM 수치(미확인):** 공식이 수치를 정적으로 공개하지 않고 AI Studio에서 동적으로만 제공한다. 정적 확정 불가 → 런타임 429(RESOURCE_EXHAUSTED) 가드로 대응.
- **무료 티어 데이터 사용(미확인 약관 원문):** 무료 티어 입력 데이터는 모델 개선에 사용될 수 있다(유료 티어와 정책 상이). 한국 기업의 개인정보·기밀 처리 시 **유료 티어 사용 + 약관 확인**(https://ai.google.dev/gemini-api/terms) 권장.
