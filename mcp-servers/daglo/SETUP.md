# 키 발급 가이드 — 다글로 (daglo)

> **주의: 아래 발급 경로(메뉴/버튼)는 미확인 — 정확하지 않을 수 있습니다.** 발급 콘솔이 로그인 게이트/계정별이라 외부에서 화면을 확인하지 못했습니다. 실제 UI와 다를 수 있으니 공식 콘솔에서 직접 확인하세요. (추적: `mcp-servers/_SETUP_GAPS.md`)

> 출처: `_workspace/daglo/02_validation.md`(검증일 2026-06-26, doc-validator) 및 `mcp-servers/daglo/README.md`. 추측 없이 출처 있는 사실만 정리한다. 발급 디테일이 공식 문서로 확인 안 되는 항목은 "미확인"으로 남긴다.

## 연동 유형 / 인증 방식

- **연동 유형:** 공개 REST API 직접 호출(로컬 stdio MCP 서버). Anthropic 공식 원격 커넥터 디렉터리에는 Daglo 항목 없음.
- **인증 방식:** API Key — HTTP 헤더 `Authorization: Bearer <DAGLO_API_TOKEN>`. 모든 REST 엔드포인트(STT/NLP/TTS) 공통. OAuth2 아님, 사전 토큰 교환 없음.
- **토큰 흐름:** 콘솔에서 토큰 발급 → 환경변수로 주입 → 매 요청 헤더에 그대로 사용.
- 출처: https://developers.daglo.ai/guide/STT-Sync.html , https://developers.daglo.ai/guide/Language-chat-completion.html

## 발급 절차 (스텝)

1. 다글로 개발자 콘솔에 접속한다: https://developers.daglo.ai/console
   - **직링크:** 토큰 발급 화면으로 바로 가는 워크스페이스/딥링크 URL은 공식 문서에 없음. 콘솔 진입점은 위 `/console` 루트 하나뿐(공식 가이드 본문에 명시된 유일한 URL). `/console/token` 류 직링크는 문서·검색에서 확인되지 않음.
2. 계정 가입 후 로그인한다(심사 없음). 공식 가이드 문구: "API Console에 접속하여 회원가입 후 로그인합니다."
3. 콘솔의 **토큰 메뉴**에 들어가 API 토큰을 발급한다. 공식 가이드 문구(원문 인용): "토큰 메뉴에 들어가 새로운 토큰을 발급합니다."
   - 즉 경로는 **콘솔 접속 → 로그인 → 토큰 메뉴 → 새 토큰 발급**.
4. 발급된 토큰 값을 복사해 환경변수 `DAGLO_API_TOKEN`에 넣는다(코드/설정 파일에 실값 하드코딩 금지).

> 신규 계정에는 테스트 크레딧이 제공된다(비동기 STT 약 10시간 분량). 출처: https://developers.daglo.ai/guide/en/FAQ.html
> **미확인(버튼 라벨):** "토큰 메뉴" 진입까지는 공식 가이드로 확정되나, 토큰 메뉴 안의 정확한 버튼 라벨(예: "토큰 발급"/"새 토큰" 등 표기)은 콘솔·`/reference`·`/story` 페이지가 모두 SPA 동적 렌더링이라 문서·검색에서 추출 불가 → 콘솔 로그인 후 직접 확인. (검증일 2026-06-28, WebFetch/WebSearch 재확인)

## 전제조건

- **플랜/사업자 심사:** 없음(가입 시 심사 없이 토큰 발급, 신규 테스트 크레딧 제공). 출처: https://developers.daglo.ai/guide/en/FAQ.html
- **발신번호 사전등록 등:** 해당 없음(음성/언어 API라 통신 발신번호 개념 없음).
- **결제 등록:** 초기 사용에는 불필요하나, 무료 크레딧 소진 후에는 결제 등록이 필요하다(미등록 시 요청이 막힐 수 있음, 403 류). 출처: `02_validation.md` 누락·리스크 5, README 중요 고지 5.
- **실시간 gRPC STT(StreamingRecognize):** 별도 권한 신청 필요 — api-support@daglo.ai 문의. 본 MCP 빌드 범위 밖. 출처: https://developers.daglo.ai/guide/STT-Realtime.html , https://developers.daglo.ai/guide/en/FAQ.html

## 스코프 / 권한

- 세분 스코프 없음. 토큰 1개로 REST 전체(STT 동기/비동기, NLP 회의록·챗, TTS) 호출 가능.
- 실시간 gRPC STT만 별도 권한이 필요하며 본 빌드에서 제외.
- 출처: `02_validation.md` 인증(확정), 누락·리스크 6.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `DAGLO_API_TOKEN` | 다글로 콘솔(https://developers.daglo.ai/console)에서 발급한 API 토큰 | 필수 |
| `DAGLO_API_BASE_URL` | 베이스 URL 오버라이드. 미설정 시 기본 `https://apis.daglo.ai`. 스테이징/프록시용 | 선택 |
| `DAGLO_TTS_OUTPUT_DIR` | TTS WAV 저장 폴더. 미설정 시 OS 임시 폴더 | 선택 |

> 시크릿 하드코딩 금지. `.env.example`에는 `DAGLO_API_TOKEN=` placeholder만 두고 실제 값은 사용자가 주입.

## 출처 (공식 URL)

- 개발자 콘솔(토큰 발급 진입점, 직링크 없음): https://developers.daglo.ai/console
- 토큰 발급 절차 원문("회원가입 후 로그인 → 토큰 메뉴에 들어가 새로운 토큰을 발급"): https://developers.daglo.ai/guide/Quick-Speech-Voice-To-Text.html
- 인증 헤더(Bearer): https://developers.daglo.ai/guide/STT-Sync.html
- 가입·심사 없음·신규 크레딧: https://developers.daglo.ai/guide/en/FAQ.html
- 베이스 URL `https://apis.daglo.ai`: https://developers.daglo.ai/guide/STT-Async.html
- 실시간 gRPC 권한 문의: https://developers.daglo.ai/guide/STT-Realtime.html
- 개발자 문서 허브: https://developers.daglo.ai/guide/

## 미확인 / 주의

- **콘솔 내 토큰 발급 화면 경로/라벨:** 정확한 메뉴 위치·버튼명 미확인(콘솔/가이드 SPA 동적 렌더링으로 본문 추출 불가). 콘솔 로그인 후 직접 확인.
- **토큰 스코프 세분·만료·재발급 정책:** FAQ 미기재 → 콘솔/지원 문의 필요. 출처: https://developers.daglo.ai/guide/en/FAQ.html
- **데이터 보관(retention) 정책:** 공식 본문 미기재 → 미확인.
- **레이트리밋 구체 수치 / 동시작업 한도:** `429` 코드만 정의되고 분당·동시 수치 본문 없음. 코드로 한도를 가정하지 말 것(429/503 시 지수 백오프 대응).
- **가격(STT 분당/TTS 글자당/챗 토큰당)·정확 크레딧 수치:** pricing 페이지 동적 렌더링으로 미확인("약 10시간" 외 단가 불명).
- **문서 버전 노후 주의:** 모든 가이드가 "20240902 ver1.0" 표기(약 1년 9개월 전). v2/폐기/마이그레이션 공지는 없어 현행 유효로 판단하나, 첫 호출에서 401/404가 나면 토큰·경로를 우선 점검.
