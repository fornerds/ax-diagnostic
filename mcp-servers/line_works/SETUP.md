# 키 발급 가이드 — 네이버웍스(LINE WORKS) (line_works)

> **주의: 아래 발급 경로(메뉴/버튼)는 미확인 — 정확하지 않을 수 있습니다.** 발급 콘솔이 로그인 게이트/계정별이라 외부에서 화면을 확인하지 못했습니다. 실제 UI와 다를 수 있으니 공식 콘솔에서 직접 확인하세요. (추적: `mcp-servers/_SETUP_GAPS.md`)

> 출처 기준: `_workspace/line_works/02_validation.md`(검증일 2026-06-26) · `mcp-servers/line_works/README.md`. 모든 사실은 아래 "출처(공식 URL)"의 LINE WORKS 공식 개발자 문서로 검증됨. 추측 없음 — 불명확한 항목은 "미확인 / 주의"에 명시.

## 연동 유형 / 인증 방식

- **연동 유형:** 로컬 MCP 서버 빌드(공식 Claude/원격 MCP 커넥터 없음). REST 직접 호출, 전송 stdio.
- **인증 방식(1차 구현):** Service Account JWT (`grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`).
  - RSA 개인키로 RS256 JWT(assertion) 서명 → `POST https://auth.worksmobile.com/oauth2/v2.0/token` → `access_token` 획득 → 모든 API 호출에 `Authorization: Bearer {access_token}`.
  - JWT 헤더 `{"alg":"RS256","typ":"JWT"}`, 클레임 `iss`=Client ID, `sub`=Service Account 이메일, `iat`=now, `exp`=now+최대 3600초.
- **선택 방식(Phase 2, 미구현):** OAuth 2.0 Authorization Code — mail/drive 등 사용자 위임 작업용. authorize/token/revoke = `https://auth.worksmobile.com/oauth2/v2.0/{authorize|token|revoke}`.
- 출처: https://developers.worksmobile.com/en/docs/auth-jwt , https://developers.worksmobile.com/kr/docs/api-call

## 발급 절차 (스텝)

1. **테넌트 가입** — LINE WORKS 테넌트(무료 플랜 포함) 가입. 외부 사업자 심사 없이 시작 가능.
2. **앱 등록** — [Developer Console](https://developers.worksmobile.com/) 접속 → 앱 등록 → **Client ID / Client Secret** 발급.
3. **Service Account 생성** — 콘솔에서 Service Account(이메일) 생성 + **RSA 개인키(PEM)** 발급(RS256 서명용).
4. **(봇 사용 시) Bot 등록** — 콘솔에서 **Bot 생성 후 테넌트(도메인)에 추가/공개** 선행. 이 절차를 거쳐야 `bot_id`가 유효해진다. 출처: https://developers.worksmobile.com/en/docs/bot-domain-register
5. **스코프 확인/부여** — 발급할 스코프를 토큰 요청에 포함(기본 `bot bot.message user.read directory.read orgunit.read calendar`).
6. **값 확인 → 환경변수 주입** — Client ID/Secret, Service Account 이메일, 개인키(PEM)를 아래 "환경변수 매핑"에 따라 `.env`에 채운다.

## 전제조건

- **요금제(플랜):** 대상 테넌트의 요금제(Free/Standard/Advanced)에 따라 사용 가능한 스코프·레이트리밋이 다름. 무료 플랜에서도 시작 가능(분당 한도 Free 60). 권한 부족 시 403, 한도 초과 시 429 가 그대로 노출됨.
  - 출처: https://developers.worksmobile.com/jp/docs/rate-limits , https://developers.worksmobile.com/en/docs/auth-scope
- **사업자 심사:** 없음(외부 사업자 심사 없이 시작 가능 — 출처 02_validation §E).
- **발신번호 사전등록:** 해당 없음(메시지는 SMS가 아닌 봇 메시지. 출처 bot-user-message-send).
- **Bot 사전 등록:** 봇 메시지 도구를 쓰려면 Bot 생성 + 도메인 추가/공개 선행 필요(위 4단계).
- **정부(공공) 테넌트:** API 호스트 `gov.worksapis.com`, 인증 호스트 `auth.gov-naverworks.com`. `LINEWORKS_REGION=gov`로 분기. 출처: https://developers.worksmobile.com/kr/docs/gov

## 스코프 / 권한

1차 구현 도구가 요구하는 스코프(공백 구분 1세트로 토큰 발급):

`bot bot.message user.read directory.read orgunit.read calendar`

| 스코프 | 사용 도구 | 비고 |
|---|---|---|
| `bot` / `bot.message` | `send_message_to_user`, `send_message_to_channel` | 봇 메시지 발송 |
| `user.read` (또는 `user` / `directory.read`) | `list_users`, `get_user` | 구성원 조회 |
| `orgunit.read` (또는 `orgunit` / `directory.read`) | `list_orgunits` | 조직(부서) 조회 |
| `calendar` (조회는 `calendar.read` 가능) | `create_calendar_event`, `list_calendar_events` | 기본 캘린더 일정 |

- Service Account(JWT)로 위 스코프 모두 호출 가능(JWT 불가 목록 = mail/file/drive/group.note/group.folder/task/form, 이번 범위 외).
- 출처: https://developers.worksmobile.com/en/docs/auth-scope , https://developers.worksmobile.com/en/docs/auth-jwt

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|---|---|---|
| `LINEWORKS_CLIENT_ID` | Developer Console 앱의 Client ID (JWT `iss`, token 요청 `client_id`) | 필수 |
| `LINEWORKS_CLIENT_SECRET` | Developer Console 앱의 Client Secret (token 요청) | 필수 |
| `LINEWORKS_SERVICE_ACCOUNT` | 콘솔에서 생성한 Service Account 이메일 (JWT `sub`) | 필수 |
| `LINEWORKS_PRIVATE_KEY` | Service Account RSA 개인키(PEM, RS256). 한 줄로 넣을 땐 줄바꿈을 `\n`으로 이스케이프 | 필수* |
| `LINEWORKS_PRIVATE_KEY_PATH` | PEM 파일 경로(값 대신 파일로 관리). 설정 시 `LINEWORKS_PRIVATE_KEY`보다 우선 | 필수* |
| `LINEWORKS_SCOPES` | 발급 스코프(공백구분). 기본 `bot bot.message user.read directory.read orgunit.read calendar` | 선택 |
| `LINEWORKS_DEFAULT_BOT_ID` | 메시지 도구에서 `bot_id` 미지정 시 기본값. 콘솔 Bot 등록 후 부여된 Bot ID | 선택 |
| `LINEWORKS_REGION` | `public`(기본) / `gov`. gov면 베이스 `gov.worksapis.com` + 인증 `auth.gov-naverworks.com` 사용 | 선택 |

\* `LINEWORKS_PRIVATE_KEY` 또는 `LINEWORKS_PRIVATE_KEY_PATH` 중 하나는 반드시 제공.

> 시크릿(Client Secret · Private Key)은 절대 코드/리포지토리에 하드코딩 금지. `.env`(gitignore) + `.env.example` 사용. `*.pem`/`*.key`도 gitignore 처리.

## 출처 (공식 URL)

- Developer Console: https://developers.worksmobile.com/
- API 공통/베이스 URL/헤더: https://developers.worksmobile.com/kr/docs/api-call
- Service Account(JWT) 인증: https://developers.worksmobile.com/en/docs/auth-jwt
- OAuth2 엔드포인트·토큰수명: https://developers.worksmobile.com/jp/docs/auth-oauth
- 스코프 목록: https://developers.worksmobile.com/en/docs/auth-scope
- Bot 등록/도메인 추가: https://developers.worksmobile.com/en/docs/bot-domain-register
- 레이트리밋: https://developers.worksmobile.com/jp/docs/rate-limits
- 에러코드: https://developers.worksmobile.com/jp/docs/error-codes
- 정부(공공) 테넌트: https://developers.worksmobile.com/kr/docs/gov

## 미확인 / 주의

- **무료 플랜 스코프 정확 허용범위:** 공식 스코프표는 Standard/Advanced 열만 노출(Free 열 없음). "Free=특정 5종" 같은 구체 목록은 **공식 미확인**(커뮤니티 근거). 런타임에서 403/scope 부족을 그대로 노출하고, 대상 테넌트 요금제를 사전 확인할 것.
- **토큰 수명:** Access Token 1시간 또는 24시간(콘솔 설정값). 응답 `expires_in`을 신뢰해 캐시 만료 산정. Refresh Token 90일, Authorization Code 10분(1회용). 출처 auth-oauth.
- **레거시 API 혼동 금지:** 구 "API 1.0"(`apis.worksmobile.com`, 제공종료예정)과 현행 `https://www.worksapis.com/v1.0` 혼동 금지. 발급한 키는 현행 API에만 사용.
- **콘솔 화면의 정확한 메뉴 명칭/슬러그:** 02_validation은 "콘솔 생성 후 도메인 추가" 절차의 존재를 확인했으나, 콘솔 UI의 정확한 메뉴 경로/버튼 라벨은 콘솔 접속 시점 기준 확인 필요(문서 슬러그 변동 가능).
- **개인키 발급 형식/파일명:** RSA 개인키(PEM)가 발급된다는 사실은 검증됨. 파일명 규칙(예: `private_YYYYMMDD.key`)은 예시이며 발급 시점 콘솔 표기를 따른다.
