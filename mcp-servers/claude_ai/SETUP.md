# 키 발급 가이드 — Claude (claude_ai)

> 검증 출처: `_workspace/claude_ai/02_validation.md`(2026-06-26, 적대적 검증) + `mcp-servers/claude_ai/README.md`.
> 추측 금지 — 아래는 공식 문서/검증본에 출처가 있는 사실만 기재한다.

## 연동 유형 / 인증 방식

- **연동 유형:** route=build. "Claude에 로그인 연결"하는 공식 원격 MCP 커넥터는 **존재하지 않는다**(Claude 자체가 MCP 호스트). 이 커넥터는 Anthropic **Messages REST API**를 감싸는 로컬 stdio MCP 서버이며, Claude / Claude Code가 호스트로서 이를 소비한다.
- **인증 방식:** **API Key**(정적 시크릿). OAuth/토큰 교환 흐름 없음.
- 모든 요청에 헤더 3종 필수(공식 TS SDK `@anthropic-ai/sdk`가 자동 부착):
  - `x-api-key: <ANTHROPIC_API_KEY>`
  - `anthropic-version: 2023-06-01`
  - `content-type: application/json`
- 베이스 URL: `https://api.anthropic.com` (엔드포인트 prefix `/v1/...`).

## 발급 절차 (스텝)

1. **Anthropic Console 접속** — `https://console.anthropic.com` 에 가입/로그인.
2. **결제수단/크레딧 준비** — 키 사용을 위해 결제수단 등록 또는 크레딧 구매(USD 청구). 사업자/제휴 심사는 **불필요**.
3. **API Key 생성** — Console의 **Settings > Keys**(API Keys)에서 키를 즉시 자가 발급. (출처: API overview "Getting API keys")
4. **키 보관** — 생성 직후 표시되는 키 값을 안전하게 복사·보관(시크릿). 환경변수 `ANTHROPIC_API_KEY`로만 주입하고 코드에 하드코딩하지 않는다.

## 전제조건

- **사업자 심사:** 없음.
- **제휴/파트너 심사:** 없음.
- **발신번호 사전등록 등 한국 특화 게이트:** 없음.
- **플랜/결제:** 결제수단 또는 크레딧 필요(USD 청구). Tier 1은 $5 크레딧 구매로 진입, 초기 월 $500 spend limit. (구현 자체는 유효한 키만 있으면 동작.)
- 요약: console.anthropic.com 가입 + 결제수단/크레딧만으로 즉시 발급 — 발급 장벽 매우 낮음.

## 스코프 / 권한

- 키 단위 세부 스코프/권한 enum은 **공개 API 문서에 미공개**(Console UI 기반). 구현은 키를 **불투명 시크릿**으로 취급하므로 무관.
- 워크스페이스 분리/권한 운영은 운영자(콘솔 관리자) 책임.
- 베타 기능(Files API 등) 사용 시 호출에 `anthropic-beta: files-api-2025-04-14` 추가 필요(현재 1차 구현 범위에서는 제외).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `ANTHROPIC_API_KEY` | Anthropic Console > Settings > Keys 에서 발급한 API 키. SDK가 자동으로 `x-api-key`에 부착. **하드코딩 금지.** | 필수 |
| `ANTHROPIC_BASE_URL` | 베이스 URL 오버라이드(프록시/엔터프라이즈). 미설정 시 `https://api.anthropic.com`. | 선택 |
| `ANTHROPIC_DEFAULT_MODEL` | 도구 호출 시 `model` 미지정의 기본값. 미설정 시 `claude-opus-4-8`. | 선택 |

## 출처 (공식 URL)

- API overview / 인증 헤더 / 키 발급(Settings > Keys): https://platform.claude.com/docs/en/api/overview
- API 버전(`anthropic-version: 2023-06-01`): https://platform.claude.com/docs/en/api/versioning
- 모델 개요(권장 모델 ID): https://platform.claude.com/docs/en/about-claude/models/overview
- 레이트리밋/tier: https://platform.claude.com/docs/en/api/rate-limits
- Console(키 발급 위치): https://console.anthropic.com

## 미확인 / 주의

- **키 권한/스코프 enum:** 미확인(공개 문서에 미공개). 키는 불투명 시크릿으로 취급.
- **tier별 정확한 RPM/TPM 수치:** 미확인(계정별·동적). 하드코딩 금지 — 런타임에서 429 + `retry-after`로 흡수.
- **주의(401 흔한 원인):** `ANTHROPIC_API_KEY`와 `ANTHROPIC_AUTH_TOKEN`을 **동시에** 설정하면 SDK가 401을 낼 수 있다. 키는 **하나만** 설정(이 서버는 API_KEY 경로 사용).
- **주의(모델 ID):** 권장 모델 ID(`claude-opus-4-8` 등)에 날짜접미사를 임의로 붙이지 말 것 — 잘못된 ID는 404.
