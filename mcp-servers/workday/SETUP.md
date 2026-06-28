# 키 발급 가이드 — Workday (workday)

> 출처: `_workspace/workday/02_validation.md`(검증일 2026-06-26, 적대적 검증) 및 `mcp-servers/workday/README.md`. 검증된 출처 있는 사실만 기재. 추측 항목은 "미확인"으로 명시.

## 연동 유형 / 인증 방식

- **연동 유형**: REST API (JSON). Workday 호스팅 공식 원격 MCP 커넥터 없음 + Anthropic Connectors Directory 미등재 → 로컬 MCP 서버 빌드.
- **인증 방식**: OAuth 2.0 **Client Credentials grant** (서버-투-서버, ISU 기반).
  - 헤드리스 MCP 서버에 적합. 사용자 로그인/redirect URI 불필요.
  - **client_credentials는 refresh token을 발급하지 않음** → access token(수명 ~3600s/60분) 만료 시 재요청.
  - *Authorization Code / Non-Expiring Refresh Token 흐름은 사람-동의형이라 MCP 서버용 미채택(보류).*
- **토큰 흐름**:
  1. `POST {WORKDAY_TOKEN_ENDPOINT}` (테넌트 발급 URL, 패턴 `https://{host}/ccx/oauth2/{tenant}/token`)
  2. 인증: HTTP Basic `Authorization: Basic base64(CLIENT_ID:CLIENT_SECRET)` 또는 바디 `client_id`/`client_secret`
  3. 바디: `grant_type=client_credentials` (필요 시 `scope=...`)
  4. 응답 `access_token`을 만료시각과 함께 캐시 → 만료 임박 시 재요청
  5. 이후 모든 REST 호출에 `Authorization: Bearer {access_token}`

## 발급 절차 (스텝)

> Workday 테넌트 관리자가 수행. 아래 1~4단계가 선행되지 않으면 토큰을 받아도 **모든 호출이 403**이다.

1. **Integration System User(ISU) 생성** — "Do Not Allow UI Sessions" 권장.
2. **ISU를 Integration System Security Group(ISSG)에 멤버로 추가.**
3. **대상 도메인에 Domain Security Policy 권한(View/Modify) 부여** — 예: Worker Data, Absence 등 사용할 도메인.
4. **Register API Client 등록** — grant 유형 = **Client Credentials**, 필요한 **Scope** 선택.
5. **"View API Clients" 리포트에서 발급값 확인** — **Token Endpoint** 전체 URL, **REST API Endpoint** 전체 베이스 URL, **Client ID**, **Client Secret**.
   - 이 값들을 환경변수로 **그대로 주입**(서버가 호스트를 조립하면 테넌트별 환경 차이로 깨질 수 있으므로 발급값 우선).

> 콘솔 위치: Workday 테넌트 내 "Register API Client" 태스크로 등록, "View API Clients" 리포트로 발급값 확인. (Token Endpoint·REST API Endpoint·Client ID/Secret은 테넌트마다 호스트·환경(prod/impl)이 다름.)

## 전제조건

- **Workday 테넌트 보유** + **관리자가 위 발급 절차 1~4단계 수행** 필수. 무료 공개 API 없음(엔터프라이즈 게이팅).
- **ISU + ISSG + Domain Security Policy** 선행 설정이 인증의 필수 전제(누락 시 403).
- 데이터 민감도(PII·급여) → 토큰/시크릿 안전 보관, 최소권한 Scope, 로깅 마스킹.
- 사업자 심사·발신번호 사전등록 등 별도 심사 절차: **해당 없음**(테넌트 관리자 설정으로 충족).

## 스코프 / 권한

- **Scope**: 기능 도메인 단위(예: Worker Data, Absence)로 API Client 등록 시 선택.
- 실제 권한은 **ISU에 부여한 Domain Security Policy**가 결정(Scope와 도메인 권한 모두 필요).
- **정확한 스코프 문자열 목록은 미확인** → 고객 테넌트의 API Client 설정값을 따른다(서버가 강제하지 않음). 환경변수 `WORKDAY_SCOPE`로 선택 전달, 미설정 시 API Client 기본 scope 사용.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `WORKDAY_HOST` | 테넌트 호스트(예: `wd10.myworkday.com` 또는 `wd2-impl-services1.workday.com`). ENDPOINT 미설정 시 폴백 조립용. | 필수(또는 두 ENDPOINT 모두 설정 시 대체) |
| `WORKDAY_TENANT` | 테넌트명(경로의 `{tenant}`). | 필수 |
| `WORKDAY_TOKEN_ENDPOINT` | "View API Clients"의 **Token Endpoint** 전체 URL. 미설정 시 `https://{WORKDAY_HOST}/ccx/oauth2/{WORKDAY_TENANT}/token` 폴백. | 권장(또는 HOST/TENANT 폴백) |
| `WORKDAY_REST_API_ENDPOINT` | "View API Clients"의 **REST API Endpoint** 전체 베이스 URL(`…/ccx/api`까지). 미설정 시 `https://{WORKDAY_HOST}/ccx/api` 폴백. | 권장(또는 폴백) |
| `WORKDAY_CLIENT_ID` | API Client의 **Client ID**. | 필수 |
| `WORKDAY_CLIENT_SECRET` | API Client의 **Client Secret**. | 필수 |
| `WORKDAY_SCOPE` | 토큰 요청 scope 문자열. 미설정 시 API Client 기본 scope. | 선택 |
| `WORKDAY_PATH_ABSENCE_BALANCES` | absence 잔액 path 오버라이드(`{worker_id}` 치환). 실측 path가 다를 때만. | 선택(가드) |
| `WORKDAY_PATH_ELIGIBLE_ABSENCE_TYPES` | 신청 가능 absence 유형 path 오버라이드. | 선택(가드) |
| `WORKDAY_PATH_TIME_OFF_DETAILS` | 타임오프 상세 path 오버라이드. | 선택(가드) |
| `WORKDAY_PATH_REQUEST_TIME_OFF` | 휴가 신청 path 오버라이드. | 선택(가드) |

## 출처 (공식 URL)

- Workday REST API 디렉터리(공식, 로그인 벽): https://community.workday.com/sites/default/files/file-hosting/restapi/index.html
- OAuth 2.0 인증 / API Client 등록·발급값 확인 절차: https://www.apideck.com/blog/how-to-get-your-workday-api-keys
- Workday API 통합(인증·페이지네이션·레이트리밋 개요): https://www.getknit.dev/blog/workday-api-integration-in-depth
- OAuth 2.0 authentication(절차 참고): https://documentation.sailpoint.com/connectors/workday/help/integrating_workday/oauth_2.0_authentication.html
- 베이스 URL·리소스 path 교차검증(Workday 공식 OpenAPI 자동생성 Ballerina 커넥터, 중립 미러): https://central.ballerina.io/ballerinax/workday.common/latest · https://central.ballerina.io/ballerinax/workday.absencemanagement/latest
- Anthropic Connectors Directory FAQ(미등재 확인): https://support.claude.com/en/articles/11596036-anthropic-connectors-directory-faq

## 미확인 / 주의

- **정확한 Scope 문자열 목록 미확인** — 테넌트 API Client 설정값을 따른다.
- **Token Endpoint / REST API Endpoint 호스트는 테넌트 발급값** — 서버가 문자열로 조립하지 말고 "View API Clients" 값을 그대로 주입(테넌트별 호스트·환경 차이로 조립 시 깨짐).
- **페이지네이션 최대 limit 수치 미확인** — 보수적으로 limit=100 권장(기본 20).
- **레이트리밋 공식 수치 미공개** — "10 req/s"는 파트너 경험치(비공식). 확정 규약 = HTTP 429 + `Retry-After` 헤더 → 대기 후 지수 백오프.
- **일부 리터럴 path 세부 미확인(가드)** — absence balances·eligibleAbsenceTypes·timeOffDetails·requestTimeOff의 정확 세그먼트/바디 스키마는 함수명 수준까지 확정. 테넌트 실측 후 다르면 `WORKDAY_PATH_*` 환경변수로 오버라이드(코드 수정 불필요).
- **공식 1차 클라이언트 SDK(npm/pip) 미확인** — 제3자 래퍼만 존재 → 직접 fetch 호출.
- **에러 JSON 필드명 미확인** — 401/429 동작은 확인, 정확 에러 바디 필드명은 런타임 방어 처리.
