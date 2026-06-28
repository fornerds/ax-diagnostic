# 키 발급 가이드 — 다우오피스 (daouoffice)

> 근거: `_workspace/daouoffice/02_validation.md`(검증일 2026-06-26, 공식 본문 1:1 대조) · `mcp-servers/daouoffice/README.md`
> 원칙: 출처 있는 사실만 기재. 본문 미확보 항목은 "미확인"으로 남긴다.

## 연동 유형 / 인증 방식

- **연동 유형**: 다우오피스(DOAS) OpenAPI(REST) — 로컬 빌드 MCP 서버(공식 원격 MCP 커넥터 없음).
- **인증 방식(현행 표준)**: OAuth2 `client_credentials` → Bearer 토큰.
  - (운영자 1회) 인증키 발급으로 `clientId` / `clientSecret` 확보 → 환경변수 주입.
  - (런타임, 서버 자동) `POST https://api.daouoffice.com/public/auth/v1/oauth2/token`
    - 헤더: `Authorization: Basic base64("{clientId}:{clientSecret}")`, `Content-Type: application/x-www-form-urlencoded`
    - 바디: `grant_type=client_credentials`
    - 응답: `{ token_type:"bearer", expires_in:86400, access_token }` → 메모리 캐시(TTL 86400초).
  - 이후 모든 호출에 `Authorization: Bearer {access_token}` 부착.
- **레거시(하위호환)**: 일부 기능 스펙은 본문(body)에 `clientId`/`clientSecret`을 받는 예시가 남아 있다. body 인증은 레거시이며 순차적으로 토큰 기반으로 전환 예정. 서버는 과도기 안전을 위해 Bearer 헤더 + body 자격증명을 함께 전송한다.

## 발급 절차 (스텝)

> 인증키 발급/조회 API는 **최고관리자 ID/PW(평문)** 를 요구한다. 따라서 **운영자가 사전에 1회 수동 수행**하고, MCP 서버에는 결과로 받은 `clientId`/`clientSecret`만 주입한다. MCP 서버에 admin PW를 두지 않는다.

### 발급 위치 (직링크 vs 메뉴 경로)

- **발급 페이지 직링크: 없음.** 다우오피스 OpenAPI 인증키 발급은 **콘솔 UI 버튼이 아니라 REST API 호출(`POST /public/v2/alliance/company`)로 수행**된다. 또한 관리자 콘솔은 고객사 자체 그룹웨어 도메인(`https://{고객사}.daouoffice.com`)에 있어 **공용 직링크 URL이 존재하지 않는다**. 따라서 "직링크 없음 + 아래 메뉴 경로 + API 호출"이 정확한 발급 방식이다.
- **관리자 콘솔 메뉴 경로(OpenAPI 활성화/연동 관리, 최고관리자 권한)**:
  - OpenAPI 기능 확인·연동 관리: **통합설정⚙️ → 시스템연동 → 연동관리 → 오픈API** (메뉴 미노출 시 고객센터 1599-9460 문의)
  - Access Token 발행(현행 표준 인증): **OpenAPI → 인증키 연동 → 인증 Token 발행**
  - (근태 연동 시) 자동 동기화 주기: **통합설정⚙️ → 시스템 연동 → 연동 관리 → [OpenAPI]** 에서 "API 추가"
- **인증키(Client ID/Secret) 발급 자체**: 위 메뉴에는 활성화/관리 기능이 있고, **키 생성·조회는 운영자가 아래 2~4번 API를 호출해 수행**한다(엔터프라이즈형은 별도 신청 없이 기본 제공). 방화벽 사전 점검: 서버 IP `35.216.3.121`, Port 443(권장)/80 허용.

1. **상품 유형 확인** — 고객사가 엔터프라이즈형(단독형/설치형/구축형)인지 먼저 확인한다(전제조건 참조).
2. **인증키 발급 요청** — `POST https://api.daouoffice.com/public/v2/alliance/company` (JSON)
   - 필수 Body: `siteUrl`, `adminId`(최고관리자 이메일), `adminPw`(최고관리자 비밀번호), `apiType`, `partnerCode`
   - 선택 Body: `productName`, `productVersion`, `clientCompanyName`
   - `apiType` 값(1개): `APPR`(전자결재) / `ATTND`(근태) / `ACCOUNT`(계정정보) / `DEPT`(조직도)
   - `partnerCode` 값: `OPENAPI_D`(공개용) / `BIZPLAY` / `KSYSTEM`(영림원) / `ADTCAPS` / `TELECOP`(KT텔레캅) / `SECOM`(에스원)
   - 고객사당 1키, 재호출 시 동일 키 반환.
3. **clientId / clientSecret 확인** — 발급 응답에서 `clientId`/`clientSecret`을 받는다.
4. **(필요 시) 인증키 조회** — `POST https://api.daouoffice.com/public/v1/alliance/key` (JSON)
   - 발급과 동일 파라미터(adminId/adminPw 필요), `apiType` 없이도 조회 가능. 응답에 `clientId`/`clientSecret`.
5. **(Works 사용 시) Works 토큰 + 앱 설정** — Works 도구를 쓰려면 Works 전용 `token`(사용자 계정 기준 발급)과 Works 앱의 "외부데이터 가져오기" ON이 선행되어야 한다. → 전제조건/미확인 참조.
6. **환경변수 주입** — 받은 값을 아래 "환경변수 매핑"대로 설정한다.

## 전제조건

- **상품 유형**: 엔터프라이즈형(단독형/설치형/구축형)만 OpenAPI 제공. **SaaS(서비스형/공유형)·무료 상품은 OpenAPI 미제공** → 호출 시 401/권한오류로 실패. (빌드 전 필수 확인)
- **최고관리자 계정**: 인증키 발급/조회에 최고관리자 ID(이메일)/PW가 필요. 운영자가 보유·수행.
- **Works 도구 사용 시**: Works 전용 `token` + Works 앱 "외부데이터 가져오기" ON 선행.
- **사업자 심사 / 발신번호 사전등록**: 없음(출처에서 확인된 별도 심사·등록 절차 없음).

## 스코프 / 권한

- **OAuth2 스코프 개념 없음.** 권한은 인증키 발급 시 `apiType`(APPR/ATTND/ACCOUNT/DEPT)로 분기된다.
- **1개 clientId가 여러 apiType를 동시에 커버하는지는 미확인** → 기능군별로 별도 키가 필요할 수 있다(운영 가드). 권한오류 발생 시 해당 기능군 apiType로 별도 발급을 검토.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|---|---|---|
| `DAOUOFFICE_CLIENT_ID` | 인증키 발급(`/public/v2/alliance/company`) 또는 조회(`/public/v1/alliance/key`) 응답의 `clientId` | 예 |
| `DAOUOFFICE_CLIENT_SECRET` | 위 응답의 `clientSecret` | 예 |
| `DAOUOFFICE_BASE_URL` | 기본 `https://api.daouoffice.com` (오버라이드 불필요 시 생략) | 아니오(기본값) |
| `DAOUOFFICE_WORKS_TOKEN` | Works 전용 토큰(사용자 계정 기준 발급) — 발급 경로 미확인(아래 참조) | Works 도구 사용 시 예 |
| `DAOUOFFICE_PRODUCT_NAME` | 운영자 임의 지정(로그 추적용 제휴 제품명) | 아니오 |
| `DAOUOFFICE_PRODUCT_VERSION` | 운영자 임의 지정(로그 추적용 버전) | 아니오 |
| `DAOUOFFICE_CLIENT_COMPANY_NAME` | 운영자 임의 지정(로그 추적용 고객사 식별) | 아니오 |

> **admin ID/PW는 환경변수에 두지 않는다.** 인증키 발급/조회는 운영자가 사전에 수행하고 위 clientId/clientSecret만 주입한다.

## 출처 (공식 URL)

- 인증키 발급 + 베이스 URL + 상품 자격: https://helpdesk.daouoffice.co.kr/hc/ko/articles/48695613027609
- OpenAPI 이용 안내(관리자 콘솔 메뉴 경로·연동 관리 위치·서버 IP/Port): https://helpdesk.daouoffice.co.kr/hc/ko/articles/48339915864857
- Access Token 발행 API(OAuth2 client_credentials): https://helpdesk.daouoffice.co.kr/hc/ko/articles/51882238222361
- 인증키 조회: https://helpdesk.daouoffice.co.kr/hc/ko/articles/51397767484953
- Works 등록/수정/삭제(+ token 필요·외부데이터 ON 선행): https://helpdesk.daouoffice.co.kr/hc/ko/articles/49712631882777
- 베이스 URL(고정) 및 알림 스펙: https://manual.daouoffice.co.kr/hc/ko/articles/24410057065753

## 미확인 / 주의

- **Works `token` 발급 경로**: "사용자 계정 기준 발급"이라는 사실만 확인됨. 발급 화면/경로는 본문 미확보 → **미확인**(UI 발급으로 추정). MCP 서버는 발급하지 않고 환경변수로 주입받는다.
- **인증키 재발급 경로/바디**: 문서 존재만 확인, 본문 미확보 → **미확인**(운영 1회성).
- **다중 apiType 단일키 커버 여부**: **미확인**. 권한오류 시 기능군별 별도 키 필요 가능.
- **body ↔ Bearer 과도기 호환**: 토큰 경로가 막힌 환경에서의 동작은 문서로 단정 불가 → 운영자 런타임 확인 필요.
- **공식 SDK(npm/pip)**: 미발견. JSP 샘플만 제공 → 직접 HTTP 구현. (발급 자체와 무관, 참고)
- **보안 주의**: 인증키 발급/조회 API는 최고관리자 PW를 평문 전송한다. 발급은 안전한 운영 채널에서 1회 수행하고 PW를 저장·재사용하지 않는다.
