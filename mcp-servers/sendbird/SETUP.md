# 키 발급 가이드 — 센드버드(Sendbird) (sendbird)

> 근거: `_workspace/sendbird/02_validation.md` (verdict = spec_ready) + Sendbird 공식 문서. 추측 없이 출처 있는 사실만 기재한다.

## 연동 유형 / 인증 방식

- **연동 유형:** REST API (Chat Platform API **v3**). 공식 원격 MCP 커넥터는 없음 → 로컬 서버 빌드 + 정적 토큰 주입.
- **인증 방식:** API Key (정적 토큰). OAuth / 토큰 교환 / 리프레시 **없음**.
- **헤더:** 모든 요청에 `Api-Token: {SENDBIRD_API_TOKEN}` + `Content-Type: application/json; charset=utf8`.
- **토큰 종류:** 마스터 API 토큰 1개(앱당 고정, 폐기·변경 불가) + 세컨더리 토큰(생성/조회/폐기 가능). **운영에는 세컨더리 토큰 권장** — 마스터는 유출 시 폐기할 수 없어 치명적.
- **중요:** API는 **서버 전용**이다. 클라이언트 앱에서 호출 금지(토큰 유출 위험) → 토큰은 서버 env에만 보관.

## 발급 절차 (스텝)

1. **대시보드 가입/로그인:** `https://dashboard.sendbird.com` 접속. 신규는 `https://dashboard.sendbird.com/auth/signup` 에서 셀프 가입(신용카드·사업자·실명 심사 불요).
2. **애플리케이션 생성/선택:** 대시보드에서 사용할 애플리케이션을 만들거나 선택한다.
3. **Application ID 확인:** **Settings > Application > General** 에서 `Application ID` 확인 → `SENDBIRD_APPLICATION_ID`. (대소문자 구분, 하드코딩 금지)
4. **마스터 토큰 확인:** 같은 **Settings > Application > General** 화면의 **API tokens** 섹션에서 마스터 API 토큰 확인.
5. **세컨더리 토큰 생성(권장):** 동일 API tokens 영역에서 세컨더리 토큰을 발급 → 이 값을 `SENDBIRD_API_TOKEN` 으로 사용. (마스터 대신 세컨더리를 쓰면 유출 시 폐기/회전 가능)
6. **env 주입:** 위 두 값을 서버 환경변수로 주입(`.env.example` 복사 → `.env`).

## 전제조건

- **플랜:** 무료 Developer 플랜으로 즉시 시작 가능. 별도 유료 전환 불필요.
- **사업자/실명 심사:** **없음** (셀프 가입, 신용카드 불요).
- **발신번호 사전등록 등 한국 규제:** Chat 범위에서는 **없음**. (알림톡/SMS 등 Business Messaging은 별도 제품이며 본 MCP 범위 밖 — 미확인)
- **구현 차단 요인:** 없음. 가입 직후 마스터 토큰 발급 → 세컨더리 토큰 생성하여 즉시 사용 가능.

## 스코프 / 권한

- **세분 스코프 체계 없음.** 마스터·세컨더리 토큰 모두 **애플리케이션 단위 전체 권한**을 가진다. OAuth식 scope 협상 개념 없음.
- 마스터/세컨더리 차이:
  - 마스터 = 세컨더리 토큰을 발급/삭제할 수 있으나 **자신은 폐기·변경 불가**.
  - 세컨더리 = 애플리케이션 전체 권한 + **폐기/회전 가능**.
- 최소권한 분리가 불가하므로 멀티테넌트로 쓰려면 **애플리케이션·토큰 자체를 분리**해야 한다(MCP 서버는 단일 앱·단일 토큰 전제).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|---|---|---|
| `SENDBIRD_APPLICATION_ID` | 대시보드 **Settings > Application > General** 의 `Application ID`. 베이스 URL 호스트 조립(`api-{id}.sendbird.com`)에 사용. **대소문자 구분**. | 예 |
| `SENDBIRD_API_TOKEN` | 같은 화면 **API tokens** 섹션의 토큰. **세컨더리 토큰 권장**(마스터는 폐기 불가). `Api-Token` 헤더 값. 서버 전용·하드코딩 금지. | 예 |
| `SENDBIRD_API_BASE_URL` | (선택) 베이스 URL 전체 오버라이드(테스트/프록시용). 미설정 시 `https://api-${SENDBIRD_APPLICATION_ID}.sendbird.com/v3` 자동 조립. | 아니오 |

## 출처 (공식 URL)

- API 준비/인증·토큰·베이스 URL: https://sendbird.com/docs/chat/platform-api/v3/prepare-to-use-api
- 토큰 보안(마스터/세컨더리): https://sendbird.com/docs/security/documentation/app/sendbird-api-token-security-guide
- 대시보드: https://dashboard.sendbird.com (가입: https://dashboard.sendbird.com/auth/signup)
- 무료 플랜/트라이얼: https://sendbird.com/free-trial , https://sendbird.com/blog/build-chat-for-free-introducing-the-new-developer-plan
- API 버전(v3 현행): https://sendbird.com/docs/chat/platform-api/v3/overview

## 미확인 / 주의

- **베이스 URL은 애플리케이션별로 다르다:** `https://api-{application_id}.sendbird.com/v3`. `application_id`가 틀리면 모든 호출이 실패한다. 하드코딩 금지·런타임 조립 필수.
- **레이트리밋이 낮다(무료 플랜):** GET 10/s, 쓰기 5/s, 메시지 발송 5/s 고정. 429(`code=500910`) 발생 시 `X-RateLimit-RetryAfter` 기반 백오프 필요.
- **버전 혼동 주의:** "Chat Platform API v3 deprecated"는 오정보(클라이언트 SDK v4와 혼동). Platform API는 v3가 현행 최신이며 v4 Platform API는 없다.
- **범위 밖(미확인):** 모더레이션 개별 엔드포인트, 데이터 익스포트(async job), Business Messaging(알림톡/SMS, 한국 발신번호·템플릿 승인 규제)은 본 커넥터 범위에 포함되지 않으며 발급 전제도 별도 조사 필요.
- 대시보드 UI 메뉴 명칭은 Sendbird 개편에 따라 일부 달라질 수 있다. 핵심 위치는 Settings > Application > General(Application ID + API tokens)이다.
