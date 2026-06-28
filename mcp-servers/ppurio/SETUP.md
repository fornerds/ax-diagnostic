# 키 발급 가이드 — 비즈뿌리오 / 뿌리오 (ppurio)

> **주의: 아래 발급 경로(메뉴/버튼)는 미확인 — 정확하지 않을 수 있습니다.** 발급 콘솔이 로그인 게이트/계정별이라 외부에서 화면을 확인하지 못했습니다. 실제 UI와 다를 수 있으니 공식 콘솔에서 직접 확인하세요. (추적: `mcp-servers/_SETUP_GAPS.md`)

> 본 MCP 서버의 **1차 구현 대상은 비즈뿌리오 API(`api.bizppurio.com`)** 입니다.
> 다우기술은 두 제품(비즈뿌리오, 뿌리오 send-api)을 운영하며 인증 문자열·호스트·바디 스키마가 서로 다릅니다.
> 아래는 비즈뿌리오 기준이며, send-api를 계약한 경우는 "대안 경로" 섹션을 참고하세요.
> 출처는 모두 공식/1차 문서이며, 검증본은 `_workspace/ppurio/02_validation.md` 입니다.

---

## 연동 유형 / 인증 방식

- **연동 유형**: 자체 빌드 로컬 MCP 서버(공식 원격 MCP 커넥터 없음). REST API 직접 호출(stdio 전송).
- **인증 방식**: 2단계 토큰 인증.
  1. `POST /v1/token` 호출 — 헤더 `Authorization: Basic {Base64(계정ID:암호)}` → 응답으로 `accesstoken`(JWT), `type`("Bearer"), `expired`(yyyyMMddHHmmss) 수신.
  2. 이후 모든 호출 — 헤더 `Authorization: Bearer {accesstoken}`.
- **토큰 유효기간**: 24시간. 만료 전 자동 재발급(메모리 캐시). 매 호출 재발급 금지(레이트리밋·과금 소모).
- 출처: https://biztech.gitbook.io/webapi/llms-full.txt , https://github.com/bizppurio-api/api-documentation

---

## 발급 절차 (스텝)

> 비즈뿌리오는 **콘솔에서 즉시 발급되는 API 키 방식이 아니라**, 기업회원 가입 후 연동 신청·심사를 거쳐 계정 ID/암호를 발급받는 방식입니다.

1. **기업회원 가입** — 뿌리오 가입(개인회원 불가, 기업회원 전용). 가입/인증: https://www.ppurio.com/user/join/auth
2. **발신번호 사전등록** — 전기통신사업법 §84의2 법정 의무. 사용할 발신번호를 사전 등록(미등록 번호 발송 불가).
3. **연동(API) 신청·심사** — 영업일 1~2일 소요. 심사 통과 시 **연동용 계정 ID/암호** 발급.
4. **(카카오 사용 시)** 카카오 채널 개설 → 발신프로필키(senderkey) 발급 → 알림톡/친구톡 템플릿 등록 → 카카오 심사.
5. **환경변수 설정** — 발급받은 계정 ID/암호를 `BIZPPURIO_ACCOUNT` / `BIZPPURIO_PASSWORD` 에 설정(아래 매핑 표).
6. **토큰 발급 검증** — 서버가 `POST /v1/token`(Basic 인증)으로 액세스 토큰을 정상 발급받는지 확인(서버가 자동 수행).

- 출처: https://www.ppurio.com/send-api/guide (기업회원 전용·연동 심사 영업일 1~2일), https://www.ppurio.com/user/join/auth (발신번호 사전등록)

---

## 전제조건

- **기업회원 가입 필수**(개인회원 불가).
- **연동 신청·심사**(영업일 1~2일) 통과 후 계정 발급.
- **발신번호 사전등록**(법정 의무) — 미등록 번호 발송 불가.
- **카카오(알림톡/친구톡) 사용 시**: 채널 개설 + 발신프로필키 + 템플릿 등록 + 카카오 심사.
- **발송은 유료·비가역** — 건당 과금, 취소 불가(접수 후). 운영상 파괴적 작업으로 취급.
- **(send-api 대안 경로 한정)**: 발송 서버 **고정 IP 사전등록(allowlist)** 후 인증키 발급 — 동적 IP/로컬 PC 운영 곤란, 고정 IP 호스트 배치 필요.

- 출처: https://www.ppurio.com/send-api/guide , https://biztech.gitbook.io/webapi/status-code/at-ai-ft , 전기통신사업법 §84의2 (https://law.go.kr/LSW/lsLinkProc.do?lsNm=전기통신사업법&chrClsCd=010202&mode=20)

---

## 스코프 / 권한

- 별도 OAuth scope 없음. **계정 단위 권한**(발급된 계정 ID/암호가 곧 권한 단위).
- 발송 가능 채널은 계약/심사 범위에 따름(SMS/LMS/MMS, 카카오 알림톡/친구톡 등).
- 출처: `_workspace/ppurio/02_validation.md` (인증 섹션, "별도 OAuth scope 없음(계정 단위 권한)")

---

## 환경변수 매핑

### 비즈뿌리오 (1차 구현 대상)

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `BIZPPURIO_ACCOUNT` | 연동 심사 통과 후 발급된 **비즈뿌리오 계정 ID** (토큰 발급 Basic 인증용) | 예 |
| `BIZPPURIO_PASSWORD` | 발급된 **계정 암호** (Base64 인증용) | 예 |
| `BIZPPURIO_BASE_URL` | 베이스 URL. 기본 `https://api.bizppurio.com` (검수: `https://dev-api.bizppurio.com:10443`) | 아니오(기본값 제공) |
| `PPURIO_DEFAULT_SENDER` | **사전등록된 발신번호**. 지정 시 발송 도구의 `from` 생략 가능 | 아니오 |

### 뿌리오 send-api (대안 경로 — send-api 제품 계약 시)

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `PPURIO_ACCOUNT` | 뿌리오 send-api 계정 ID | send-api 빌드 시 예 |
| `PPURIO_API_KEY` | 연동 신청 시 발급되는 **연동 인증키**(고정 IP 등록 후 발급) | send-api 빌드 시 예 |
| `PPURIO_BASE_URL` | 기본 `https://message.ppurio.com` | 아니오(기본값 제공) |

> 시크릿은 **환경변수로만** 주입합니다. 코드·`.mcp.json` 에 실값 하드코딩 금지(`${VAR}` 확장 사용).

---

## 출처 (공식 URL)

- 비즈뿌리오 공식 규격서(GitHub): https://github.com/bizppurio-api/api-documentation
- 비즈뿌리오 전체 매뉴얼(LLM export): https://biztech.gitbook.io/webapi/llms-full.txt
- 비즈뿌리오 GitBook 개요: https://biztech.gitbook.io/webapi
- 카카오 상태코드: https://biztech.gitbook.io/webapi/status-code/at-ai-ft
- 뿌리오 send-api 가이드: https://www.ppurio.com/send-api/guide
- 회원가입/발신번호 사전등록: https://www.ppurio.com/user/join/auth
- 발신번호 사전등록 근거 법령(전기통신사업법): https://law.go.kr/LSW/lsLinkProc.do?lsNm=전기통신사업법&chrClsCd=010202&mode=20

---

## 미확인 / 주의

- **콘솔 내 정확한 "연동 신청" 메뉴 경로 / API 키 화면 위치**: 공식 문서가 "기업회원 가입 → 연동 신청·심사(영업일 1~2일)"만 명시하고, 로그인 후 콘솔의 정확한 클릭 경로(메뉴 위치)는 공개 문서에 명문화되어 있지 않음 → **미확인**(가입·심사 후 발급 안내를 따를 것).
- **send-api 절대 호스트**(`https://message.ppurio.com`): 공식 가이드는 경로만 명시하고 호스트는 발급 시 안내됨. 실호출 코드 교차검증으로 확정했으나 1차 공식 본문 단정은 아님 → 사용 전 발급 안내값으로 확인 권장.
- **비즈뿌리오 레이트리밋 정확 수치**: 미공개(미확인). 메커니즘만 확정(초과 시 HTTP 429 + code 5002, 응답헤더 `RateLimit-Limit/Remaining/Reset`) → 헤더 기반 동적 백오프로 처리(수치 하드코딩 금지).
- **"접수 ≠ 전달"**: 발송 응답 `code:1000` 은 **접수 성공일 뿐 실제 전달이 아님**. 전달 여부는 `get_send_result`(PULL) 또는 PUSH 결과코드(SMS 4100, LMS/MMS 6600, 카카오 7000 등)로 판정.
