# 키 발급 가이드 — Trello (trello)

> 출처: `_workspace/trello/02_validation.md`(검증일 2026-06-26) · `mcp-servers/trello/README.md`
> 원칙: 아래는 공식 문서에 출처가 있는 사실만 적었다. 미확인 항목은 "미확인 / 주의"에 분리.

## 연동 유형 / 인증 방식

- **연동 유형:** Trello **REST API v1** 직접 호출. 베이스 URL `https://api.trello.com/1/`.
- **인증 방식:** **API Key + Token** (Trello 1.0a 스타일 토큰 인증).
  - 서버가 모든 요청에 query string `?key={KEY}&token={TOKEN}` 을 자동 부착한다. 도구 입력에는 인증 파라미터가 없다.
  - **OAuth 2.0 미사용.** `GET /search`·`GET /search/members` 가 OAuth2/Forge 앱 호출을 명시적으로 거부하므로(검색 도구 보존을 위해) API Key + Token 으로 확정했다.
- 출처:
  - https://developer.atlassian.com/cloud/trello/guides/rest-api/authorization/
  - https://developer.atlassian.com/cloud/trello/guides/rest-api/api-introduction/

## 발급 절차 (스텝)

> API Key 발급 전에 **Power-Up 생성이 선행 조건**이다(단순 발급 버튼이 아님).

1. **Power-Up 생성** — https://trello.com/power-ups/admin 에 접속해 새 Power-Up 을 만든다.
2. **API Key 발급** — 생성한 Power-Up 의 "API Key" 탭에서 **Generate** 클릭 → 발급된 값을 `TRELLO_API_KEY` 로 사용.
3. **Token 발급** — 같은 화면의 Token 링크(authorize 플로우, `/1/authorize` 라우트)로 인증한다.
   - scope: `read,write`
   - expiration: 운영 정책상 `30days`(또는 `never`) 권장 — 공식 지정 가능 값은 `1hour / 1day / 30days / never`.
   - 발급된 토큰을 `TRELLO_TOKEN` 으로 사용.
- 출처:
  - https://developer.atlassian.com/cloud/trello/guides/rest-api/api-introduction/ ("you first need to have created a Trello Power-Up")
  - https://developer.atlassian.com/cloud/trello/guides/rest-api/authorization/

## 전제조건

- **사업자/제휴 심사:** 없음. 계정 + Power-Up 생성만으로 즉시 발급(글로벌 SaaS).
  - 출처: https://developer.atlassian.com/cloud/trello/guides/rest-api/api-introduction/
- **플랜 요건:** 별도 유료 플랜 요건은 검증본에 명시되지 않음(미확인 — 아래 참조).
- **발신번호 사전등록 / 공동인증서 등 한국 특화 제약:** 없음(글로벌 SaaS, 사업자·발신번호·공동인증서 무관).
- **선행 작업:** Power-Up 생성(위 1단계)이 API Key 발급의 전제조건.

## 스코프 / 권한

- **필수 스코프:** `read` + `write` (카드/리스트/보드 생성·수정 포함).
- **불필요 스코프:** `account` (멤버 이메일 읽기 등) — 본 커넥터는 사용하지 않으므로 부여 불필요.
- 공식 scope 정의(verbatim): `account` = "read member email, writing of member info, marking notifications read".
- 출처: https://developer.atlassian.com/cloud/trello/guides/rest-api/authorization/

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `TRELLO_API_KEY` | Power-Up 의 "API Key" 탭에서 **Generate** (1~2단계) | 필수 |
| `TRELLO_TOKEN` | API Key 화면의 Token 링크로 인증 후 발급. scope `read,write`. **계정 전체 접근 권한 — 비밀로 취급** | 필수 |
| `TRELLO_BASE_URL` | (선택) 베이스 URL 오버라이드. 기본값 `https://api.trello.com/1` | 아니오 |

> 시크릿은 코드/설정 파일에 평문으로 쓰지 말고 환경변수로 주입한다. `.mcp.json` 에서는 `${TRELLO_API_KEY}` / `${TRELLO_TOKEN}` 환경변수 확장을 사용한다.

## 출처 (공식 URL)

- 인증 가이드: https://developer.atlassian.com/cloud/trello/guides/rest-api/authorization/
- API 소개 / 베이스 URL / Power-Up 선행: https://developer.atlassian.com/cloud/trello/guides/rest-api/api-introduction/
- 레이트리밋: https://developer.atlassian.com/cloud/trello/guides/rest-api/rate-limits/
- REST 레퍼런스: https://developer.atlassian.com/cloud/trello/rest/
- 검색 API(OAuth2 불가 명시): https://developer.atlassian.com/cloud/trello/rest/api-group-search/
- 발급 콘솔(Power-Up 관리): https://trello.com/power-ups/admin

## 미확인 / 주의

- **보안 — Token = 계정 전체 접근.** `TRELLO_TOKEN` 이 유출되면 해당 계정의 **모든 보드**에 접근 가능하다. env/secret 로만 보관하고 로그에 노출하지 않는다. expiration 을 `never` 대신 기간제(예: `30days`)로 두는 것을 권장.
- **플랜 요건 미확인.** 발급에 특정 유료 플랜이 필요한지 검증본에 명시되지 않음(검증본은 "계정 + Power-Up 만으로 즉시 발급"이라고만 함).
- **레이트리밋(발급 후 운영 주의).** 토큰 100req/10s, 키 300req/10s. 또한 `/search`·`/membersSearch`·`/members` 는 별도 **100req/900s 특수제한** 적용(단 `/members/me/boards` 등 nested 리소스는 제외). 출처: https://developer.atlassian.com/cloud/trello/guides/rest-api/rate-limits/
- **포워드 리스크 1 — 공식 Trello MCP 임박.** Atlassian 이 "곧 출시, 원클릭 연결"을 공지(커뮤니티, 2026-06)했으나 출시일·원격 URL·정식 문서는 미공개. 출시되면 본 로컬 빌드는 connect 전환 재검토 대상. 확인처: claude.ai/directory, https://www.atlassian.com/platform/remote-mcp-server
- **포워드 리스크 2 — OAuth 2.0(3LO) 마이그레이션 진행 중.** RFC-89(2025-03) 단계로 강제 폐기일 미공표. 현행 API Key+Token 은 authorization 가이드에 유효. 단 OAuth2 강제 전환 시 `search_trello`/`search_members`(검색 도구)가 깨지므로 영향 재평가 필요. 출처: https://developer.atlassian.com/cloud/trello/changelog/
- **공식 단일 에러코드 레퍼런스 미확인.** 4xx 상세 에러코드 표 페이지가 공식에 없어, 구현은 HTTP status + 응답 본문 `message` 문자열로 처리.
