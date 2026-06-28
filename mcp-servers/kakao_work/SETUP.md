# 키 발급 가이드 — 카카오워크 (kakao_work)

> **주의: 아래 발급 경로(메뉴/버튼)는 미확인 — 정확하지 않을 수 있습니다.** 발급 콘솔이 로그인 게이트/계정별이라 외부에서 화면을 확인하지 못했습니다. 실제 UI와 다를 수 있으니 공식 콘솔에서 직접 확인하세요. (추적: `mcp-servers/_SETUP_GAPS.md`)

> 출처: 검증본 `_workspace/kakao_work/02_validation.md` 및 `mcp-servers/kakao_work/README.md`.
> 원칙: 출처 URL 있는 사실만 기재. 불명확한 항목은 "미확인"으로 남김.

## 연동 유형 / 인증 방식

- **연동 유형**: 공식 원격 MCP 커넥터 없음 → 로컬 stdio MCP 서버 빌드(route=build).
  - (카카오워크의 "MCP"는 카카오워크가 외부를 소비하는 역방향 기능이며, claude.ai 커넥터 디렉터리에 카카오워크는 존재하지 않음.)
- **인증 방식**: 봇 **App Key**를 Bearer 토큰으로 전달. **OAuth 아님.**
  - 헤더: `Authorization: Bearer ${KAKAOWORK_APP_KEY}` + `Content-Type: application/json`
  - 토큰 흐름: 정적 키(서버 시작 시 env 주입). 토큰 교환/갱신 흐름 없음.
- **베이스 URL**: `https://api.kakaowork.com/v1`
- 출처: https://docs.kakaoi.ai/kakao_work/webapireference/commonguide/ , https://docs.kakaoi.ai/kakao_work/botdevguide/process/

## 발급 절차 (스텝)

App Key는 **워크스페이스 관리자**가 카카오워크 Admin에서 발급한다. 외부 사용자 단독 발급 불가.

1. 카카오워크 워크스페이스 **Admin(관리자)** 에 접속한다(관리자 권한 필요).
2. **개발자 등록**을 진행한다.
3. **봇 생성**을 한다.
4. 생성한 봇의 **"Bot 기본정보" 탭**에서 **App Key**를 확인한다.
5. 확인한 App Key를 환경변수 `KAKAOWORK_APP_KEY`로 주입한다(코드/저장소 하드코딩 금지).

- 출처: https://docs.kakaoi.ai/kakao_work/botdevguide/process/

## 전제조건

- **외부 심사 / 제휴 / 승인**: **없음** (self-service, 워크스페이스 관리자 권한만 필요).
- **사업자 심사 · 발신번호 사전등록 · 실명/공동인증 등 통신규제 전제**: **없음** (워크스페이스 내 봇 메시지이므로 해당 없음).
- **필요 권한**: 워크스페이스 관리자(Admin) 권한.
- 출처: https://docs.kakaoi.ai/kakao_work/botdevguide/process/

## 스코프 / 권한

- 명시적 **OAuth scope 체계 없음**. 단일 봇 App Key로 본 커넥터의 도구를 호출한다.
- API별 권한 토글/scope 표는 공식 문서에서 확인되지 않음(아래 "미확인" 참조). 단일 App Key Bearer로 본 스펙 작업 호출이 가능한 것으로 보임.
- **봇 컨텍스트 한정**: `conversations.*` 조작은 "봇이 만든/참여한 채팅방" 범위로 한정될 가능성. `messages.send`의 `conversation_id`도 봇이 접근 가능한 방이어야 함.
- 권한/만료 관련 실패는 런타임에서 인증 에러 코드(`invalid_authentication` / `expired_authentication` / `unauthorized`)로 안내한다.
- 출처: https://docs.kakaoi.ai/kakao_work/webapireference/commonguide/

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `KAKAOWORK_APP_KEY` | Admin → 개발자 등록 → 봇 생성 → "Bot 기본정보" 탭의 App Key. Bearer 토큰으로 모든 API 인증 | 예 |
| `KAKAOWORK_BASE_URL` | 베이스 URL 오버라이드. 미설정 시 기본값 `https://api.kakaowork.com/v1` | 아니오 |
| `KAKAOWORK_ENABLE_KICK` | `true`면 파괴적 도구 `kick_from_conversation` 노출(기본 비활성) | 아니오 |

> 시크릿(App Key)은 코드/저장소에 **하드코딩 금지**(공식 문서 보안 경고). `.env`/환경변수로만 주입하고, 저장소에는 `.env.example`(키 이름만)을 커밋한다.

## 출처 (공식 URL)

- Bot 개발 프로세스(App Key 발급): https://docs.kakaoi.ai/kakao_work/botdevguide/process/
- 공통 가이드(인증·레이트리밋·페이지네이션·에러): https://docs.kakaoi.ai/kakao_work/webapireference/commonguide/
- Web API 인덱스: https://docs.kakaoi.ai/kakao_work/webapireference/
- 커넥터(route=build 근거): https://support.claude.com/en/articles/11176164-use-connectors-to-extend-claude-s-capabilities
- 카카오워크 MCP(역방향 기능, 혼동 주의): https://kakaowork.gitbook.io/kakao-work/workservice/ai/undefined-1/mcp

## 미확인 / 주의

- **App Key 만료 / 회전 / 재발급 정책**: 공식 문서가 침묵. 유출 주의 경고만 존재. → **미확인.**
- **사용자 OAuth scope 체계**: 공식 근거 미발견. API 인증은 봇 App Key Bearer만 확인됨. → **미확인.**
- **API별 권한 토글 / scope 표**: 공식 문서에서 못 찾음. → **미확인.**
- **레이트리밋**: 워크스페이스 단위 분당 200회, 초과 시 429. `retry-after`(초) 존중 권장.
- **콘솔 정확 URL**: 발급은 "워크스페이스 Admin" 내에서 이뤄지나, 절차 출처(botdevguide/process)는 단계만 명시하고 고정 콘솔 URL은 명시하지 않음. 워크스페이스마다 Admin 진입 경로가 다를 수 있음. → 진입 경로 세부는 **미확인**.
