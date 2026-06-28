# 키 발급 가이드 — Oracle Fusion Cloud ERP (oracle_erp)

> 검증 출처: `_workspace/oracle_erp/02_validation.md` (검증일 2026-06-26, Oracle 공식 farfa/fapra 26A 문서 + 공식 IDCS/IAM OAuth 문서 기준).
> 핵심 전제: Oracle은 외부에서 "키 신청"하는 방식이 아니다. 자격증명은 **고객 자신의 Fusion 인스턴스(테넌트)의 IDCS/IAM 관리자**가 직접 발급한다. 공용/데모 키 없음.

---

## 연동 유형 / 인증 방식

- **연동 유형:** REST API (Oracle FSCM REST API). 공식 원격 MCP 커넥터는 없으며, 본 서버는 로컬 REST 래퍼로 동작한다.
  - 출처: https://docs.oracle.com/en/cloud/saas/financials/26a/farfa/index.html
- **베이스 URL:** `https://<server>.fa.<region>.oraclecloud.com` (고객 pod별 상이 → env 주입 필수, 하드코딩 금지)
  - 출처: https://docs.oracle.com/en/cloud/saas/financials/26a/farfa/Quick_Start.html
- **리소스 프리픽스(고정):** `/fscmRestApi/resources/11.13.18.05/`
- **인증 방식(권장):** OAuth 2.0 Bearer — IDCS/IAM에서 발급한 access token을 `Authorization: Bearer <token>`로 전송.
  - 서버-서버 기본 그랜트: **Client Credentials (2-legged)**.
  - 출처: https://docs.oracle.com/en/cloud/saas/sales/faaps/Configure_OAuth_Using_Oracle_IDCS_or_IAM.html
- **대안 인증:** Basic over SSL (`-u user:pass`, `Authorization: Basic`) — 동일 REST에 유효하나 운영은 OAuth 권장.
  - 출처: https://docs.oracle.com/en/cloud/saas/financials/26a/farfa/Quick_Start.html ("Multi Token Over SSL RESTful Service Policy")

---

## 발급 절차 (스텝)

> 작업 주체 = 고객 Fusion 인스턴스의 **IDCS/IAM 관리자**. 외부에서 Oracle에 직접 신청하는 단계는 없다.

1. **Fusion 인스턴스(pod) 호스트 확인.** 구독 중인 인스턴스의 base 호스트 `https://<server>.fa.<region>.oraclecloud.com`를 확인한다. (예: `https://servername.fa.us2.oraclecloud.com`)
2. **IDCS/IAM 콘솔 접속.** 해당 테넌트의 Identity Domain(IDCS/IAM) 관리 콘솔에 관리자로 로그인한다.
   - 토큰 엔드포인트·메타데이터는 IDCS 호스트 기준: 토큰 = `https://<IDCS_HOST>/oauth2/v1/token`, 메타 = `https://<IDCS_HOST>/.well-known/idcs-configuration`.
3. **Confidential Application 등록.** IDCS/IAM에서 Confidential App을 새로 생성하고, Fusion(FA) 리소스에 대한 접근을 구성한다. 생성 후 **Client ID**와 **Client Secret**을 확보한다.
   - 공식 문구: "Add a confidential application … Make note of Client ID and Client Secret."
   - 출처: https://docs.oracle.com/en/cloud/saas/sales/faaps/Configure_OAuth_Using_Oracle_IDCS_or_IAM.html
4. **정확한 스코프 문자열 확인.** 해당 IDCS의 Fusion 앱에 노출된 스코프를 조회한다 (아래 "스코프/권한" 참조).
5. **토큰 발급(검증).**
   ```
   POST https://<IDCS_HOST>/oauth2/v1/token
   Authorization: Basic base64(clientId:clientSecret)
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&scope=<FA_SCOPE>
   ```
   응답의 `access_token`을 모든 REST 호출에 `Authorization: Bearer <access_token>`로 사용.
6. **직무역할·데이터 보안 부여.** 통합에 사용할 계정에 대상 리소스별 Fusion 직무역할(job/duty roles)·데이터 보안을 부여한다 (아래 "전제조건" 참조).

---

## 전제조건

- **고객 Fusion 인스턴스 구독 필수.** base 호스트가 고객 pod별로 다르므로 공용 키 없음. 호스트·자격증명 전량 env 주입.
- **IDCS/IAM 관리자 권한 필요.** Confidential App 등록은 고객 테넌트 관리자 작업.
- **Fusion 직무역할·데이터 보안 부여 필요.** REST 인증 성공 ≠ 데이터 접근 성공. 통합 계정에 AP 인보이스 읽기/쓰기, GL 조회, 지급 조회 등 각 리소스별 권한을 관리자가 별도로 부여해야 한다. 권한 부족 시 200 빈 결과 또는 403 발생.
- **한국 특유 게이트 없음.** 사업자 심사·제휴 신청·발신번호 사전등록 등은 해당 없음 (검증됨).
  - 출처: https://docs.oracle.com/en/cloud/saas/sales/faaps/Configure_OAuth_Using_Oracle_IDCS_or_IAM.html

---

## 스코프 / 권한

- **OAuth 스코프 문자열:** 단독 `urn:opc:resource:consumer::all`이 아니라 **인스턴스 스코프와 함께** 사용:
  ```
  urn:opc:resource:fa:instanceid=<INSTANCE_ID> urn:opc:resource:consumer::all
  ```
  - Refresh 토큰이 필요하면 `offline_access` 추가.
  - **정확한 스코프 문자열은 고객 IDCS의 Fusion 앱에서 조회**해 env로 주입한다 (인스턴스마다 다름).
  - 출처: https://www.ateam-oracle.com/authenticate-and-work-with-fusion-v1-api-rest (보강), 공식 OAuth 문서 https://docs.oracle.com/en/cloud/saas/sales/faaps/Configure_OAuth_Using_Oracle_IDCS_or_IAM.html
- **그랜트 타입:** Client Credentials(2-legged, 기본), Client Credentials+JWT Assertion, Authorization Code(3-legged) 3종 지원. 본 서버 기본 범위는 Client Credentials.
- **데이터 권한:** 위 OAuth 스코프와 **별개로** Fusion 직무역할/데이터 보안이 동시 충족되어야 실제 데이터 접근 가능.

---

## 환경변수 매핑

| 변수명 | 값을 어디서 받나 | 필수 |
|--------|------------------|------|
| `ORACLE_FUSION_BASE_URL` | 구독 중인 Fusion 인스턴스 pod 호스트 `https://<server>.fa.<region>.oraclecloud.com` | 필수 |
| `ORACLE_IDCS_TOKEN_URL` | IDCS/IAM 호스트 + `/oauth2/v1/token` (예: `https://idcs-xxxx.identity.oraclecloud.com/oauth2/v1/token`) | OAuth 사용 시 필수 |
| `ORACLE_OAUTH_CLIENT_ID` | IDCS/IAM Confidential App 등록 시 발급된 Client ID | OAuth 사용 시 필수 |
| `ORACLE_OAUTH_CLIENT_SECRET` | IDCS/IAM Confidential App 등록 시 발급된 Client Secret | OAuth 사용 시 필수 |
| `ORACLE_OAUTH_SCOPE` | 고객 IDCS Fusion 앱의 스코프 (`urn:opc:resource:fa:instanceid=<INSTANCE_ID> urn:opc:resource:consumer::all`) | OAuth 사용 시 필수 |
| `ORACLE_FUSION_USERNAME` | (Basic 폴백) Fusion 사용자명 | OAuth 미설정 시 필수 |
| `ORACLE_FUSION_PASSWORD` | (Basic 폴백) Fusion 비밀번호 | OAuth 미설정 시 필수 |
| `ORACLE_ALLOW_DESTRUCTIVE` | 파괴적 도구(`cancel_payables_invoice` 등) 활성화(`true`) | 선택(기본 비활성) |

> 시크릿(Client Secret/Password)은 하드코딩·로그 노출 금지. `.mcp.json`은 `${VAR}` 확장 사용.

---

## 출처 (공식 URL)

- REST API 개요(farfa 26A): https://docs.oracle.com/en/cloud/saas/financials/26a/farfa/index.html
- Quick Start (base URL·인증·describe·페이지네이션): https://docs.oracle.com/en/cloud/saas/financials/26a/farfa/Quick_Start.html
- OAuth (IDCS/IAM Confidential App·토큰): https://docs.oracle.com/en/cloud/saas/sales/faaps/Configure_OAuth_Using_Oracle_IDCS_or_IAM.html
- 스코프 보강(인스턴스 스코프 사용법): https://www.ateam-oracle.com/authenticate-and-work-with-fusion-v1-api-rest
- Oracle MCP 현황(Fusion ERP용 외부 MCP 미존재): https://www.oracle.com/mcp/

---

## 미확인 / 주의

- **정확한 스코프 문자열의 `INSTANCE_ID` 값**은 고객 IDCS Fusion 앱에서만 확인 가능 — 본 가이드로는 확정 불가(테넌트별 상이). env로 주입.
- **레이트리밋 수치 미공개.** 공식 1차 출처 없음. 429 지수 백오프·동시성 제한으로 런타임 가드 위임.
- **GL 분개 "생성" REST 미지원.** `journalBatches`에 POST 없음 → 분개는 조회만. 입력은 journalImport/ERP Integration Service(파일·SOAP) 경유 필요(본 서버 범위 외).
- **AR 인보이스 PATCH는 3개 필드만**(`InvoiceStatus`/`PaymentTerms`/`TransactionDate`).
- **파괴적 액션**(`cancelInvoice` 등)은 기본 비활성. `ORACLE_ALLOW_DESTRUCTIVE=true` + 확인 게이트 필요.
- **Webhook(아웃바운드)·GraphQL·한국 전자세금계산서(국세청) REST 노출**: 표준 farfa/fapra 리소스 아님 → 미확인, 본 서버 범위 제외.
