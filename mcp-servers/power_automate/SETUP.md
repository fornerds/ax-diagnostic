# 키 발급 가이드 — Power Automate (power_automate)

> 검증 출처: `_workspace/power_automate/02_validation.md` (검증일 2026-06-26) + 서버 `README.md`.
> 모든 사실은 공식 Microsoft Learn 문서에 근거한다. 추측 없음. 불명확한 항목은 "미확인"으로 표기.

## 연동 유형 / 인증 방식
- 연동 유형: **로컬 stdio MCP 서버(build)**. 공식 원격 OAuth 커넥터는 존재하지 않는다.
  출처: https://claude.com/blog/connectors-directory
- 인증: **Microsoft Entra ID OAuth2**. 발급받는 것은 단일 API "키"가 아니라 **Entra 앱(서비스 주체)의 클라이언트 ID/시크릿**이다.
- **두 개의 독립 인증 컨텍스트**(리소스별 토큰 분리):
  1. **Power Platform API** (`api.powerplatform.com`) — `list_flow_runs`, `list_flow_actions` 도구.
     scope `https://api.powerplatform.com/.default`.
  2. **Dataverse Web API** (`https://<org>.<region>.dynamics.com`) — 나머지 8개 도구.
     scope `{DATAVERSE_ORG_URL}/.default`.
- 토큰 흐름: **client_credentials**(서비스 주체, 무인 자동화 권장) 또는 사용자 위임.
- 토큰 엔드포인트: `POST https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token`
  출처: https://learn.microsoft.com/en-us/power-platform/admin/programmability-authentication-v2

## 발급 절차 (스텝)
> 발급 위치 = **Azure Portal의 Microsoft Entra ID(앱 등록)**. 아래 순서대로.

1. **Entra 앱 등록**: Azure Portal → Microsoft Entra ID → App registrations → New registration. 등록 후 **Application (client) ID** 와 **Directory (tenant) ID** 를 기록한다.
2. **클라이언트 시크릿 발급**: 해당 앱 → Certificates & secrets → New client secret. 생성된 **시크릿 값**을 즉시 복사(이후 재확인 불가).
3. **API 권한 추가**: 앱 → API permissions → Add a permission → **"Power Platform API"** 등록(GUID `8578e004-a5c6-46e7-913e-12f58912df43`)의 delegated 권한을 추가.
4. **관리자 동의**: API permissions에서 **Grant admin consent** 수행(테넌트 관리자 권한 필요할 수 있음).
5. **대상 환경에 서비스 주체 RBAC 역할 할당**: 대상 Power Platform 환경에 이 서비스 주체를 **Reader(읽기)** 또는 **Contributor(쓰기)** 역할로 할당.
   - **[치명]** Power Platform API는 application permission을 쓰지 않는다(delegated 전용). 서비스 주체는 **RBAC 역할**로만 권한을 받는다. 토큰이 발급돼도 RBAC가 없으면 **403**.
   - 출처: https://learn.microsoft.com/en-us/power-platform/admin/programmability-authentication-v2 (Tutorial: Assign RBAC roles to service principals)
6. **Dataverse 접근 확인**: Dataverse 도구 사용 시, 같은 서비스 주체가 해당 Dataverse 환경에 접근 권한을 가져야 한다. 조직 URL(`https://<org>.<region>.dynamics.com`)은 Power Apps의 **"View developer resources"** 에서 확인.
   출처: https://learn.microsoft.com/en-us/power-automate/manage-flows-with-code
7. **환경변수 채우기**: 아래 매핑 표대로 `.env`(또는 MCP 설정 env)에 값을 채운다.

## 전제조건
- **플랜/사업자 심사/실명인증/발신번호 사전등록: 없음.** 글로벌 SaaS로 한국식 발급 심사는 해당 없다.
  출처: 02_validation.md 항목별 검증("한국식 사업자/제휴 심사·실명인증: 해당 없음", 검증됨).
- 단, 다음이 **사실상 게이트**다:
  - **테넌트 관리자 협조**: Grant admin consent 미수행 시 "needs admin approval" 에러.
  - **서비스 주체 RBAC 역할 할당**(최소 Reader, 쓰기엔 Contributor). 누락 시 403.
  - **대상 환경에 Microsoft Dataverse 데이터베이스 존재**. 없으면 flowRuns/flowActions·Dataverse 호출 모두 404/실패.

## 스코프 / 권한
- Power Platform API 토큰: `scope=https://api.powerplatform.com/.default`
- Dataverse Web API 토큰: `scope={DATAVERSE_ORG_URL}/.default`
- 권한 모델: **delegated 권한만 존재**. 서비스 주체 권한은 **환경 RBAC 역할(Reader/Contributor)** 로 부여(application permission 사용 금지).
- 추가 API 권한: "Power Platform API" (GUID `8578e004-a5c6-46e7-913e-12f58912df43`).
- 출처: https://learn.microsoft.com/en-us/power-platform/admin/programmability-authentication-v2

## 환경변수 매핑
| 변수명 | 값을 어디서 | 필수 |
|---|---|---|
| `POWER_PLATFORM_TENANT_ID` | Entra 앱 등록의 **Directory (tenant) ID** (App registration 개요) | 예 |
| `POWER_PLATFORM_CLIENT_ID` | Entra 앱 등록의 **Application (client) ID** (App registration 개요) | 예 |
| `POWER_PLATFORM_CLIENT_SECRET` | 앱의 **Certificates & secrets → New client secret** 의 값(생성 직후 1회 복사) | 예(서비스 주체 사용 시) |
| `DATAVERSE_ORG_URL` | 조직 Dataverse 베이스 URL `https://<org>.<region>.dynamics.com` — Power Apps "View developer resources"에서 확인(리전별 상이, 하드코딩 금지) | 예(Dataverse 도구 사용 시) |
| `POWER_PLATFORM_ENVIRONMENT_ID` | 기본 환경 ID(도구 파라미터 미지정 시 폴백) — Power Platform Admin Center의 환경 상세 | 아니오 |

> 시크릿은 전부 env로만. 코드/리포에 실값 하드코딩 금지. `.env.example`에는 키 이름만.

## 출처 (공식 URL)
- 인증·스코프·앱 등록·RBAC: https://learn.microsoft.com/en-us/power-platform/admin/programmability-authentication-v2
- 코드로 플로우 관리(Dataverse 경로·조직 URL): https://learn.microsoft.com/en-us/power-automate/manage-flows-with-code
- Dataverse OAuth 인증: https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/reference/retrievesharedprincipalsandaccess 외 — https://learn.microsoft.com/en-us/power-apps/developer/data-platform/authenticate-oauth
- Power Platform REST(Power Automate, api-version 2024-10-01): https://learn.microsoft.com/en-us/rest/api/power-platform/powerautomate/flow-runs/list-flow-runs
- 원격 커넥터 부재 확인: https://claude.com/blog/connectors-directory

## 미확인 / 주의
- **레이트리밋 수치 미확인**: Power Platform API·Dataverse 공통 throttling 수치를 공식 문서로 확정하지 못함. 구현은 429의 `Retry-After`를 존중하는 백오프로 방어(수치 가정 금지).
- **클라이언트 시크릿 만료**: Entra 시크릿은 만료 기간이 있다(생성 시 지정). 만료 시 재발급 필요 — 구체 정책은 조직 설정에 따름(미확인).
- **사용자 위임 흐름 세부**: 본 가이드는 client_credentials(서비스 주체)를 기준으로 한다. 사용자 위임(password/auth-code) 흐름의 단계별 발급 디테일은 위 인증 문서 참조(본 서버 기본 경로 아님).
- **범위 제약(주의)**: 목록/생성/수정/삭제 도구는 **Solutions에 포함된 플로우만** 대상. "My Flows"(솔루션 외 개인 플로우)는 코드 관리 공식 미지원.
- **`api.flow.microsoft.com` 금지**: 공식 비지원. 사용하지 않는다.
