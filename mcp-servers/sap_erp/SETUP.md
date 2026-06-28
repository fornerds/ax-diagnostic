# 키 발급 가이드 — SAP ERP (sap_erp)

> 대상: **SAP S/4HANA**(Cloud Public/Private Edition / On-Premise)의 표준 OData(V2/V4).
> 출처: `_workspace/sap_erp/02_validation.md`(공식문서 대조 검증본, 검증일 2026-06-26) + `README.md`.
> **핵심 전제 — 셀프서비스 아님:** 공개 API 키가 존재하지 않는다. 각 고객 테넌트마다 SAP 관리자(Basis)가 통신(Communication) 설정을 사전 구성해야만 자격증명이 발급된다. 아래 "발급 절차"는 그 관리자 작업이다.

## 연동 유형 / 인증 방식
- **연동 유형:** REST(OData V2/V4) over HTTPS. 재무전표 쓰기는 SOAP(본 서버 범위 외).
- **인증 방식(둘 중 택1, env로 결정):**
  - **OAuth 2.0 — Client Credentials**(권장, 서버-서버 백엔드 통합에 적합). SAML Bearer / Authorization Code도 시스템 설정에 따라 가능.
  - **Basic Authentication**(Communication User ID / 비밀번호).
- HTTPS 필수(OAuth는 HTTP 불가).
- **쓰기(POST/PUT/PATCH/DELETE)**는 X-CSRF-Token 선행 Fetch가 필요(서버가 자동 처리 — 발급 단계와 무관).

## 발급 절차 (스텝)
> 모두 **고객 테넌트의 SAP 관리자(Basis/관리자 권한)** 작업이다. 셀프서비스 콘솔이 아니라 S/4HANA 시스템 내 통신 관리 앱에서 수행한다.

1. **전제 확인:** 고객사가 S/4HANA 라이선스를 보유하고, 관리자 권한이 있는지 확인한다.
2. **Communication User 생성** — Fiori 앱 *Maintain Communication Users*에서 통신 사용자(ID/비밀번호 또는 인증서)를 생성한다. → Basic 인증의 `SAP_USERNAME`/`SAP_PASSWORD`가 여기서 나온다.
3. **Communication System 생성** — Fiori 앱 *Communication Systems*에서 Inbound 전용 시스템을 만들고, 위 통신 사용자를 사용자(User for Inbound Communication)로 지정한다. OAuth 사용 시 이 단계에서 OAuth 2.0 설정(Client ID/Secret, Authorization Code 시 Redirect URI)을 등록한다. → OAuth의 `SAP_CLIENT_ID`/`SAP_CLIENT_SECRET`이 여기서 나온다.
4. **Communication Arrangement 생성(시나리오별)** — Fiori 앱 *Communication Arrangements*에서 사용할 비즈니스 시나리오를 추가한다. 시나리오가 곧 권한 범위다(아래 "스코프/권한"). 생성하면 **테넌트별 서비스 엔드포인트(호스트)**가 표시된다. → `SAP_BASE_URL`이 여기서 확인된다.
5. **토큰 엔드포인트 확인(OAuth 시)** — 테넌트의 OAuth 토큰 URL(예: `https://myXXXX.authentication.<region>.hana.ondemand.com/oauth/token`)을 확인한다. → `SAP_OAUTH_TOKEN_URL`.
6. **자격증명 인계** — 위에서 얻은 호스트·토큰 URL·Client ID/Secret(또는 User/PW)을 본 MCP 서버의 환경변수로 주입한다(실값 하드코딩 금지).

> 참고: 호출 검증 시 `GET <SAP_BASE_URL>/sap/opu/odata/sap/<SERVICE>/$metadata`로 엔티티셋·필드·가용 메서드를 확인한다(에디션/릴리스에 따라 표면이 다름).

## 전제조건
- **S/4HANA 라이선스** 보유(고객 테넌트). — 필수
- **관리자(Basis) 작업**으로 Communication User + Communication System + Communication Arrangement(시나리오) 사전 구성. — 필수, 셀프서비스 불가
- **에디션 차이:** Cloud Public Edition = 화이트리스트 공개 API만(커스텀 RFC 불가), Private/On-Premise = 더 넓음. 동일 "SAP ERP"라도 가용 엔드포인트가 다르다.
- 사업자 심사·발신번호 사전등록 등 한국 통신/금융 특화 전제: **없음**(해당 없음). 단 한국 현지화(전자세금계산서/DRC)는 표준 API 밖이며 본 서버 범위 외.

## 스코프 / 권한
- **고정 scope 문자열 없음.** SAP는 권한을 OAuth scope 목록이 아니라 **Communication Arrangement(통신 시나리오 / 비즈니스 카탈로그) 단위**로 부여한다. "어느 작업이 되는가"는 부여된 Arrangement에 종속된다. → 구현은 scope를 하드코딩하지 않는다(미확인 항목으로 명시됨).
- 본 서버가 사용하는 시나리오(발급 단계 4에서 추가):

| 시나리오 ID | 용도 |
|------|------|
| `SAP_COM_0109` | 판매오더 (`API_SALES_ORDER_SRV`) |
| `SAP_COM_0008` | 비즈니스파트너 (`API_BUSINESS_PARTNER`) — Customer/Supplier Integration |
| `SAP_COM_0053` | 구매오더 (`API_PURCHASEORDER_PROCESS_SRV`, V2 deprecated) |
| `SAP_COM_0009` | 제품/자재 마스터 (`API_PRODUCT_SRV`) |
| `SAP_COM_0106` | 출고/납품 (`API_OUTBOUND_DELIVERY_SRV`) — Delivery Processing Integration |

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `SAP_BASE_URL` | Communication Arrangement 생성 시 표시되는 테넌트 호스트 (예 `https://myXXXX.s4hana.cloud.sap`) | 필수 |
| `SAP_AUTH_TYPE` | 인증 방식 선택: `oauth_client_credentials`(기본) 또는 `basic` | 필수 |
| `SAP_OAUTH_TOKEN_URL` | 테넌트 OAuth 토큰 엔드포인트 (예 `https://myXXXX.authentication.<region>.hana.ondemand.com/oauth/token`) | OAuth 시 필수 |
| `SAP_CLIENT_ID` | Communication System의 OAuth 2.0 설정에서 발급된 Client ID | OAuth 시 필수 |
| `SAP_CLIENT_SECRET` | 동일 OAuth 2.0 설정의 Client Secret | OAuth 시 필수 |
| `SAP_USERNAME` | Communication User ID(Maintain Communication Users에서 생성) | Basic 시 필수 |
| `SAP_PASSWORD` | Communication User 비밀번호 | Basic 시 필수 |
| `SAP_SAP_CLIENT` | (선택) `sap-client` 클라이언트 번호(예 100) — 일부 시스템 필요 | 선택 |
| `SAP_PO_USE_V4` | (선택) 구매오더 V4 후속(`API_PURCHASEORDER_2`) 사용 토글(`true`) | 선택 |

> 시크릿(`SAP_CLIENT_SECRET`/`SAP_PASSWORD` 등)은 **전부 환경변수로 주입**한다. 코드·`.mcp.json`에 실값을 쓰지 말 것.

## 출처 (공식 URL)
- OAuth 2.0 + S/4HANA OData 호출(Client Credentials): https://community.sap.com/t5/technology-blog-posts-by-sap/step-by-step-calling-s-4hana-private-cloud-standard-odata-apis-with-oauth-2/ba-p/14346113
- Authorization Code 활성화 + Communication System/Arrangement 온보딩 절차(판매오더 A2X): https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/enable-oauth-2-0-authorization-code-for-sales-order-a2x-api/ba-p/13617148
- Communication Arrangements(Public Cloud) 개요: https://community.sap.com/t5/technology-blog-posts-by-members/communication-arrangements-in-sap-s-4-hana-public-cloud/ba-p/14334468
- 비즈니스파트너 OData 서비스(path·시나리오): https://help.sap.com/docs/SAP_S4HANA_CLOUD/3c916ef10fc240c9afc594b346ffaf77/85043858ea0f9244e10000000a4450e5.html
- X-CSRF-Token / E-Tag 동작: https://community.sap.com/t5/technology-blog-posts-by-members/s-4hana-cloud-x-csrf-token-and-e-tag-validation/ba-p/13427700
- 구매오더 V2 deprecated(SAP Note 3502308 / KBA 3360429 V4 후속): https://userapps.support.sap.com/sap/support/knowledge/en/3502308 , https://userapps.support.sap.com/sap/support/knowledge/en/3360429

## 미확인 / 주의
- **OAuth scope 문자열: 미확인(고정 목록 없음).** 권한은 Communication Arrangement(시나리오) 단위. scope 하드코딩 금지.
- **개별 엔티티셋·키·필드 스키마: 미확인(런타임 확정 대상).** 에디션/릴리스에 의존 → 운영 전 대상 시스템 `$metadata`로 확정.
- **구매오더 V2 deprecated:** `API_PURCHASEORDER_PROCESS_SRV`(V2)는 release 2308에서 폐기 표시(SAP Note 3502308). 현재 동작하나(최소 12개월 adoption 보장) 신규는 `SAP_PO_USE_V4=true`로 V4 후속(`API_PURCHASEORDER_2`) 권장. V4 컬렉션 path/엔티티셋은 시스템 카탈로그에 따라 다를 수 있어 `$metadata`로 확인.
- **재무전표 쓰기(생성/변경/청산):** SOAP이며 본 서버 범위 외(2단계 보류). 정식 WSDL·네임스페이스·필수 필드 미확인. 본 서버의 재무 도구는 읽기(`list_journal_entry_items`)만 제공.
- **한국 현지화(전자세금계산서/DRC):** 표준 OData 밖, 본 서버 제외. 필요 시 SAP DRC 별도 조사.
- **공식 원격 MCP 커넥터:** 없음(SAP–Anthropic은 Business AI Platform 임베드 발표 단계, 엔드유저용 공개 원격 엔드포인트 아님). 본 서버는 직접 build 경로.
