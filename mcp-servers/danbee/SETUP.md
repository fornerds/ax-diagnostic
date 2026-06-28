# 키 발급 가이드 — 단비ai (danbee)

> 검증 출처: `_workspace/danbee/02_validation.md` (검증일 2026-06-26) + 공식 문서 https://doc.danbee.ai/channel_native_app.html (Updated: Oct 24, 2024) 1회 보강 확인.

## 연동 유형 / 인증 방식

- **연동 유형:** REST (JSON over HTTPS POST). 베이스 URL `https://danbee.ai`, 경로 프리픽스 `/chatflow`.
- **인증 방식: 인증 없음.** Authorization / API key / 서명 헤더가 **불필요**합니다 (공식 문서 + 공식 예제 2종 교차확인).
- 챗봇 식별은 오직 **`chatbot_id`(챗봇 UUID)** 하나로 이뤄집니다. 경로의 `{chatbotId}` 가 단일 진실원입니다.
- **주의:** 인증이 없으므로 `chatbot_id` 가 사실상 접근키 역할을 하지만 비밀성이 약합니다. 강한 시크릿으로 취급하지 말고 환경변수로만 주입하세요. (별도의 API 토큰 발급 콘솔은 존재하지 않습니다.)

## 발급 절차 (스텝)

별도의 "키 발급" 콘솔이 없습니다. `chatbot_id` 는 단비 플랫폼(저작도구)에서 챗봇을 만든 뒤 확인하는 값입니다.

1. **단비 플랫폼(저작도구) 로그인** — 직링크: **`https://danbee.ai/platform/#/loginPrivateStep`** (저작도구 로그인 화면). 계정이 없으면 먼저 가입(랜딩 `https://danbee.ai/` 의 `[2주 무료 만들기]` 또는 `https://danbee.ai/landing.html`).
2. **챗봇 생성** — 최초 로그인 시 챗봇이 하나도 없는 상태. 화면 하단 **`[처음부터 만들기]`** 영역의 **`[Chatbot 생성]`** 버튼을 클릭해 챗봇 이름·설명을 입력하고 생성한다. (샘플을 쓰려면 상단 샘플 영역의 `[샘플 더보기]`.)
3. **챗봇 배포 → 메신저 연결** — 콘솔 화면 경로 `[챗봇 만들기] > [챗봇 배포] > [메신저 연결]` 로 이동.
4. **chatbot_id 확인** — 위 배포/메신저 연결 화면에서 챗봇 UUID(`chatbotId` / 바디 `chatbot_id`)를 확인하고 복사한다. 이 값을 `DANBEE_CHATBOT_ID` 환경변수에 넣는다. *(공식 문서가 `chatbotId` 를 API 경로 파라미터로만 명세하고 콘솔 내 정확한 표시 필드명은 별도로 적지 않으므로, 배포 화면에서 챗봇 UUID 표기를 직접 확인. → 아래 "미확인" 참조.)*

> 참고: `evaluate_message` 의 `log_id` 는 발급값이 아니라 직전 `send_message` 응답의 `responseSet.result.log_id` 런타임 값입니다.

## 전제조건

- **사업자 심사 / 제휴 승인:** 문서상 요건 **없음** (Trial 셀프서비스로 가입 즉시 사용). *단, 실제 가입 화면의 본인인증 단계 유무는 미검증 — 아래 "미확인" 참조.*
- **발신번호 사전등록:** 해당 없음 (대화 API, 알림톡/문자 발송 아님).
- **플랜:** Trial(4주 / 1000cc) 제공. Trial 종료 시 Starter(₩3,000/월~)로 **자동 전환**. **유료 SaaS(cc 과금)** 이며, 응답 말풍선 1회 또는 API 요청 1회당 1cc 가 과금됩니다. cc 소진 시 호출이 실패할 수 있습니다.
- **네트워크(방화벽):** MCP 서버가 사내망 뒤에 있다면 `danbee.ai` 로의 **아웃바운드 HTTPS** 를 허용해야 합니다. (단비 서버 참고 IP: `13.248.170.89`, `76.223.41.54` — 인바운드가 아니라 MCP 서버가 단비로 나가는 호출입니다.)

## 스코프 / 권한

- **스코프/권한 모델 없음.** 인증·토큰이 없으므로 권한 범위 개념이 없습니다. `chatbot_id` 를 아는 주체가 해당 챗봇 대화 API 를 호출할 수 있습니다.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `DANBEE_CHATBOT_ID` | 단비 저작도구(`https://danbee.ai/platform/#/loginPrivateStep` 로그인) → `[챗봇 만들기] > [챗봇 배포] > [메신저 연결]` 화면의 챗봇 UUID | 필수 |
| `DANBEE_BASE_URL` | 베이스 URL 오버라이드 (기본 `https://danbee.ai`) | 선택 |
| `DANBEE_VERSION` | API 버전 path 값 (기본 `v2.0`, 하위호환 `v1.0`) | 선택 |

> `trigger_event` 의 `event_id` 는 환경변수가 아니라 도구 호출 입력값입니다. 자동 조회 수단이 문서화돼 있지 않으므로 **단비 콘솔에서 직접 확인**해 입력해야 합니다.

## 출처 (공식 URL)

- 대화 API / 인증 / 엔드포인트 / chatbot_id 위치: https://doc.danbee.ai/channel_native_app.html
- 요금제(Trial 4주·1000cc → Starter 자동전환, cc 과금 정의): https://doc.danbee.ai/pricing.html
- 챗봇 생성 콘솔 절차(저작도구 로그인 URL `platform/#/loginPrivateStep`, `[처음부터 만들기] > [Chatbot 생성]` 버튼명): https://doc.danbee.ai/basic_create_chatbot.html
- 시작/가입 랜딩: https://danbee.ai/ , https://danbee.ai/landing.html
- 검증 노트(상호 교차대조 결과): `_workspace/danbee/02_validation.md`

## 미확인 / 주의

- **가입 시 본인인증 단계 유무 — 미확인.** 문서상 사업자/제휴 심사 요건은 없으나(Trial 셀프서비스), 실제 가입 화면의 본인인증 단계 존재 여부는 검증되지 않았습니다.
- **콘솔 내 chatbot_id 정확 표시 필드 — 부분 미확인.** 저작도구 로그인 직링크(`https://danbee.ai/platform/#/loginPrivateStep`)와 메뉴 경로(`[챗봇 만들기] > [챗봇 배포] > [메신저 연결]`)·생성 버튼명(`[Chatbot 생성]`)은 공식 문서로 확인됨. 다만 공식 문서는 `chatbotId` 를 API 경로 파라미터로만 명세하고 배포 화면 내 정확한 표시 라벨/필드명은 적지 않으므로, 챗봇 UUID 의 실제 표기 위치는 콘솔에서 직접 확인이 필요합니다.
- **문서 최신성:** 공식 문서 Updated Oct 24, 2024 (현시점 기준 약 20개월 경과). 폐기/마이그레이션 공지는 없어 현행 유효로 간주하나, 사용 전 콘솔에서 chatbot_id 위치를 재확인 권장.
- **인증 없음 = `chatbot_id` 가 접근키.** 비밀성이 약하므로 코드/`.mcp.json` 에 실값 하드코딩 금지, 환경변수로만 주입.
