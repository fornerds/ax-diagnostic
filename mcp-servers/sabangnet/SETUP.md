# 키 발급 가이드 — 사방넷 OMS (sabangnet)

> 제공사: ㈜다우기술(DAOU TECH). 본 가이드는 **OMS(`sabangnet.co.kr`, RTL_API)** 전용입니다.
> 풀필먼트/WMS(`sbfulfillment.co.kr`, REST/JSON/HMAC)는 **별개 제품**이므로 그 인증/엔드포인트와 혼용하지 마세요.
> 모든 내용은 `_workspace/sabangnet/02_validation.md`(검증일 2026-06-26)의 출처 확인된 사실만 반영하며, 미확인 항목은 명시했습니다.

## 연동 유형 / 인증 방식
- **연동 유형**: API Key 기반 + XML/HTTP 교환 (RTL_API). OAuth/토큰 교환 **아님**.
- **인증 식별자**:
  - 사방넷 로그인 ID (= Shop ID)
  - 쇼핑몰 ID (기본정보 > 쇼핑몰관리 [Set-Up])
  - 비밀값: **API key (연동키)**
- **인증 전달 위치/파라미터명**: **미확인(가드)** — 쿼리 vs 헤더, 키명(`api_key`/`auth_key`/회사코드 등) 모두 공식 가이드(유료 게이팅)로만 확정 가능. WMS의 `LIVE-HMAC-SHA256`/`Credential(.../srwms_request)`/`Signature` 헤더는 OMS 근거가 없어 차용 금지.

## 발급 절차 (스텝)
1. 사방넷에 **유료 고객으로 가입** 후 로그인 (https://www.sabangnet.co.kr).
2. **"API 서비스" 부가서비스 결제** — 사방넷 사용료에 포함되지 않는 유료 부가서비스입니다 (가격은 아래 전제조건 참조).
3. **마이페이지 > 서비스 관리 > 연동키 관리**(= API 인증키 관리)로 이동.
4. 해당 화면에서 **API key(연동키) 확인/발급**.
5. 쇼핑몰 ID는 **기본정보 > 쇼핑몰관리 [Set-Up]** 에서 확인.
6. RTL_API 베이스 URL은 **고객사별·시기별로 다름**(예 `http://r.sabangnet.co.kr`, `http://r200.sabangnet.co.kr`). 자신의 계정에 세팅된 URL을 확인해 사용 (하드코딩 금지).

## 전제조건
- **사방넷 유료 고객 가입** 필수.
- **"API 서비스" 부가서비스 유료 결제** 필수. 공식 가격표(verbatim):
  - 기본(싱글/레귤러/브론즈): **50,000원/월**
  - 실버·골드: **100,000원**
  - 무제한 스페셜1·2: **300,000원**
  - 스페셜3·4: **400,000원**
  - 플래티넘1·2: **500,000원**
- 발신번호 사전등록·사업자 심사 같은 별도 게이트: **확인되지 않음** — API 고객 자체에 별도 심사 게이트가 있다는 근거는 발견되지 않았습니다(상담·결제 경로). (`addition_connect.html`(쇼핑몰연동개발)은 쇼핑몰이 셀러를 모집하는 별개 채널이며 API 서비스와 무관.)
- **공식 RTL_API 가이드(`r200.sabangnet.co.kr/RTL_API/guide`)는 외부에서 전 경로 HTTP 403**(유료 가입 고객 전용 게이팅)이므로, 엔드포인트별 입력/응답 XML 스키마는 가입 계정으로만 확보 가능.

## 스코프 / 권한
- 데이터 범위(공식): **상품·주문·재고** 정보 연동. 공식 verbatim: "ERP, WMS, 자체 시스템 등 타 솔루션과의 데이터 연동", "상품, 주문, 재고정보까지".
- 세분화된 스코프/권한 토글 체계: **미확인** — 공식 권한 모델 문서 없음. 권한은 가입한 "API 서비스" 상품군(싱글/실버/골드/스페셜/플래티넘) 단위로 갈리는 것으로 보이나, 도구별 권한 매핑은 확인되지 않음.

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `SABANGNET_API_BASE_URL` | 계정에 세팅된 RTL_API 베이스 URL(예 `http://r200.sabangnet.co.kr`). 고객사별 상이 — **하드코딩 금지** | 예 |
| `SABANGNET_LOGIN_ID` | 사방넷 로그인 ID(= Shop ID) | 예 |
| `SABANGNET_API_KEY` | 마이페이지 > 서비스 관리 > 연동키 관리에서 발급한 API 인증키(연동키) — 시크릿 | 예 |
| `SABANGNET_MALL_ID` | 기본정보 > 쇼핑몰관리 [Set-Up]의 쇼핑몰 ID | 작업에 따라(가드) |
| `SABANGNET_AUTH_PARAM_STYLE` | (가드) 인증 전달 방식 — 공식 가이드 확정 후 `query`/`header` 명시 | 가이드 확정 시 |
| `SABANGNET_MIN_INTERVAL_MS` | 호출 간 최소 간격(ms). 미설정 시 `1000`(보수적 기본값) | 아니오 |

## 출처 (공식 URL)
- API 서비스 소개·가격·제공사: https://www.sabangnet.co.kr/service-intro/api-service
- 연동키 발급 위치(마이페이지 > 서비스 관리 > 연동키 관리), 인증 식별자: https://qxguide.oopy.io/6dd1c266-4538-4fb1-bcb3-9b8be5a93795 , https://welcome.poomgo.com/guide/sabangnet
- 인증 식별자(쇼핑몰 샵 ID = 사방넷 로그인 ID + API key + 쇼핑몰 ID): https://www.analytics-docs.laplacetec.com/guide/startguide/connect/solution/sabangnet
- 베이스 URL 고객사별 변동: https://www.cheonyu.com/customer/sabangnet.html
- 쇼핑몰연동개발 vs API 서비스 구분: https://www.sabangnet.co.kr/affiliate/mall-connect
- 공식 가이드 호스트(외부 403): http://r200.sabangnet.co.kr/RTL_API/guide/index.html

## 미확인 / 주의
- **인증 전달 위치/파라미터명 미확인**: 쿼리 vs 헤더, 정확한 키명을 공식 가이드로 확정해야 함.
- **엔드포인트별 입력/응답 XML 스키마 비공개**: 공식 가이드가 유료 게이팅(403)이라 외부에서 확보 불가. 가입 계정으로만 입수 가능.
- **베이스 URL 절대 하드코딩 금지**: 고객사별·시기별로 다름(`r.` → `r200.` 등). 반드시 `SABANGNET_API_BASE_URL`로 외부화.
- **OMS ≠ WMS**: WMS(`sbfulfillment.co.kr`)의 HMAC/REST 스펙을 OMS에 적용하면 오류. 본 가이드는 OMS 전용.
- **레이트리밋/에러규약 미확인**: 사방넷 공식 수치 없음. (3자 솔루션의 "30분 주기"는 사방넷 API 규약이 아님.) 구현 측 보수적 백오프·최소 간격 기본값 사용.
- **403 수신 시**: "API 서비스 미가입 / IP·세션 게이팅 / 베이스 URL 오설정" 가능성을 의심. 검증 중 외부에서 모든 RTL_API 경로가 403이었습니다.
