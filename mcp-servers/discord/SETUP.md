# 키 발급 가이드 — Discord (discord)

## 연동 유형 / 인증 방식
- **연동 유형:** 로컬 MCP 서버 (Discord REST API v10 직접 호출). 공식 원격 MCP 커넥터는 없음.
- **인증 방식:** **Bot Token(봇 토큰)**. 모든 요청에 헤더 `Authorization: Bot <DISCORD_BOT_TOKEN>` 사용.
- 봇 토큰 자체가 런타임 자격이므로 **OAuth2 인가 코드 교환은 불필요**. OAuth2는 봇을 서버에 초대할 때(URL Generator)만 사용한다.
- 베이스 URL: `https://discord.com/api/v10` (반드시 v10 명시 — 미지정 시 레거시 v6로 라우팅됨).

## 발급 절차 (스텝)
1. **Developer Portal 접속:** <https://discord.com/developers/applications> 에 Discord 계정으로 로그인.
2. **Application 생성:** "New Application" 으로 앱 생성.
3. **Bot 추가 + 토큰 발급:** 앱의 **Bot** 설정으로 이동 → **Reset Token** 으로 봇 토큰 발급. 토큰은 **1회만 표시**되므로 즉시 복사해 `.env` 의 `DISCORD_BOT_TOKEN` 에 저장(분실 시 재생성 가능).
4. **Privileged Intents 활성화** (Bot 설정에서 ON):
   - **MESSAGE CONTENT INTENT** (`MESSAGE_CONTENT`) — 끄면 메시지 조회 결과의 `content` 가 빈 문자열로 온다.
   - **SERVER MEMBERS INTENT** (`GUILD_MEMBERS`) — 끄면 `list_guild_members` 가 실패/제한된다.
5. **봇 초대:** **OAuth2 → URL Generator** 에서 `bot` 스코프 + 권한 비트 선택 → 생성된 URL 로 대상 서버(길드)에 초대. 권한 없으면 50001(Missing Access) / 50013(권한 부족) 에러.

## 전제조건
- **심사/플랜/사업자인증/발신번호 사전등록: 없음.** 누구나 Discord 계정으로 Developer Portal 에서 앱·봇·토큰을 무료로 생성할 수 있다(한국 특화 제약 없음).
- 단, **100+ 길드에 배포**하려면 Discord 의 별도 봇 검증(verification)이 필요하다. 사내 소규모(단일/소수 서버) 사용에는 무관.
- 봇이 대상 길드에 초대되어 있어야 하고, 사용하는 도구에 맞는 권한 비트가 부여돼 있어야 한다(아래 스코프/권한 참고).

## 스코프 / 권한
- **OAuth2 스코프(초대 시):** `bot` (서버 초대용). `applications.commands` 는 `bot` 에 기본 포함.
- **필수 Privileged Intents:** `MESSAGE_CONTENT`, `GUILD_MEMBERS` (위 절차 4단계).
- **권장 권한 비트(URL Generator 에서 선택):** View Channels, Send Messages, Send Messages in Threads, Read Message History, Add Reactions, Attach Files.
  - 관리 도구(`modify_channel`, `create_guild_channel`)용: **Manage Channels**.
  - 웹훅 도구(`create_webhook`, `list_channel_webhooks`)용: **Manage Webhooks**.
- 본 MCP 는 봇 토큰으로 전 작업을 커버하므로 User OAuth2 Bearer 흐름은 범위 밖.

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `DISCORD_BOT_TOKEN` | Developer Portal → 앱 → Bot → Reset Token 으로 발급된 봇 토큰 | 필수 |
| `DISCORD_API_BASE` | API 베이스 오버라이드용. 기본 `https://discord.com/api/v10` | 선택 |
| `DISCORD_USER_AGENT` | `User-Agent` 헤더 값. 기본 `DiscordBot (https://github.com/your-org/mcp-discord, 1.0.0)` (모든 요청에 필수 헤더 — 누락 시 Cloudflare 차단) | 선택 |
| `DISCORD_DEFAULT_GUILD_ID` | 길드 ID 미지정 도구 호출 시 기본값(편의용) | 선택 |

> 시크릿(`DISCORD_BOT_TOKEN`)은 `.env` 로만 주입하고 코드/리포지토리/`.mcp.json` 에 하드코딩하지 않는다. `.env.example` 에는 키 이름만 둔다.

## 출처 (공식 URL)
- Developer Portal(앱·봇·토큰 발급): <https://discord.com/developers/applications>
- API 레퍼런스(베이스 URL·v10·인증 헤더·User-Agent 필수): <https://docs.discord.com/developers/reference>
- OAuth2 / 스코프(`bot` 등): <https://docs.discord.com/developers/topics/oauth2>
- 레이트리밋(전역 50 req/s, 헤더, 429): <https://docs.discord.com/developers/topics/rate-limits>
- 멤버 목록 intent 요건(`GUILD_MEMBERS`): <https://docs.discord.com/developers/resources/guild>

## 미확인 / 주의
- **길드 전체 메시지 검색 불가:** `GET /guilds/{guild.id}/messages/search` 는 봇 토큰으로 403 "Bots cannot use this endpoint"(code 20001). 계약에서 제외됨. 검색 수요는 `search_recent_messages`(채널 단위 최근 N건 스캔 후 클라이언트 측 키워드 필터)로 대체 — 전체 서버 인덱스 검색이 아님.
- **MESSAGE_CONTENT 미활성 시** 메시지 `content` 가 빈 문자열, **GUILD_MEMBERS 미활성 시** `list_guild_members` 실패 — Intent 활성화를 먼저 확인.
- **파괴적 작업:** `delete_message`, `modify_channel` 은 비가역. 명시한 단일 ID 만 처리.
- **봇 검증(100+ 길드) 임계값·절차 상세: 미확인** — 사내 소규모 사용엔 무관하나, 대규모 배포 시 Discord 문서로 별도 확인 필요.
- **Gateway Intents 상세 매핑 표: 미확인**(원문 표 미열람) — 단 본 REST 스펙에 필요한 2종(MESSAGE_CONTENT, GUILD_MEMBERS)은 리소스 문서로 개별 확인됨.
