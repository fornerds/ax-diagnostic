# 키 발급 가이드 — 스윗(Swit) (swit)

> 검증 출처: `_workspace/swit/02_validation.md` (검증일 2026-06-26, route=build, verdict=spec_ready) 및 `mcp-servers/swit/README.md`. 모든 사실은 아래 "출처"의 공식 문서로 확인됨. 추측은 "미확인"으로 명시.

## 연동 유형 / 인증 방식

- **연동 유형**: 공식 원격 MCP/커넥터 **없음** → 로컬 MCP 빌드. Swit은 OAuth 앱(Swit Store) 모델만 제공.
- **인증 방식**: **OAuth 2.0 Authorization Code Grant**. 호출 시 헤더 `Authorization: Bearer {access_token}`.
- **토큰 만료**: `expires_in = 604800`초(**7일**). 만료 임박 또는 **401 수신 시 refresh_token으로 갱신**.
- **런타임 운용 권고**: 토큰 발급(authorize→code→token)은 1회성 부트스트랩으로 분리. 런타임 MCP 서버는 `SWIT_ACCESS_TOKEN`(+선택적 refresh)만 사용.

## 발급 절차 (스텝)

1. **개발자 콘솔 접속**: `https://developers.swit.io/apps` → **New app** 으로 앱 생성. (셀프서비스 — 개발용 앱은 심사 없이 생성 가능)
2. **Authentication 탭**: `client_id` / `client_secret` 확인, **Allowed redirect URLs** 등록.
3. **Scopes 탭**: 사용할 스코프 등록(authorize 요청에는 여기 등록된 스코프만 전달 가능).
4. **Authorize (인가 코드 받기)**:
   `GET https://openapi.swit.io/oauth/authorize?client_id=...&redirect_uri=...&response_type=code&scope=...&state=...`
   - `response_type` 은 `code` 만 지원, `scope` 는 **공백 구분**.
5. **Token 교환 (액세스 토큰 받기)**:
   `POST https://openapi.swit.io/oauth/token`
   - `Content-Type: application/x-www-form-urlencoded`
   - body: `grant_type=authorization_code, client_id, client_secret, redirect_uri, code`
   - 응답: `{ access_token, expires_in: 604800, refresh_token, token_type: "Bearer", scope }`
6. **갱신(필요 시)**: 같은 token endpoint 에 `grant_type=refresh_token, client_id, client_secret, refresh_token`. (서버는 401 수신 시 자동 갱신 후 1회 재시도)

### Incoming Webhook URL 발급 (발신 전용, 별도)

- `send_incoming_webhook` 용 URL은 **채널 UI에서 발급**한다. 형식: `https://hook.swit.io/chat/<CHANNEL_ID>/<WEBHOOK_ID>`. 토큰 불필요, body `{"text":"…"}`.

## 전제조건

- **플랜**: MCP 기본 도구 세트는 일반 플랜에서 동작. `admin:read|write` 스코프는 **Advanced 플랜 전용**(MCP 기본 세트에서 제외).
- **사업자 심사 / 발신번호 사전등록 / 실명인증**: **없음** (사내 협업툴이며 SMS/알림톡이 아니므로 레거시 발신번호·실명인증 규제 비해당).
- **production 전환 심사**: 자기 조직 내 개인/봇 토큰으로만 호출하는 본 MCP 용도에는 **스토어 등록·심사 불요**. (외부 조직 배포/미검증 경고 제거 시에만 Swit 리뷰팀 심사 필요 — 본 용도엔 비해당. 단 "전혀 무심사"의 명문은 없음 → 아래 "미확인" 참조)
- **봇 모드 운용 시**: 봇이 대상 **채널/프로젝트에 초대돼 있어야** 메시지/태스크/댓글 작성 가능(미초대 시 403).

## 스코프 / 권한

- **스코프 모델**: `{resource}:{read|write}` (read=GET, write=POST).
- **MCP 기본 세트(권장)**:
  ```
  message:read message:write channel:read channel:write task:read task:write project:read workspace:read user:read idea:read idea:write
  ```
- **봇 모드**: authorize 에 **`app:install`** 추가 → 봇 사용자(bot) 토큰 발급.
- **제외**: `admin:read|write`(Advanced 플랜 전용 + 관리 용도).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `SWIT_ACCESS_TOKEN` | OAuth Token 교환 응답의 `access_token`(Bearer). 런타임 호출에 사용. | 예 |
| `SWIT_REFRESH_TOKEN` | Token 교환 응답의 `refresh_token`. 401/만료 시 자동 갱신용. | 권장 |
| `SWIT_CLIENT_ID` | `developers.swit.io/apps` → Authentication 탭. 자동 갱신 시 필요. | 자동갱신 시 |
| `SWIT_CLIENT_SECRET` | `developers.swit.io/apps` → Authentication 탭. **절대 노출 금지.** | 자동갱신 시 |
| `SWIT_REDIRECT_URI` | 앱에 등록한 Allowed redirect URL. 토큰 발급/갱신 절차 구현 시. | 조건부 |
| `SWIT_API_BASE` | 기본 `https://openapi.swit.io/v1` (오버라이드용, 미설정 시 기본값). | 아니오 |

> 시크릿은 전부 env로만. 하드코딩 금지. `.env` 는 커밋하지 않는다(`.gitignore` 포함).

## 출처 (공식 URL)

- 개발자 콘솔(앱 등록·client_id/secret): https://developers.swit.io/apps
- OAuth flow(authorize/token/scope/`app:install`/만료): https://devdocs.swit.io/docs/guides/25i4ipzs76uyw-o-auth-flow
- 개발자 콘솔/스코프/봇 초대 전제·Distribution(심사): https://devdocs.swit.io/docs/guides/bkbnq5rnz3ms8-explore-the-swit-developer-console
- 베이스 URL·경로 규약(Core API v1): https://devdocs.swit.io/docs/core1/ref
- Incoming Webhook(공식 호스트 발급): https://help.swit.io/docs/using-swit/incoming-webhooks?lang=en
- Rate limits: https://devdocs.swit.io/docs/guides/nm5k7dn0fxlz9-rate-limits
- Status codes: https://devdocs.swit.io/docs/guides/ooop1b3rurloy-status-codes
- (참고) 구 문서 URL `https://developers.swit.io/documentation` 는 **404로 폐기** → 현행 레퍼런스는 `devdocs.swit.io`.

## 미확인 / 주의

- **production "무심사" 명문**: 셀프서비스로 앱 생성·토큰 발급은 가능하나, 자기 조직 내 사용이 "전혀 무심사"라는 공식 명문은 확인되지 않음 → **미확인**. 외부 조직 배포·미검증 경고 제거 시에는 리뷰팀 심사 필요.
- **응답 스키마 전체 필드**: 요청 파라미터/핵심 필드는 OpenAPI로 확정했으나 응답 모델 전 필드는 일부만 확인 → 도구 출력은 응답 JSON passthrough 권고(가공 최소화).
- **봇 토큰 함정**: 봇 모드에서 작성 요청이 403이면 "봇 미초대"가 가장 흔한 원인. 또한 `create_channel` 은 봇 토큰 비호환(사용자 토큰으로 호출).
- **`to_status_id`(태스크 상태 변경)**: 프로젝트별 동적 ID. 하드코딩 금지 — `list_project_statuses` 로 먼저 조회.
- **`offset`(페이지네이션)**: 불투명 커서 문자열(정수 아님). 응답 `data.offset` 값을 그대로 다음 요청에 전달.
