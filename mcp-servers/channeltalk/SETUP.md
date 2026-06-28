# 키 발급 가이드 — 채널톡 (channeltalk)

> 근거: `_workspace/channeltalk/02_validation.md` (검증일 2026-06-26) 및 `mcp-servers/channeltalk/README.md`.
> 추측 없이 출처 URL 있는 사실만 기재. 발급 디테일 중 공식 문서로 확인 불가한 항목은 "미확인"으로 표시.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 빌드 MCP 서버 (공식 원격 MCP 커넥터 없음 — route=build).
- **인증 방식**: HTTP 헤더 인증 (OAuth 아님). 모든 요청에 두 헤더를 포함한다.
  - `x-access-key: <ACCESS_KEY>`
  - `x-access-secret: <ACCESS_SECRET>`
- **토큰 흐름**: 토큰 교환·갱신 없음. 데스크에서 발급한 정적 키쌍(Access Key / Access Secret)을 그대로 사용한다.
- 출처: https://developers.channel.io/docs/authentication-2

## 발급 절차 (스텝)

> **발급 페이지 직링크: 없음(미확인).** 채널톡 데스크의 API 키 관리 화면으로 바로 가는 공개 직링크 URL은 공식 문서·도움말 어디에도 명시되어 있지 않다(데스크 URL은 워크스페이스별로 다름). 아래 **웹 데스크에서 메뉴 경로로 진입**한다.
> 데스크 진입점: 웹 데스크(`https://desk.channel.io` 로그인) → **좌측 하단의 채널/스페이스 설정** 아이콘.

채널톡 데스크 UI에서는 동일 기능에 두 갈래 경로가 있다(둘 중 하나로 진입). 현행 한국어 데스크 기준:

**경로 A (스페이스 설정 기준):**
1. 웹 데스크에 로그인한다(`https://desk.channel.io`).
2. 좌측 하단 **스페이스 설정**으로 들어간다.
3. **연동** 으로 이동한다.
4. **API 인증 키 관리** 탭을 연다.
5. **새 인증 키 만들기** 버튼을 클릭한다(보통 우측 상단).
6. 키 이름을 입력하고 **확인**을 누른다.
7. 생성된 **Access Key**(`x-access-key`) 와 **Access Secret**(`x-access-secret`) 값을 확인하여 즉시 보관한다.
   - **주의: Access Secret 은 생성 직후에만 표시되고 다시 조회 불가** → 그 자리에서 복사·저장할 것.

**경로 B (채널 설정 기준):**
1. 웹 데스크 좌측 하단 **채널 설정**으로 들어간다.
2. **보안 및 개발** 으로 이동한다.
3. **API** 메뉴를 연다.
4. **새 인증 키 만들기** 를 클릭하고 이름 입력 후 **확인**.
5. (이하 동일) **Access Key / Access Secret** 을 즉시 보관.

> 영문 개발자 문서 표기로는 위 경로가 **Settings > API Key management > Create new credential** 에 대응한다.

- 데스크에서 자가 발급이며, 공개 신청/승인 절차 문구는 공식 문서에 없음(= 별도 심사 불필요로 보임).
- 출처(영문 발급 경로): https://developers.channel.io/docs/authentication-2
- 출처(한국어 데스크 메뉴 경로 — 스페이스 설정 > 연동 > API 인증 키 관리): https://docs.channel.io/help/ko/articles/%EC%8A%A4%ED%8E%98%EC%9D%B4%EC%8A%A4-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0-8c81b1ba

## 전제조건

- **사업자/제휴 심사**: 불필요로 보임 (공식 문서에 신청·승인 절차 문구 없음).
- **발신번호 사전등록 / 실명인증**: 불필요로 보임 — 채널톡 상담 메시지는 자사 메신저 채널 내 발신이므로 통신사 발신번호 등록 대상이 아님.
- **요금제 게이트 (미확인·중요)**: API Key 관리 메뉴/Open API 사용이 특정 유료 요금제에서만 열리는지 공식 문서로 확인 불가. 무료/하위 플랜에서는 발급/사용이 막힐 수 있음 → 발급 단계에서 실측 확인 필요.
- **키 생성 역할 (미확인)**: 소유자/관리자만 발급 가능한지 미확인. 발급이 안 되면 데스크 계정 권한을 확인할 것.
- 참고: 채널톡을 통한 외부 카카오 알림톡 발신은 본 커넥터 범위 밖(별도 제약 가능).

## 스코프 / 권한

- **API Key 세분 스코프 없음 (미확인)**: 읽기/쓰기 분리 등 세분 스코프 체계는 공식 문서에서 미발견. 단일 자격증명(Access Key + Secret)이 모든 도구 권한을 갖는다고 가정한다.
- 앱(App) 토큰 방식에서만 Channel/User/Manager 권한 선택이 존재하나, 본 커넥터는 App 토큰이 아닌 **API Key 단일 자격증명** 방식을 사용한다.
  - 출처(앱 인증 참고): https://developers.channel.io/reference/app-authentication-kr
- 일부 쓰기 도구가 401/403 을 반환하면 키 권한 또는 요금제 게이트 문제로 간주한다.

## 환경변수 매핑

| 변수명 | 값을 어디서 | 필수 |
|--------|-------------|------|
| `CHANNEL_ACCESS_KEY` | 데스크 > Settings > API Key management 에서 생성한 **Access Key** (`x-access-key` 헤더 값) | 예 |
| `CHANNEL_ACCESS_SECRET` | 동일 화면에서 생성한 **Access Secret** (`x-access-secret` 헤더 값) | 예 |
| `CHANNEL_API_BASE_URL` | (선택) 베이스 URL 오버라이드. 기본값 `https://api.channel.io` | 아니오 |

- 베이스 URL: 메인 `https://api.channel.io/open/v5` (managers 도구만 `https://api.channel.io/open/v4`).
- 시크릿 실값을 `.mcp.json` 등 파일에 직접 쓰지 말고 환경변수 확장(`${VAR}`) 또는 `--env` 플래그로 주입한다.

## 출처 (공식 URL)

- 인증·키 발급 경로(영문): https://developers.channel.io/docs/authentication-2
- 한국어 데스크 메뉴 경로(스페이스 설정 > 연동 > API 인증 키 관리): https://docs.channel.io/help/ko/articles/%EC%8A%A4%ED%8E%98%EC%9D%B4%EC%8A%A4-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0-8c81b1ba
- 레이트리밋: https://developers.channel.io/docs/rate-limiting
- 앱 인증(권한 참고): https://developers.channel.io/reference/app-authentication-kr

> **발급 페이지 직링크 미확인**: API 키 관리 화면으로 직접 가는 공개 URL은 공식 문서/도움말에 명시 없음. 진입은 웹 데스크(`https://desk.channel.io`) 로그인 후 위 메뉴 경로로만 가능.

## 미확인 / 주의

- **요금제 게이트 (미확인)**: API Key 사용이 특정 유료 요금제에서만 열리는지 공식 문서로 확인 불가. 첫 사용 전 실제 키로 `list_user_chats` 1회 호출하여 게이트 통과 여부를 스모크 테스트할 것.
- **발급 역할 제한 (미확인)**: 소유자/관리자만 발급 가능한지 미확인.
- **세분 스코프 (미확인)**: API Key 단위 읽기/쓰기 스코프 분리 체계 미발견 → 단일 자격증명이 전체 권한을 갖는다고 가정.
- **에러 응답 바디 스키마 (미확인)**: 429 외 표준 HTTP 상태코드 사용은 확인되나, 에러 JSON의 코드/메시지 필드 구조는 공식 문서에서 미발견 → 구현은 HTTP status 기반 처리, 바디는 필드명 가정 없이 패스스루.
- **인증 예시 버전 주의**: 공식 인증 문서의 예시 curl 은 v3 경로(`/open/v3/channel`)지만 실제 호출은 v5(managers만 v4)를 사용한다. 발급한 키 자체는 버전 무관하게 동일 키쌍을 그대로 쓴다.
