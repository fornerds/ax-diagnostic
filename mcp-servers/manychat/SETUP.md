# 키 발급 가이드 — 매니챗 (manychat)

> 출처: `_workspace/manychat/02_validation.md` (검증일 2026-06-26, route=build). 추측 없이 검증된 사실만 기재. 발급 메뉴 경로는 공식 Help 검색 인덱싱본 기준(원문은 Cloudflare 403으로 직접 열람 불가) — 아래 "미확인 / 주의" 참조.

## 연동 유형 / 인증 방식
- **연동 유형:** 로컬 MCP 서버 빌드 (Manychat 공식 원격 MCP/Claude 커넥터 없음).
- **인증:** HTTP Bearer. 모든 요청 헤더 `Authorization: Bearer <MANYCHAT_API_TOKEN>`.
- **토큰 흐름:** 단일 워크스페이스(page) 개인 API Key. 토큰 교환/리프레시 없음(정적 키).
- **토큰 형식:** `<pageId>:<secret>` — 콜론을 포함한 1개 문자열 전체를 그대로 시크릿으로 보관 (예: `1234567890123456:1234567890ABCDEFGHIJKLMNOPQRSTUV`).
- **베이스 URL:** `https://api.manychat.com` (버전 경로 프리픽스 없음, `info.version=beta`).

## 발급 절차 (스텝)
1. Manychat 앱에 로그인한다 (https://app.manychat.com). **Pro 플랜이어야 API Key 생성 가능**(전제조건 참조).
2. **Settings → API** 메뉴로 이동한다.
3. **Generate your API Key**를 눌러 API Key를 생성한다.
4. 생성된 토큰(`<pageId>:<secret>` 형식 전체 문자열)을 복사해 환경변수 `MANYCHAT_API_TOKEN`에 넣는다.

> 토큰 1개 = 해당 page 전체 권한. 토큰을 갱신/삭제하면 그 토큰에 연결된 모든 API 메서드가 비활성화되므로 운영 시 주의한다.

## 전제조건
- **Manychat Pro 플랜 필수** (무료 플랜에서는 API 토큰 생성 불가). 출처: 공식 Help 인덱싱본.
- **사업자 심사 / 실명·공동인증 심사: 없음.** 키 발급은 셀프(즉시 발급).
- **발신번호 사전등록 등 한국 특이 절차: 없음** (해당 사실은 검증본에 별도 한국 규제 절차 기록 없음 — 발송 도달은 Meta/채널 정책에 종속, "미확인 / 주의" 참조).

## 스코프 / 권한
- 개인 API Key에 **세분화된 OAuth 스코프 없음** (spec에 scope 정의 없음).
- 키 1개 = 해당 page 전체 권한 (발송·구독자·태그·커스텀필드·Flow 등 연결된 모든 메서드).

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `MANYCHAT_API_TOKEN` | Manychat 앱 → Settings → API → Generate your API Key 로 생성한 `<pageId>:<secret>` 전체 문자열 | 필수 |
| `MANYCHAT_BASE_URL` | 베이스 URL 오버라이드용. 기본값 `https://api.manychat.com` (보통 설정 불필요) | 선택 |

## 출처 (공식 URL)
- 인증/베이스URL/토큰형식 (라이브 OpenAPI spec): https://api.manychat.com/swagger/compileJson?type=Page_API
- 토큰 형식 `<pageId>:<secret>` 예시 (공식 PHP SDK): https://github.com/manychat/manychat-api-php
- 발급 메뉴 경로 + Pro 플랜 전제 (공식 Help, 검색 인덱싱본): https://help.manychat.com/hc/en-us/articles/14959510331420
- Manychat 앱: https://app.manychat.com

## 미확인 / 주의
- **발급 메뉴 경로(Settings → API → Generate your API Key)와 Pro 플랜 전제는 공식 Help의 검색 인덱싱본으로 확인**했으며, 원문 페이지는 Cloudflare 봇 차단(403)으로 직접 열람하지 못했다. 메뉴 명칭/경로가 UI 개편으로 달라졌을 수 있으니, 화면에서 보이지 않으면 Manychat 앱 내 Settings 하위에서 "API" 항목을 찾는다.
- **발송 도달은 키 발급과 별개:** API가 200(`status:"success"`)을 반환해도 Meta/채널 정책(24시간 윈도우, message tag/OTN)에 따라 실제 도달은 보장되지 않는다. 윈도우 밖 발송에는 `message_tag`/`otn_topic_name` 사용.
- **429/레이트리밋 초과 응답 형식은 spec에 미정의**(Retry-After 명시 없음). 클라이언트는 지수 백오프 재시도로 가드한다.
