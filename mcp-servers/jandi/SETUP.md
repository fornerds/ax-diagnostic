# 키 발급 가이드 — 잔디 (jandi)

잔디의 공개 표면은 **인커밍 웹훅**뿐입니다. 발신측 공개 REST/OAuth/SDK는 없으며, "키"는 곧 **발급된 웹훅 URL 전체**(경로에 토큰 내장)입니다. 그래서 발급 = 잔디 앱에서 대화방별 웹훅 URL을 만들어 복사하는 것이 전부입니다.

## 연동 유형 / 인증 방식
- **연동 유형**: 잔디 커넥트 **인커밍 웹훅**(단방향 발신, POST). 조회/검색/관리 API는 존재하지 않음.
- **인증 방식**: **URL 내장 토큰**. 발급된 웹훅 URL의 경로(`.../webhook/{token}`)에 토큰이 포함되어 있고, 그 URL 자체가 인증 수단입니다. 별도 인증 헤더·OAuth·토큰 교환 **없음**.
- **필수 요청 헤더** (호출 시):
  - `Accept: application/vnd.tosslab.jandi-v2+json`
  - `Content-Type: application/json`
- **API 버전**: 미디어타입 v2 (`application/vnd.tosslab.jandi-v2+json`). 안정 버전, 폐기 공지 없음.

## 발급 절차 (스텝) — 표준 토픽 인커밍 웹훅 (기본)
1. 잔디 앱(웹/데스크톱)에 접속합니다.
2. **커넥트(설정) > 인커밍 웹훅 > "연동 추가"** 로 이동합니다.
3. 메시지를 받을 **대화방(토픽)을 선택**합니다.
4. 발급된 URL을 복사합니다 — 형식: `https://wh.jandi.com/connect-api/webhook/{token}`
5. 이 URL 전체를 환경변수 `JANDI_INCOMING_WEBHOOK_URL`에 설정합니다.

> **대화방 = URL 1:1 고정.** 웹훅 URL 1개는 대화방(토픽) 1개에 묶입니다. 런타임에 채널/DM을 임의 지정할 수 없으므로, 보낼 대화방마다 URL을 발급해 환경변수로 매핑하세요(다중 라우팅은 `JANDI_INCOMING_WEBHOOK_URL_{ALIAS}` 패턴).

## 발급 절차 (스텝) — Team 인커밍 웹훅 (선택, 유료 팀 전용)
특정 멤버(들)에게 email 라우팅으로 보내야 할 때만 사용합니다. **유료 팀 + 관리자/소유자만 생성 가능**합니다.
1. 잔디 앱에서 **전체 멤버용 Team Incoming Webhook** 메뉴로 이동합니다(관리자/소유자 권한 필요).
2. 발급된 URL을 복사합니다 — 형식: `https://wh.jandi.com/connect-api/team-webhook/{teamId}/{token}`
3. 이 URL 전체를 환경변수 `JANDI_TEAM_WEBHOOK_URL`에 설정합니다.
4. 호출 시 `email`(잔디 가입 이메일, 콤마구분 최대 100명) 필드가 **필수**입니다.

## 전제조건
- **사업자/제휴 심사**: 없음 (표준 토픽 인커밍 웹훅 기준).
- **발신번호 사전등록**: 없음 (SMS가 아닌 웹훅 메시지).
- **플랜 / 권한**:
  - **표준 토픽 인커밍 웹훅**: 자가발급·무료 워크스페이스 가능. 단, 워크스페이스 요금제·관리자 정책에 따라 커넥트(인커밍 웹훅) 사용이 제한될 수 있습니다. 발급 화면이 안 보이면 워크스페이스 관리자/플랜을 확인하세요.
  - **Team 인커밍 웹훅**: **유료 팀 + 관리자/소유자만 생성** 가능("해당 기능은 유료 팀에만 적용 가능합니다").

## 스코프 / 권한
- **스코프 개념 없음** — 인커밍 웹훅에는 OAuth 스코프가 없습니다.
- 발급 권한 = 워크스페이스 멤버/관리자(플랜 의존). Team 웹훅은 유료 팀 관리자/소유자 한정.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `JANDI_INCOMING_WEBHOOK_URL` | 잔디 앱 > 커넥트 > 인커밍 웹훅 > 연동 추가 → 대화방 선택 후 발급된 전체 URL(`https://wh.jandi.com/connect-api/webhook/{token}`) | 필수(기본 도구 `send_jandi_message`용) |
| `JANDI_INCOMING_WEBHOOK_URL_{ALIAS}` | 추가 대화방용으로 같은 방법으로 발급한 전체 URL. 예: `JANDI_INCOMING_WEBHOOK_URL_ALERTS` | 선택(다중 대화방 라우팅) |
| `JANDI_TEAM_WEBHOOK_URL` | 잔디 앱(유료 팀, 관리자/소유자) Team Incoming Webhook에서 발급된 전체 URL(`https://wh.jandi.com/connect-api/team-webhook/{teamId}/{token}`) | 선택(설정 시에만 `send_jandi_team_message` 도구 등록) |

> **시크릿 취급**: 웹훅 URL 자체가 인증 토큰입니다. 코드/공유 설정에 평문으로 두지 말고 `.env`(gitignore)에만 넣으세요. `.mcp.json`에는 `${VAR}` 환경변수 확장만 사용합니다.

## 출처 (공식 URL)
- 잔디 커넥트 소개: https://support.jandi.com/ko/articles/6352675-잔디-커넥트가-무엇인지-궁금합니다
- 인커밍 웹훅(표준 토픽) 공식 문서: https://support.jandi.com/ko/articles/6352697-잔디-커넥트-인커밍-웹훅-incoming-webhook-으로-외부-데이터를-잔디로-수신하고-싶습니다
- Team 인커밍 웹훅(전체 멤버·email 라우팅): https://support.jandi.com/ko/articles/6352712-전체-멤버에게-team-incoming-webhook을-발송하고-싶습니다
- (참고) 아웃고잉 웹훅: https://support.jandi.com/en/articles/Using-Team-Outgoing-Webhook-for-All-Members-aea2650b

## 미확인 / 주의
- **공개 개발자 포털(developers.jandi.com) 미확인**: 공식 도메인을 확인하지 못했습니다. 연동 문서는 support 헬프센터에만 존재.
- **OPEN API / SCIM(조직도·사용자 프로비저닝) 미확인**: 공식 근거를 확인하지 못했으며, 있어도 관리자 전용으로 본 커넥터 범위 밖.
- **인커밍 웹훅 토큰 재발급/폐기(회전) 절차 미확인**: 공식 문서에 절차 미기재. 유출 시 잔디 앱에서 해당 연동 삭제 후 재발급 권고(미확정).
- **호스트 주의**: 현행 호스트는 두 엔드포인트 모두 `wh.jandi.com`입니다. 영문 구버전 페이지의 `ws.jandi.com`은 스테일 표기이므로 사용 금지 — 발급된 전체 URL을 그대로 환경변수에 넣으세요.
- **런타임 오류 안내**: 401/403/404는 URL 폐기·오타 또는 워크스페이스 플랜에서 인커밍 웹훅 비활성 가능성을 의미합니다. 잔디 앱 커넥트에서 URL 재발급을 확인하세요.
- **헬프센터 URL 마이그레이션**: 숫자만 있는 구 URL(예: `.../articles/6352697`)은 404가 날 수 있습니다. 위 출처는 현행 슬러그본입니다.
