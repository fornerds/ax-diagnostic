# 키 발급 가이드 — 한글과컴퓨터 한컴 SDK 클라우드 (hangul_office)

이 커넥터는 한컴 SDK 클라우드 REST API의 두 서비스(OCR API, 데이터 로더 API)를 사용합니다.
두 서비스는 동일 base·동일 엔드포인트를 공유하지만 **API Key가 서비스별로 분리 발급**됩니다. 사용하려는 도구의 키만 발급받으면 됩니다.

## 연동 유형 / 인증 방식
- **연동 유형**: 클라우드 REST API (자가발급 API Key). OAuth/토큰 교환 없음 — 정적 키.
- **인증**: 모든 요청에 HTTP 헤더 `X-API-Key: <api_key>`.
- **전송 보안**: HTTPS 필수 (HTTP 요청은 거부됨).
- 출처: https://documents.sdk.hancom.com/ocr-api/intro , https://documents.sdk.hancom.com/ocr-api/important-notes

## 발급 절차 (스텝)
공식 문서의 "API 키 발급 방법"(6단계)에 따른다. OCR / 데이터 로더 모두 동일 절차이며 **대상 서비스만 다르게 선택**한다.

**발급 시작점(직링크)**: https://sdk.hancom.com (로그인) → 마이페이지 https://sdk.hancom.com/orders
**메뉴 경로(버튼명)**: 로그인 → **우측 상단 프로필 → 마이페이지** → 좌측 **API 관리** 탭 → 대상 서비스의 **[API Key 관리]** 버튼 → **[API Key 생성]** 버튼 → 키 이름·메모 입력 → **[생성하기]** 버튼

1. **회원가입·로그인** — https://sdk.hancom.com 에서 한컴 SDK 계정으로 회원가입 후 로그인.
2. **마이페이지 진입** — 우측 상단 프로필 메뉴 → "마이페이지"(루트 URL https://sdk.hancom.com/orders).
3. **API 관리** — 좌측 사이드바의 "API 관리" 탭 선택.
4. **대상 서비스 선택** — 서비스 목록에서 발급할 서비스(OCR / 데이터 로더)의 **[API Key 관리]** 버튼 클릭.
5. **API Key 생성** — 우측 **[API Key 생성]** 버튼 클릭.
6. **이름·메모 입력 후 생성** — 키 이름과 메모를 입력하고 **[생성하기]** 버튼을 눌러 발급한 뒤, 생성된 키 값을 복사해 안전하게 보관(발급 직후에만 전체 노출될 수 있으므로 즉시 보관).

> 출처: https://documents.sdk.hancom.com/ocr-api/intro (OCR), https://documents.sdk.hancom.com/dataloader-api/intro (데이터 로더). 두 서비스의 발급 절차가 동일하다고 명시됨(공식 "API 키 발급 방법" 원문 일치, 2026-06 재확인).
> **API 관리/키 생성 화면 자체로 바로 가는 직링크(딥링크)는 없음**: 공식 문서·sdk.hancom.com 어디에도 발급 화면 전용 URL이 없고, 로그인 후 마이페이지(https://sdk.hancom.com/orders) 안에서 위 메뉴 경로로 진입해야 함. (마이페이지 내부는 로그인 세션 기반 SPA라 키 생성 화면에 고정 URL이 부여되지 않음.)

## 전제조건
- **한컴 SDK 계정**(sdk.hancom.com) 필요.
- **크레딧(유료)** 필요 — 키 발급은 무료(자가발급 가능)이나 실제 변환 사용은 크레딧을 소모하는 유료 서비스.
  - OCR = 이미지 1장당 2 크레딧
  - 데이터 로더 = 문서 1페이지당 10 크레딧
  - 크레딧 부족 시 API가 403을 반환. 변환 실패(지원불가형식·손상파일·서버오류·암호보호 등) 시 크레딧 자동 환불.
- **발신번호 사전등록 / 사업자 심사 등**: 해당 없음(이 API는 문자/발신 서비스가 아님).
- **무료 크레딧·무료체험 한도, 회원가입 시 사업자 인증 필요 여부**: 미확인("미확인" 참조).
- 출처: https://documents.sdk.hancom.com/ocr-api/intro , https://documents.sdk.hancom.com/dataloader-api/intro

## 스코프 / 권한
- API Key 기반 단순 인증으로, **별도의 OAuth 스코프 개념 없음**.
- **키 1개 = 서비스 1개 소속**. OCR 키로는 OCR 도구만, 데이터 로더 키로는 데이터 로더 도구만 동작.
- 잘못된 서비스의 키로 호출하면 403("API KEY가 유효하지 않습니다") 발생 가능.
- 출처: 02_validation.md 누락·리스크 #6 (https://documents.sdk.hancom.com/dataloader-api/intro)

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `HANCOM_OCR_API_KEY` | 마이페이지 → API 관리 → **OCR** 서비스 → API Key 생성 | OCR 도구 사용 시 필수 |
| `HANCOM_DATALOADER_API_KEY` | 마이페이지 → API 관리 → **데이터 로더** 서비스 → API Key 생성 | 데이터 로더 도구 사용 시 필수 |
| `HANCOM_API_BASE_URL` | base URL 오버라이드. 기본값 `https://api.sdk.hancom.com` | 선택(기본값 사용) |

> 두 키 중 **최소 하나**는 있어야 서버가 의미 있게 동작. 설정되지 않은 키에 해당하는 도구는 호출 시 "키 미설정" 오류를 명확히 반환한다.
> 키는 코드/.mcp.json 에 실값으로 쓰지 말고 환경변수 또는 `${VAR}` 확장으로 주입한다(하드코딩 금지).

## 출처 (공식 URL)
- 인증(X-API-Key): https://documents.sdk.hancom.com/ocr-api/intro
- HTTPS 필수: https://documents.sdk.hancom.com/ocr-api/important-notes
- API Key 발급 절차(OCR, "API 키 발급 방법" 6단계 원문): https://documents.sdk.hancom.com/ocr-api/intro
- API Key 발급 절차(데이터 로더, 동일 6단계): https://documents.sdk.hancom.com/dataloader-api/intro
- 발급 시작점/마이페이지: https://sdk.hancom.com → https://sdk.hancom.com/orders (마이페이지 루트; 발급 화면 전용 직링크는 없음)
- 크레딧 정책(OCR 2C/이미지): https://documents.sdk.hancom.com/ocr-api/intro
- 크레딧 정책(데이터 로더 10C/페이지): https://documents.sdk.hancom.com/dataloader-api/intro
- base URL `https://api.sdk.hancom.com` + prefix `/api/api-services`: https://documents.sdk.hancom.com/ocr-api/api/conversion-request

## 미확인 / 주의
- **발급 화면 전용 딥링크는 존재하지 않음(확인됨)**: 발급 절차(6단계)·메뉴 경로·버튼명은 공식 문서로 검증됨. 마이페이지 루트 URL은 https://sdk.hancom.com/orders 로 확인되나, "API 관리 → API Key 관리 → API Key 생성" 화면 자체로 바로 가는 고정 URL(딥링크)은 공식 문서·사이트 어디에도 없음(로그인 세션 기반 SPA). → 위 메뉴 경로로 진입한다. (추정 아님: 직링크 부재를 명시적으로 확인함, 2026-06.)
- **무료 크레딧/무료체험 한도, 사업자 인증 필요 여부 미확인**: 자가발급은 가능하나 무료 한도·가입 시 사업자 인증 요구 여부는 미확인(요금/가입 페이지 SPA).
- **크레딧 차감 시점 문서 상충**: 같은 공식 문서가 OCR 차감 시점을 "DONE 후"(intro)와 "업로드 시 즉시"(API 레퍼런스)로 다르게 기술. 발급과 무관하나 과금 인지용 주의 — 실패 시 자동 환불은 양쪽 일치.
- **유료 사용**: 키 발급 자체는 무료지만 실제 변환은 크레딧을 소모하는 유료 서비스임을 사용자에게 명시.
