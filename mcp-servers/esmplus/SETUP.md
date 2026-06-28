# 키 발급 가이드 — ESM플러스(지마켓·옥션) (esmplus)

> 출처 기준선: `_workspace/esmplus/02_validation.md`(검증일 2026-06-26) + `README.md`. 추측 없이 출처 있는 사실만 정리. 발급 디테일이 불명확한 항목은 "미확인"으로 남김.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 키 기반 MCP 서버(build). 공식 원격 MCP 커넥터·OAuth/토큰 발급 엔드포인트 **없음**.
- **인증 방식**: 클라이언트가 발급받은 **Secret Key**로 **HS256 JWT(Bearer)**를 직접 서명. 본 서버는 매 API 호출마다 즉석 서명한다(`exp` 미정의 → 캐시·재사용 회피).
  - Header: `{"alg":"HS256","typ":"JWT","kid":"<ESM_MASTER_ID>"}`
  - Payload: `{"iss":"www.esmplus.com","sub":"sell","aud":"sa.esmplus.com","iat":<현재 epoch초>,"ssi":"A:<옥션판매자ID>,G:<지마켓판매자ID>"}`
  - `ssi`는 **두 사이트 합본 형식** `"A:옥션ID,G:지마켓ID"`. 한쪽만 운영해도 합본 형식을 유지한다.
  - 호출 호스트는 `https://sa2.esmplus.com`(전 엔드포인트 공통). `aud` 클레임의 `sa.esmplus.com`과 다르며 정상.
  - 출처: https://etapi.gmarket.com/pages/API-%EA%B0%80%EC%9D%B4%EB%93%9C

## 발급 절차 (스텝)

이 API는 **공개 셀프서비스 발급이 아니다.** **발급 페이지 직링크는 존재하지 않으며**(셀프서비스 콘솔 화면 없음), 발급은 **이메일 신청 → 수동 심사** 방식이다. 다음 순서로 진행한다:

1. 지마켓·옥션 **판매자 가입**.
2. **ESM+(https://www.esmplus.com) 로그인** 후 **Master ID 생성**.
3. 개발자센터 **ESM Trading API 문서 포털**(https://etapi.gmarket.com 또는 https://etapi.ebaykorea.com)의 **API 가이드** 안내에 따라, **etapihelp@gmail.com** 으로 사용 신청 이메일 발송 → **이메일 심사**.
   - 신청서 기재 항목: **개발 API 범위(사용하려는 API 리스트)**, **ESM 마스터ID**, **서비스 중인 사이트 URL**, **최근 3개월 매출 규모**, **API 개발 기간**, **호출 IP**.
4. 심사 통과 후 **Secret Key 수동 발급**(이메일 회신으로 수령). 키 수령 전에는 실연동 불가.

> **발급 경로 요약**: 직링크 없음(셀프서비스 콘솔 화면 부재) → "문서 포털(etapi.gmarket.com) 안내 확인 → etapihelp@gmail.com 이메일 신청 → 심사 통과 시 Secret Key 이메일 수령"이 공식 경로다. ESM+ 콘솔 내 키 발급 전용 메뉴·버튼은 공식 문서에 존재하지 않는다.

출처: https://etapi.gmarket.com/pages/API-%EA%B0%80%EC%9D%B4%EB%93%9C , https://etapi.ebaykorea.com/api-start/certification-guide/

## 전제조건

- **판매자 가입**: 지마켓·옥션 판매자 계정 필요.
- **ESM+ Master ID**: ESM+ 로그인 후 Master ID 생성 선결.
- **이메일 심사 통과**: etapihelp@gmail.com 신청 → 수동 심사·발급(셀프서비스 아님).
- **셀링툴(솔루션) 업체**: 업체 사업자정보로 사이트별 판매자 가입까지 요구됨.
- 발신번호 사전등록 등 별도 통신 전제: **없음**(해당 없음).

## 스코프 / 권한

- 문서상 **명문화된 스코프 모델 없음**. 실제 접근 범위 = 사용 신청 시 부여된 범위에 따름.
- 권한 부족 시 `401 "The user does not have right access to the api"` 반환 → "신청 API 범위/자격증명 확인" 안내로 처리.
- 출처: https://etapi.gmarket.com/pages/API-%EA%B0%80%EC%9D%B4%EB%93%9C
- **미확인**: 신청 범위 ↔ 부여 권한의 정확한 매핑 표는 공식 문서에 명문 없음(미확인).

## 환경변수 매핑

| 변수명 | 값을 어디서 | 필수 |
|--------|-------------|------|
| `ESM_SECRET_KEY` | 이메일 심사 통과 후 **수동 발급된 Secret Key**(JWT HS256 서명 키). 하드코딩 금지 | 필수 |
| `ESM_MASTER_ID` | ESM+ 로그인 후 생성한 **Master ID**(JWT header `kid`). 셀링툴은 업체 마스터ID | 필수 |
| `ESM_AUCTION_SELLER_ID` | 옥션 **판매자 ID**(`ssi`의 `A:` 값) | 옥션 사용 시 필수 |
| `ESM_GMARKET_SELLER_ID` | 지마켓 **판매자 ID**(`ssi`의 `G:` 값) | 지마켓 사용 시 필수 |
| `ESM_ISS` | JWT `iss`. 기본값 `www.esmplus.com`. 발급 안내와 다르면 override | 옵션(기본값 존재) |

> `ESM_AUCTION_SELLER_ID` / `ESM_GMARKET_SELLER_ID` 중 최소 하나는 설정해야 부팅된다.

## 출처 (공식 URL)

- 문서 포털: https://etapi.gmarket.com (미러: https://etapi.ebaykorea.com)
- API 가이드(인증·JWT 클레임·발급 안내): https://etapi.gmarket.com/pages/API-%EA%B0%80%EC%9D%B4%EB%93%9C
- API 인증가이드(미러): https://etapi.ebaykorea.com/api-start/certification-guide/
- 키 발급 신청처: etapihelp@gmail.com (이메일 심사 — 셀프서비스 발급 페이지/직링크 없음)
- ESM+ 콘솔: https://www.esmplus.com (Master ID 생성용. 키 발급 전용 메뉴 없음)

## 미확인 / 주의

- **토큰 `exp` 정책 미확인**: 가이드에 `exp` 미기재 → 본 서버는 매 요청 새로 서명해 회피.
- **스코프/권한 매핑 미확인**: 명문 스코프 없음. 401 시 안내 메시지로 처리.
- **전체 에러코드 표 미확인**: 확인된 코드만 — 401(권한 없음), 3000(주문조회 5초/1회 초과), 1000(타 판매자 상품), 2141(발송처리 파라미터 오류). 그 외는 일반 실패.
- **주의 — `siteType`/`siteId` 타입이 도구 군별로 다름**: 주문/배송/반품=int(1옥션/2지마켓), 정산=string("A"/"G"), 상품검색 `siteId`=string("1"/"2"), `ssi` 클레임=문자(A/G). 단일 enum 재사용 금지.
- **주의 — `aud` vs 호출 호스트**: `aud`=`sa.esmplus.com`이지만 실제 호출은 `sa2.esmplus.com`. 혼동 금지.
- **발급 위치 표기(확인됨)**: 신청은 이메일(etapihelp@gmail.com) 심사 방식이며, 발급된 Secret Key는 **이메일 회신으로 수령**한다. 공식 가이드(etapi.gmarket.com / etapi.ebaykorea.com) 확인 결과 **셀프서비스 발급 페이지·직링크·ESM+ 콘솔 내 키 발급 전용 메뉴는 존재하지 않음**(2026-06 재검증). ESM+ 콘솔은 Master ID 생성 용도이며 키 발급 UI는 제공하지 않는다.
