# 키 발급 가이드 — flex (flex)

> flex.team(주식회사 플렉스, 한국 HR) Open API v2 연동용 인증 발급 가이드.
> 동명 타사(withflex.com / Flexmls / Twilio Flex)와 무관 — flex.team 공식 문서만 근거.
> 추측 없이 검증된 사실만 기재. 불명확한 항목은 "미확인 / 주의"에 명시.

## 연동 유형 / 인증 방식

- **연동 유형:** 로컬 MCP 서버(stdio) ↔ flex 문서화 REST Open API(Bearer). flex 공식 원격 MCP 커넥터는 없음.
- **인증 방식:** HTTP Bearer (JWT). 모든 데이터 호출에 `Authorization: Bearer <AccessToken>`.
- **토큰 흐름(런타임):** 발급물(Refresh Token **또는** Client ID+Secret)로 토큰 엔드포인트를 호출해 Access Token을 획득하고, 만료 시 갱신한다. 이 갱신은 MCP 서버 내부 인증 모듈이 자동 처리하며 별도 도구로 노출하지 않는다.
  - 토큰 엔드포인트: `POST https://openapi.flex.team/v2/auth/realms/open-api/protocol/openid-connect/token`, `Content-Type: application/x-www-form-urlencoded`.
  - **Refresh Token grant:** `grant_type=refresh_token` + `refresh_token=<RT>` (필요시 `client_id=open-api`). 응답의 `refresh_token`도 회전될 수 있어 갱신본 저장 권장.
  - **Client Credentials grant:** `grant_type=client_credentials` + `client_id=<발급ID>` + `client_secret=<발급Secret>`.

## 발급 절차 (스텝)

발급은 flex 제품(웹 콘솔) 내 **설정 화면 > Open API 설정 메뉴**에서 진행한다.

> **발급 페이지 직링크: 없음.** flex 콘솔은 고객사 워크스페이스별 인증 화면이라 발급 화면으로 바로 가는 공개 고정 URL이 공식 문서에 제공되지 않는다. 아래 메뉴 경로로 접근한다.

**메뉴 경로:** flex 웹 로그인 → **설정**(설정 화면 진입) → 화면 **우측의 "Open API 설정" 메뉴** 선택 → **"+ 추가하기"** 버튼

1. flex 제품에 로그인한 뒤 **설정 화면**으로 이동한다.
2. 설정 화면 **우측의 "Open API 설정" 메뉴**를 선택한다.
3. **"+ 추가하기"** 버튼을 클릭한다.
4. 인증 방식을 선택한다 — 두 가지 중 택일:
   - **Refresh Token 방식** (보안을 위해 주기적 갱신 가능):
     - Access Token 이름(식별용) 입력
     - Access Token 유효 기간 설정 (예: 10분)
   - **Client Credential 방식** (유효기간 동안 사용):
     - Client Credential 이름(식별용) 입력
     - Access Token 유효 기간 설정
     - Client Credential 유효 기간(전체 사용 기한) 설정 (예: 365일)
5. 발급을 완료하면 발급물(Refresh Token 방식은 Refresh Token/Access Token, Client Credential 방식은 Client ID/Client Secret/Access Token)이 **한 번만 표시**된다. 즉시 복사해 안전한 곳에 보관한다 (분실 시 재발급).
6. 보관한 발급물을 MCP 서버 환경변수에 주입한다(아래 "환경변수 매핑" 참조).

> 추가 운영: 발급된 인증정보는 같은 Open API 설정 메뉴에서 선택 후 삭제할 수 있다.

## 전제조건

외부 게이트(사업자 심사·제휴 승인 등)는 공식 문서상 **없음**. 단, 다음이 충족되지 않으면 메뉴 미노출 또는 빈 응답으로 동작하지 않는다:

- **(a) Open API 지원 요금제 구독** — Open API는 플랜 구독 고객사에만 노출.
- **(b) 사용자 권한** — 인사정보관리 권한(대부분 도구), 비용부서 권한(`get_user_cost_centers`).
- **(c) 코드 사전등록** — 사번·조직·직무 코드가 flex에 등록돼야 API에 노출됨(빈 배열의 가장 흔한 원인).
- 발신번호 사전등록 등 별도 절차: **해당 없음**(읽기 전용 HR API).

> 커스텀 필드 매핑이 필요하면 `integration@flex.team`로 별도 요청(공식 FAQ).

## 스코프 / 권한

- 토큰 응답에 `scope` 문자열은 존재하나, **공개된 scope 카탈로그·요청 파라미터는 없음**. 따라서 토큰 발급/호출 시 scope를 별도로 요청하지 않는다.
- 실제 접근 통제는 **flex 제품 권한**으로 결정된다: 인사정보관리 권한, 비용부서 권한.
- API는 **전부 GET 읽기 전용** — 외부발 근무/휴가 등록·변경·승인은 공식 미지원(쓰기 권한·도구 없음).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `FLEX_AUTH_MODE` | 사용할 방식 선택값: `refresh_token` 또는 `client_credentials` (위 발급 스텝 3에서 고른 방식) | 필수 |
| `FLEX_REFRESH_TOKEN` | Refresh Token 방식 발급 화면에 1회 표시된 Refresh Token | `FLEX_AUTH_MODE=refresh_token`일 때 필수 |
| `FLEX_CLIENT_ID` | Client Credential 방식 발급 화면의 Client ID | `FLEX_AUTH_MODE=client_credentials`일 때 필수 |
| `FLEX_CLIENT_SECRET` | Client Credential 방식 발급 화면에 1회 표시된 Client Secret | `FLEX_AUTH_MODE=client_credentials`일 때 필수 |
| `FLEX_BASE_URL` | 기본 `https://openapi.flex.team` (오버라이드용) | 선택(기본값 제공) |
| `FLEX_EXPOSE_SENSITIVE` | 민감 PII 정책 토글: `mask`(기본)·`off`·`full` | 선택(기본=마스킹) |

> 모든 시크릿은 환경변수로만 주입하고 절대 커밋하지 않는다. 발급물은 1회만 표시되므로 분실 시 재발급해야 한다.
> 인증정보는 콘솔당 **최대 5개**까지 추가 가능.

## 출처 (공식 URL)

- 토큰 발급 UI/절차(접근 경로 "설정 화면 > 우측 Open API 설정 메뉴 > + 추가하기"·방식 선택·최대 5개·1회 표시): https://developers.flex.team/reference/open-api-authentication-token-issuance.md
- Open API 설정 사용 가이드(관리자 콘솔 메뉴 안내, 직링크 없음·워크스페이스별 접근): https://guide.flex.team/1bae07ff-3cfa-419a-a64c-02a20cabbfeb
- 인증 방식(Bearer JWT)·토큰 엔드포인트·grant 형식: https://developers.flex.team/reference/authentication-token.md
- 빠른 시작(refresh→access 흐름): https://developers.flex.team/reference/open-api-quick-start.md
- 전제조건(플랜 구독·권한·코드 사전등록·읽기 전용·커스텀 필드 요청): https://developers.flex.team/reference/faq-limitation.md
- 검증본(내부): `_workspace/flex/02_validation.md`

## 미확인 / 주의

- **발급 메뉴의 정확한 권한/플랜 게이트 문구:** 발급 화면 문서 자체에는 "어떤 권한·플랜이 있어야 이 메뉴가 보이는지"가 명시돼 있지 않다. 전제조건(플랜·권한·코드 등록)은 별도 FAQ 근거이며, 발급 화면 노출 조건의 세부는 **미확인**. 고객사 도입 전 점검 권장.
- **에러 바디 표준 스키마:** 전 엔드포인트 OpenAPI가 200만 정의하고 4xx/5xx 표준 스키마를 문서화하지 않음 → 표준 에러 포맷 **미확인**. 비-2xx는 HTTP status + 원문 바디로 처리.
- **공식 SDK:** 공식 npm/pip SDK 제공 명시 없음(**미확인**). 순수 fetch/HTTP로 구현 가능하므로 발급/사용에 지장 없음.
- **레이트리밋:** 1 request / second 직렬 제한. 발급과 무관하나 호출 설계 시 주의.
- **타임존:** `work-clock-events` 등 타각 응답 시각은 UTC(Z) — KST는 +9h 변환.
