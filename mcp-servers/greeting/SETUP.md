# 키 발급 가이드 — 그리팅(Greeting) (greeting)

## 연동 유형 / 인증 방식
- **연동 유형:** REST OpenAPI 직접 호출 (공식 원격 MCP 커넥터 없음 → 로컬 stdio MCP 서버).
- **인증 방식:** 정적 API Key. OAuth 아님, 토큰 갱신 흐름 없음.
- **키 사용 위치:** 모든 API 요청 헤더에 `X-Greeting-OpenAPI: <발급된 키>` 포함. 사전 허가된 키로만 정상 동작.
- **버전 헤더:** `X-Api-Version` 는 엔드포인트마다 값이 다름(1.0/2.0/3.0). 단일 고정 금지(키 발급과 무관하나 호출 시 주의).
- 출처: https://guide.greetinghr.com/ko/articles/crtfd-1d1b706f

## 발급 절차 (스텝)
> **자동 발급 콘솔이 없다.** 그리팅은 셀프서비스 개발자 콘솔에서 키를 생성하는 방식이 아니라, 두들린(Doodlin) 팀과의 **수동 협의 후 발급**이다.

1) **사용 자격 확인** — 고객사 그리팅 요금제가 **비즈니스 플랜**인지 확인(아래 전제조건).
2) **발급 요청** — 두들린 지원팀 `support@doodlin.co.kr` 로 OpenAPI 키 발급을 요청한다.
3) **사용 목적·범위 협의** — 어떤 데이터/기능을 어떤 용도로 쓸지(사용 범위)를 두들린 팀과 협의한다. 사전 허가된 범위의 키로만 정상 수행된다.
4) **키 수령** — 협의 완료 후 두들린이 키를 수동 발급한다. 이 키를 환경변수 `GREETING_API_KEY` 에 주입.

> 키 확보까지 리드타임이 있으므로 개발 착수 전에 미리 요청해야 한다. 키 없이는 어떤 도구도 런타임 동작 불가(블로킹 전제).

## 전제조건
- **요금제:** 비즈니스 플랜 한정 (API·Webhook 모두). 출처: https://guide.greetinghr.com/ko/articles/webhook-77c3a3b5
- **수동 발급 협의:** 두들린 팀과 사용 목적·범위 협의 필수(셀프 발급 불가). 출처: https://guide.greetinghr.com/ko/articles/crtfd-1d1b706f
- **사업자 심사 / 발신번호 사전등록 등:** 공식 문서에 별도 명시 **없음**.

## 스코프 / 권한
- 공개 문서에 토큰 scope 체계(키 단위 권한 분리) **명시 없음 → 미확인**.
- 실제 권한은 발급 시 두들린과 협의하는 "사용 범위"로 결정된다. 구현은 단일 키 전권을 가정하고, 권한 부족 응답(401/403 또는 `success=false`)을 에러로 처리.
- 출처: https://guide.greetinghr.com/ko/articles/crtfd-1d1b706f

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `GREETING_API_KEY` | 두들린(`support@doodlin.co.kr`) 수동 발급 키. `X-Greeting-OpenAPI` 헤더에 사용 | 필수 |
| `GREETING_BASE_URL` | API 호스트 오버라이드(기본 `https://oapi.greetinghr.com`). `/openapi`·`/open` 접두는 코드가 부착 | 선택 |

## 출처 (공식 URL)
- 인증·API Key·발급(두들린 협의·support@doodlin.co.kr): https://guide.greetinghr.com/ko/articles/crtfd-1d1b706f
- 비즈니스 플랜 한정: https://guide.greetinghr.com/ko/articles/webhook-77c3a3b5
- API 문서 인덱스: https://guide.greetinghr.com/ko/categories/-API-문서-da70b6c4
- 베이스 URL `https://oapi.greetinghr.com/openapi/` (평가 2종만 `/open/`): https://guide.greetinghr.com/ko/articles/crtfd-1d1b706f

## 미확인 / 주의
- **키 scope 체계** — 키 단위 권한 분리 여부 미문서화. 발급 시 두들린에 문의 권장.
- **IP allowlist** — 키 발급 시 IP 허용목록 필요 여부 공식 문서 명시 없음. 두들린에 확인 권장.
- **레이트리밋 수치** — 공식 문서에 수치 없음. 런타임 429 핸들링으로 대비.
- **Webhook 서명·검증(HMAC)·구독 URL 등록 위치** — 합격자 Webhook 수신 메커니즘 미문서화. 본 MCP는 Webhook 수신을 제외하고 `list_passed_applicants` polling으로 대체. 실시간 수신이 필요하면 두들린 협의 필요. 출처: https://guide.greetinghr.com/ko/articles/pass-75052827
- **발급 리드타임** — 수동 협의 발급이라 즉시 발급 아님. 개발 착수 전 선요청 필수.
- 키 발급 시 함께 문의 권장 항목: (1) 레이트리밋 (2) IP allowlist (3) 키 scope (4) Webhook 서명 방식.
