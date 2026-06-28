# 키 발급 가이드 — SAP SuccessFactors (sap_sf)

> 출처: `_workspace/sap_sf/02_validation.md` (Validated Spec, 검증일 2026-06-26) + `README.md`. 추측 없이 출처 있는 사실만 기재. 발급/권한 작업은 **고객사 SuccessFactors 테넌트 관리자만** 수행 가능하다.

## 연동 유형 / 인증 방식

- **연동 유형:** REST — OData V2 (`/odata/v2`). 베이스 호스트는 테넌트 DC별 `https://api<DC>.<domain>`.
- **인증:** OAuth 2.0 **SAML 2.0 Bearer Assertion** grant (`grant_type=urn:ietf:params:oauth:grant-type:saml2-bearer`). 비대화형 서버-투-서버.
  - MCP 서버가 X.509 **개인키(PEM)**로 SAML 2.0 assertion을 **오프라인 자체 서명**해 `POST {SF_API_HOST}/oauth/token`으로 교환한다.
  - **`/oauth/idp`(assertion 생성 API)는 공식 폐기**(KBA 3239495, 2H 2022). 사용 금지.
  - **(레거시) HTTP Basic Auth는 2026-11-20 삭제 예정** → 신규는 OAuth만.

## 발급 절차 (스텝)

모두 **Admin Center**(테넌트 관리자 권한)에서 수행한다.

1. **Admin Center 접속** — `Admin Center` > 검색창에 **`Manage OAuth 2.0 Client Applications`** 입력 후 진입.
2. **클라이언트 애플리케이션 등록** — `Register Client Application` 클릭. Application Name, (필요 시) Application URL 등 입력.
3. **X.509 인증서 생성** — `Generate X.509 Certificate`로 인증서를 생성하고 `certificate.pem`(공개키 + **개인키** 포함)을 다운로드한다.
4. **API Key 확인** — 등록 완료 후 발급된 **API Key**(= OAuth `client_id`)를 기록한다. → `SF_CLIENT_ID`.
5. **개인키 추출** — 다운로드된 `certificate.pem`에서 **개인키(PEM)** 부분을 추출한다. → `SF_PRIVATE_KEY`(PEM 본문) 또는 `SF_PRIVATE_KEY_PATH`(파일 경로).
6. **기술계정 user_id 결정** — assertion `NameID` 및 토큰 `user_id`로 쓸 기술계정 사용자ID(예: `sfadmin`)를 확인한다. → `SF_USER_ID`.
7. **company_id / DC 호스트 확인** — 테넌트(회사) 식별자(`SF_COMPANY_ID`)와 테넌트가 속한 DC의 `apiXX` 호스트 풀 origin(`SF_API_HOST`, 예 `https://api4.successfactors.com`)을 확인한다.
8. **RBP 권한 부여** — 기술계정에 대상 엔티티/필드 접근 권한을 RBP(Role-Based Permissions)로 부여한다(아래 "스코프/권한" 참조).
9. **(필요 시) IP allowlist 등록** — 테넌트가 "API login exceptions"/IP 제한을 켰다면 서버 IP를 등록한다.

## 전제조건

- **테넌트 관리자 협조 (절대 전제):** OAuth 클라이언트 등록·X.509 키 발급·RBP 권한 부여는 고객사 SuccessFactors 관리자만 가능. 우리가 임의로 키를 받을 수 없다.
- **DC 호스트 + company_id 정확 입력:** 테넌트별 `apiXX` 호스트가 다르며 틀리면 전 호출 실패.
- **EC(Employee Central) 라이선스:** `PerPerson`/`EmpJob`/`EmpEmployment` 엔티티는 EC 보유 테넌트에서만 노출. `User` 엔티티는 EC 없이도 사용 가능(기본 진입점).
- **(조건부) IP allowlist:** 테넌트가 IP 제한을 켠 경우 서버 IP 등록 필요.
- **한국 특화 게이트(발신번호 사전등록·공동인증서·사업자 제휴심사 등):** **없음** (글로벌 SaaS, 해당 규제 비적용). 실질 게이트는 위의 "관리자 협조 + 정확한 DC 호스트"다.

## 스코프 / 권한

- **OAuth 토큰 스코프 개념 아님.** 대상 엔티티/필드 접근은 토큰이 아니라 **RBP(Role-Based Permissions)**로 통제된다.
- 토큰이 발급돼도 RBP 미부여 시 응답에서 필드가 **조용히 누락**되거나 403이 난다.
- **기술계정 최소권한 세트(엔티티/필드 ↔ RBP 항목 매핑)는 미확인** — 테넌트 RBP 구성에 의존(아래 "미확인/주의").

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|---|---|---|
| `SF_API_HOST` | 테넌트 DC 호스트 풀 origin (예 `https://api4.successfactors.com`). 테넌트 관리자/DC 목록에서 확인 | 필수 |
| `SF_COMPANY_ID` | 테넌트(회사) 식별자. 테넌트 관리자 확인 | 필수 |
| `SF_CLIENT_ID` | Admin Center > Manage OAuth 2.0 Client Applications 발급 **API Key** (= `client_id` + assertion `api_key`) | 필수 |
| `SF_USER_ID` | 기술계정 사용자ID (예 `sfadmin`; assertion `NameID` + 토큰 `user_id`) | 필수 |
| `SF_PRIVATE_KEY` | 다운로드한 `certificate.pem`에서 추출한 **개인키 PEM 본문** (개행은 `\n`) | `SF_PRIVATE_KEY_PATH`와 택1 |
| `SF_PRIVATE_KEY_PATH` | 개인키 PEM **파일 경로** | `SF_PRIVATE_KEY`와 택1 |
| `SF_ASSERTION_TTL` | assertion 유효시간(초). 기본 600 | 선택 |
| `SF_DEFAULT_PAGE_SIZE` | 기본 `top`(권장 ≤500). 기본 100 | 선택 |

> **시크릿 주의:** `SF_PRIVATE_KEY`·`SF_CLIENT_ID`는 코드/로그/`.mcp.json`에 실값을 쓰지 말 것. `.env`(gitignore) 또는 시크릿 매니저로만 주입. `.env.example`에는 빈 값만.

## 출처 (공식 URL)

- OAuth 클라이언트 등록 + X.509 키 발급 (Admin Center > Manage OAuth 2.0 Client Applications): https://community.sap.com/t5/technology-blog-posts-by-members/sap-api-management-generating-oauth-saml-assertion-for-successfactors-api/ba-p/14140473 , https://www.cloudworks.group/en/articles/sap-successfactors-odata-oauth-ias-saml-assertion
- `/oauth/idp` 폐기 확정 (KBA 3239495 "2H 2022: Deprecation of OAuth IdP API /oauth/idp"): https://userapps.support.sap.com/sap/support/knowledge/en/3239495
- 토큰 교환 파라미터(`user_id` 포함) / 오프라인 SAML 서명 레퍼런스: https://github.com/piejanssens/sf-oauth
- 레거시 Basic Auth 삭제(2026-11-20): https://userapps.support.sap.com/sap/support/knowledge/en/3146449
- RBP 권한 통제 (Permission Settings): https://help.sap.com/docs/successfactors-platform/sap-successfactors-api-reference-guide-odata-v2/permission-settings
- 베이스 URL / DC 호스트 패턴: https://www.lorenzo-datasolutions.com/sap-successfactors-api-urls/ , https://userapps.support.sap.com/sap/support/knowledge/en/2089448
- OData V2 개요: https://help.sap.com/docs/successfactors-platform/sap-successfactors-api-reference-guide-odata-v2/about-sap-successfactors-odata-apis-v2

## 미확인 / 주의

- **RBP 최소권한 매핑(엔티티/필드 ↔ 권한 항목):** 미확인 — 정본 본문이 SAP 로그인벽. 테넌트 RBP 구성 의존이라 고객사 관리자와 협의 필요.
- **DC 호스트 정본 목록("List of API Servers"):** 로그인벽으로 본문 미확인. 패턴/예시(`api4.successfactors.com`, `api17~44.sapsf.com`, `api2.successfactors.eu`, `api15.sapsf.cn` 등)는 확정. **반드시 테넌트 관리자에게 정확한 호스트를 확인할 것.**
- **함정:** 토큰 교환 시 `user_id` 누락하면 발급 실패. Audience(`www.successfactors.com`)/Recipient(`{SF_API_HOST}/oauth/token`) 오타 시 401. assertion 시계 오차(clock skew)로 `NotBefore` 실패 빈번 → UTC·여유시간 처리 필요.
- **refresh 토큰 없음:** `access_token`은 약 86399초(~24h). 만료 시 assertion 재생성·재교환으로 갱신.
- **레이트리밋 고정 수치 / 세부 SAP 에러코드 표:** 미확인(라이선스별 상이, 로그인벽) → 429/5xx/타임아웃은 백오프로 가드.
