# 키 발급 가이드 — 스마트빌 전자세금계산서 NxAPI (smartbill)

> 근거: `_workspace/smartbill/02_validation.md`(검증일 2026-06-26) · `mcp-servers/smartbill/README.md`
> 원칙: 검증된(출처 있는) 사실만 기재. 불명확한 발급 디테일은 "미확인"으로 명시.

## 연동 유형 / 인증 방식
- **연동 유형**: REST API (전 엔드포인트 REST/JSON). 공식 원격 MCP 커넥터 없음 → 로컬 MCP 서버 빌드.
- **인증 방식**: API Key → 단기 Bearer Token(10분) **2단계**.
  1. `GET https://{domain}/Auth/v1` — 헤더 `X-API-KEY: {API_KEY}`, `X-API-COMREGNO: {사업자번호 10자리}`, `Content-Type: application/json` (Body 없음). 응답 `token` 필드 사용, 유효 10분(`expireDateTime`).
  2. 이후 모든 데이터 API 호출 헤더: `Authorization: Bearer {token}`, `X-API-MESSAGEID: {20~36자, 영숫자+-}`, `X-API-COMREGNO: {사업자번호}`, `Content-Type: application/json`.
- **OAuth scope 개념 없음** — 권한은 API Key 단위. (출처: api_token / api_singleDti / guide_overview)

## 발급 절차 (스텝)
승인 게이트 기반(데모키 → 운영키). API Key는 별도 "발급 버튼"이 없고, 승인 완료 시 **자동 생성되어 마이페이지에서 확인**한다. (출처: https://developers.smartbill.co.kr/guide/dev_prep)

**발급 위치(콘솔)**
- 개발자센터: **https://developers.smartbill.co.kr/** (우측 상단 [로그인] / [간편회원가입])
- 로그인 콘솔(직접 진입): **https://mgr-developers.smartbill.co.kr/login**
- API Key 발급/확인/갱신 화면: **마이페이지** (콘솔 로그인 후)

**데모(개발용) API Key — 메뉴 경로**
1. **사업자등록번호 보유 + 스마트빌 회원가입.** (미가입 시 스마트빌 본 서비스 가입 선행)
2. 개발자센터 로그인 → **마이페이지 → [일반회원 전환] 버튼 클릭** (STEP 02 일반회원 전환).
3. 스마트빌 회원 여부 확인 → **스마트빌 기업관리자 승인 대기.**
4. 기업관리자가 일반회원 인증을 승인하면 **데모 API Key가 자동 생성** → **마이페이지에서 확인** (STEP 03).
5. 데모 환경(`demonxapi.smartbill.co.kr`)에서 연동/조회 테스트. 발급 세금계산서 상태·서비스 로그는 **대시보드 → 데모서버** 메뉴에서 조회 (STEP 04). (데모는 실제 발행·세무신고 비활성, 조회·포맷 확인용)

**운영 API Key — 메뉴 경로**
6. 일반회원 전환 후 **마이페이지 → [운영전환] 버튼 클릭**으로 운영전환 신청.
7. **기업관리자가 운영전환 인증 승인** → 운영 API Key **자동 생성** → **마이페이지에서 확인.** (운영서버 로그는 대시보드 → 운영서버)
8. 발행 기능 사용 시 **유료 서비스 가입** 필요.

**키 관리·기타**
- API Key가 노출된 경우 **마이페이지에서 새 API Key로 갱신** 가능.
- 발급된 API Key + 사업자등록번호(10자리)를 환경변수로 설정.

> **직링크 주의**: "클릭하면 바로 발급되는" 워크스페이스별 발급 직링크는 존재하지 않는다(키는 버튼 클릭이 아닌 승인 완료 시 자동 생성). 발급/확인은 콘솔 로그인 후 **마이페이지** 화면에서 처리한다. 연동/도입 문의는 개발자센터 [고객지원 → 연동문의] 또는 고객센터 1588-8064.

## 전제조건
- **사업자등록번호 보유**(필수).
- **스마트빌 회원 + 기업관리자 승인**(데모키 발급 게이트).
- **운영전환 승인**(운영키 발급 게이트).
- **유료 서비스 가입**(발행 기능 사용 시).
- 발신번호 사전등록: **해당 없음**(전자세금계산서 NxAPI는 SMS/발신번호 기반 아님).

> 본 MCP 서버 1차 릴리스는 **읽기/조회 도구만** 포함하므로, 실호출 검증에는 승인된 **데모 키**로 충분하다. (발행 계열 6종은 미구현 — certPassword RSA·인증서 등록·taxInvoice V3 스키마 미확정으로 보류)

## 스코프 / 권한
- **OAuth scope 없음.** 권한은 발급된 API Key 자체에 귀속(API Key 단위 권한).
- 데모 키 = 데모 도메인(조회/포맷 확인, 발행·세무신고 비활성), 운영 키 = 운영 도메인(실거래).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `SMARTBILL_API_KEY` | 발급받은 API Key(데모/운영). `X-API-KEY`, `Auth/v1` 토큰 호출에 사용 | 필수 |
| `SMARTBILL_COMREGNO` | 사업자등록번호 10자리. `X-API-COMREGNO`/`X-API-KEY` 호출에 사용 | 필수 |
| `SMARTBILL_ENV` | `demo`(→`demonxapi.smartbill.co.kr`) \| `prod`(→`nxapi.smartbill.co.kr`). 미설정 시 `demo` | 선택(기본 demo) |

## 출처 (공식 URL)
- 개발자센터(콘솔) 메인·로그인/간편회원가입·상단 메뉴(개발가이드/API레퍼런스/대시보드/고객지원): https://developers.smartbill.co.kr/
- 개발자 로그인 콘솔(직접 진입): https://mgr-developers.smartbill.co.kr/login
- 키 발급 절차(STEP 02 마이페이지 [일반회원 전환] 버튼 / STEP 03 데모 API Key 자동생성·마이페이지 확인 / STEP 04 대시보드 데모서버): https://developers.smartbill.co.kr/guide/dev_prep
- 운영전환(마이페이지 [운영전환] 버튼 → 기업관리자 승인 → 운영 API Key 자동생성·마이페이지 확인·갱신): https://developers.smartbill.co.kr/guide/ops_transition
- 인증 흐름(개요): https://developers.smartbill.co.kr/guide/guide_overview
- 토큰 발급 사양(GET `/Auth/v1`, 헤더, 10분, 응답 `token`): https://developers.smartbill.co.kr/refer/api_token
- 도메인/베이스 URL(운영 nxapi / 데모 demonxapi, 데모는 발행 비활성): https://developers.smartbill.co.kr/refer/api_overview
- 공통 헤더·REST 사양: https://developers.smartbill.co.kr/refer/api_singleDti

## 미확인 / 주의
- **발급 위치 확정**(2026-06-28 보강): 콘솔=developers.smartbill.co.kr(로그인 콘솔 mgr-developers.smartbill.co.kr/login), API Key 발급/확인/갱신 화면=**마이페이지**, 경로=마이페이지 [일반회원 전환]→승인→데모키 자동생성 / 마이페이지 [운영전환]→승인→운영키 자동생성. 워크스페이스별 "원클릭 발급 직링크"는 부재(승인 완료 시 자동 생성).
- **데모 환경은 발행·세무신고 비활성** — 발행 기능은 데모에서 테스트 불가(조회/포맷 확인용).
- **X-API-MESSAGEID** 는 호출마다 20~36자 고유값(앞 8자리 고객사 코드 권장 + UUID 일부)으로 생성해야 함. "정확히 36자 GUID"는 부정확.
- **토큰 10분 유효** — 메모리 캐시 권장(서버가 내부 처리). `/Auth/v1` 호출만 `X-API-KEY` 사용, 나머지는 `Authorization: Bearer`.
- **레이트리밋/쿼터 미문서화** — 공식 명시 없음. 자체 보수적 재시도·토큰 캐싱 권장.
- 발행 계열(certPassword RSA 암호화·인증서 등록 방식)·Webhook 전체 스펙은 미확정이라 본 릴리스 범위 밖.
