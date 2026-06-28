# 키 발급 가이드 — 네이버 검색광고 (naver_ad)

> 범위: 검색광고(파워링크)만. GFA(성과형 디스플레이)는 공식 파트너사 전용(Beta)이라 일반 발급 불가 → 제외.
> 검증 출처: `_workspace/naver_ad/02_validation.md`(2026-06-26) + 공식 GitHub `naver/searchad-apidoc`.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 MCP 서버 빌드(공식 원격 커넥터 없음). 광고주 셀프 키 발급 + 공개 REST.
- **인증 방식**: API Key + **요청별 HMAC-SHA256 서명** (OAuth 아님, 토큰 만료/갱신 없음).
- 매 호출 헤더: `X-Timestamp`(epoch **밀리초** 문자열), `X-API-KEY`, `X-Customer`, `X-Signature`, `Content-Type: application/json; charset=UTF-8`.
- 서명 = `base64( HMAC_SHA256( SECRET_KEY, "{X-Timestamp}.{대문자 메서드}.{쿼리 제외 경로}" ) )`. (쿼리스트링은 서명에서 제외, path 변수는 포함)

## 발급 절차 (스텝)

1. 네이버 검색광고에 가입한다 — `https://searchad.naver.com` (광고주 본인 명의 계정).
2. 광고 관리 콘솔에 로그인한다 — `https://manage.searchad.naver.com`.
3. 콘솔 상단 **도구(Tools) > API 사용 관리(API Manager)** 로 이동한다.
4. **API 라이선스 생성(Create API license)** 을 실행한다.
5. 발급된 **CUSTOMER_ID(계정 고유번호)**, **API_KEY(액세스라이선스)**, **SECRET_KEY(비밀키)** 3종을 확인·복사한다.
6. 이 3개를 MCP 서버의 환경변수로 주입한다(아래 매핑표). 시크릿은 코드/`.mcp.json`에 직접 쓰지 않는다.

## 전제조건

- **네이버 검색광고 광고주 가입**: 사업자·실명 기반 가입 필요(키 발급 자체는 가입 계정에서 셀프 발급).
- **비즈머니**: 실제 광고 집행에는 비즈머니 충전이 선행. 단, **데이터 조회(통계·목록 등)는 키만으로 가능** — 비즈머니가 광고 집행 조건이지 API 호출 조건은 아님.
- **발신번호 사전등록 등 별도 심사**: 없음(검색광고 API는 광고주 셀프 발급, 별도 사업자 심사 절차 없음).
- 참고: GFA(성과형 디스플레이) API는 공식 파트너사만 신청·권한 부여 가능 → 일반 발급 불가(본 서버 범위 밖).

## 스코프 / 권한

- 별도 OAuth scope 개념 **없음**. 키가 해당 **CUSTOMER_ID의 권한에 종속**된다.
- 대행(한 키로 여러 광고주): `list_client_accounts`(GET `/customer-links?type=MYCLIENTS`)로 클라이언트 CUSTOMER_ID 목록을 얻고, 요청별로 **X-Customer 헤더만 광고주별로 교체**(키·서명은 동일)해 호출한다.
- 권한 부족 시 별도 콘솔 설정이 아니라 **런타임 에러 코드 1006/1018** 로 확인된다.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `NAVER_AD_CUSTOMER_ID` | manage.searchad.naver.com > 도구 > API 사용 관리의 **CUSTOMER_ID(계정 고유번호)**. 헤더 `X-Customer`로 사용(대행 시 대상 광고주 번호로 교체) | 필수 |
| `NAVER_AD_API_KEY` | 같은 화면의 **API_KEY(액세스라이선스)**. 헤더 `X-API-KEY` | 필수 |
| `NAVER_AD_SECRET_KEY` | 같은 화면의 **SECRET_KEY(비밀키)**. HMAC 서명 계산용 | 필수 |
| `NAVER_AD_BASE_URL` | 베이스 URL 오버라이드. 기본값 `https://api.searchad.naver.com` | 선택 |

## 출처 (공식 URL)

- 가입: `https://searchad.naver.com`
- 키 발급 콘솔: `https://manage.searchad.naver.com` → 도구(Tools) > API 사용 관리(API Manager) > API 라이선스 생성
- 공식 발급 안내(Getting Started): `https://github.com/naver/searchad-apidoc` ("Sign up … Go to manage.searchad.naver.com … Tools > API Manager … Create API license")
- 인증/서명 공식 샘플: `https://github.com/naver/searchad-apidoc/blob/master/python-sample/examples/signaturehelper.py`
- 운영 베이스 URL 근거: `https://github.com/naver/searchad-apidoc/blob/master/python-sample/examples/ad_management_sample.py` (`BASE_URL='https://api.searchad.naver.com'`)
- 에러 코드(1006/1018/429/1016/503): `https://github.com/naver/searchad-apidoc/blob/master/NaverSA_API_Error_Code_MAP.md`
- GFA 일반 발급 제한: `https://naver-ad-api.github.io/openapi-guide/docs/intro`

## 미확인 / 주의

- **콘솔 메뉴 한글 라벨**: 공식 README는 영문 "Tools > API Manager"로 표기. 한글 콘솔 메뉴명은 "도구 > API 사용 관리"로 정리했으나, 콘솔 UI 표기가 개편될 수 있어 정확 라벨은 **현재 콘솔 화면 기준으로 확인 권장**.
- **시각 동기화**: `X-Timestamp`가 서버 시각과 크게 어긋나면 서명이 거부된다. 시스템 시계(ms)를 신뢰·동기화할 것.
- **서명 함정**: 서명 path에 쿼리스트링을 포함하면 invalid-signature 발생. 쿼리는 URL에만 붙이고 서명 메시지에는 쿼리 제외 경로만 넣는다.
- **레이트리밋 임계값**: 정확 수치(초당/일일) 미공개. 429/1016 수신 시 백오프 재시도로 대응.
- **계약 제외(미확인)**: GFA 전체, `/api/ad-accounts`·`/api/manager-accounts`, `/api/tool/*`, 비즈머니 histories(충전/소진 이력) — 운영 경로가 공식 실행 샘플로 확인되지 않아 본 서버 미구현.
