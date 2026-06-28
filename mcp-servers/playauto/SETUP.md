# 키 발급 가이드 — 플레이오토 (playauto)

> 출처: `_workspace/playauto/02_validation.md`(검증일 2026-06-26) 및 `README.md`. 추측 금지 — 아래는 출처 URL이 있는 사실만 옮겼고, 공식 문서로 확정 못 한 값은 "미확인"으로 표시했다.

## 연동 유형 / 인증 방식

- **연동 유형:** REST OPEN API (PLTO 2.0 / PLAYAUTO 2.0 OPEN API). 공식 원격 MCP 커넥터 없음 → 로컬 MCP 서버(build)로 연동.
- **인증 방식:** API Key 헤더 방식 (OAuth2 아님 — 토큰 발급 전용 엔드포인트 없음, 관리자 콘솔에서 인증키/토큰 직접 발급). 자동 리프레시 흐름 없음(만료 시 수동 재발급).
- **권한:** 인증키 발급/조회는 **마스터계정 전용**(하위계정 불가). (출처: plto.com 13304)
- **제공 범위:** 공식상 **조회(read) API만 제공**. 작업(write)API는 별도 고객센터 신청 대상. (출처: plto.com additional/API)

## 발급 절차 (스텝)

1. **플랜·부가서비스 준비:** PLTO 2.0 **BASIC version 이상** 플랜에서 **"API 연동" 부가서비스**(월 ₩50,000, VAT 별도)를 구매한다. 구매 경로(콘솔 메뉴): **결제 > [이용료 결제] > 구독형 부가서비스 > [API 연동] 추가**. (출처: https://www.plto.com/additional/API/ , https://www.plto.com/customer/HelpDesc/gmp/13711/)
2. **개발자센터 신청 + 승인 대기:** 개발자센터(**직링크: https://developers.playauto.io/**)에 **플토2.0 접속정보로 로그인** → **[사용신청] 버튼** 클릭 → **약관동의** → **사용자구분·신청정보 작성** 후 신청한다. **2~5일 정도 승인 심사**가 소요될 수 있다. 승인 후 **개발자센터 > [API 문서]** 메뉴에서 OPEN API 매뉴얼을 열람할 수 있다. (출처: https://www.plto.com/customer/HelpDesc/gmp/13711/)
3. **인증키 발급(마스터계정):** **2.0 솔루션에 마스터계정으로 로그인** → **설정 > 환경설정 > API 사용설정** 메뉴 → **[솔루션 인증키]** 항목에서 인증키를 발급/조회한다. (제휴 연동 대행사 경유 시엔 같은 메뉴에서 **"API 대행사 설정"을 '사용함'으로 지정** 후 인증키 확인.) — **발급 페이지 직링크 없음**(인증키 화면은 워크스페이스(고객사)별 솔루션 콘솔 내부라 공용 URL이 없다. 위 메뉴 경로로만 도달). (출처: https://www.plto.com/customer/HelpDesc/gmp/13304/ , https://www.plto.com/customer/HelpDesc/gmp/13711/)
4. **토큰 확보(헤더 구성에 따라):** 필요 시 JWT 토큰을 직접 발급한다. (2키 체계일 경우 Secret/Secure Key) — **헤더 구성·토큰 발급 위치는 미확인**(아래 미확인 참조).
5. **환경변수 주입:** 발급한 키/토큰을 아래 [환경변수 매핑]대로 env로 주입한다. 실값은 파일에 쓰지 말고 환경변수 확장(`${VAR}`)을 쓴다.

## 전제조건

- **플랜:** PLTO 2.0 **BASIC version 이상** 필수. (출처: plto.com additional/API)
- **부가서비스:** "API 연동" 부가서비스 구매(월 ₩50,000, VAT 별도). 구매 경로: 결제 > [이용료 결제] > 구독형 부가서비스 > [API 연동] 추가. (출처: plto.com additional/API, 13711)
- **승인 심사:** 개발자센터 신청 후 **2~5일 승인 심사**. (출처: plto.com 13711)
- **계정 권한:** 인증키 발급은 **마스터계정**에서만 가능. (출처: plto.com 13304)
- **작업카운트(과금 함정):** API 작업(주문 수집·상품 전송 등) 시 보유 "작업카운트"가 차감되며 별도 구매 대상. **MCP 도구 호출 = 비용 발생** 가능. (출처: plto.com additional/API)
- 사업자 심사·발신번호 사전등록: **해당 없음**(공식 문서상 별도 요건 언급 없음).

## 스코프 / 권한

- 명시적 OAuth scope **없음**(미확인). "API 사용 설정"에서 일괄 허용으로 추정. (출처: plto.com 13304 — scope 언급 없음)
- 표준 제공 범위는 **조회(read) 전용**. 쓰기(작업API: 주문 등록/상품 등록·수정/재고 갱신/송장 등록/상태 변경)는 고객센터 별도 신청 없이는 호출 불가 → MCP 표준 서버에서 제외됨. (출처: plto.com additional/API)

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `PLAYAUTO_API_KEY` | 마스터계정 → 설정 > 환경설정 > API 사용설정에서 발급한 **솔루션 인증키**(x-api-key / APIKey 값) | 예 |
| `PLAYAUTO_AUTH_TOKEN` | JWT 토큰(`Authorization: Token <…>` 값). 2키 체계일 경우 Secret/Secure Key | 조건부(헤더 구성에 따라) |
| `PLAYAUTO_BASE_URL` | 베이스 URL. 기본 `https://openapi.playauto.io` (**미검증** 기본값) | 권장(기본값 존재) |
| `PLAYAUTO_AUTH_MODE` | 인증 헤더 구성: `tokenA`(기본) 또는 `twokey` | 아니오 |
| `PLAYAUTO_ORDERS_PATH` | 주문 조회 경로. 기본 `/api/orders`(**미검증**) | 아니오 |
| `PLAYAUTO_RATE_DELAY_MS` | 호출 간 지연(ms). 기본 350(**미검증**) | 아니오 |
| `PLAYAUTO_MAX_PAGE_SIZE` | 페이지당 건수. 기본 1000(최대 3000 **미검증**) | 아니오 |

### 인증 헤더 모드 (미검증 — 구현에서 설정 가능)
- `tokenA`(기본, 보조출처): `x-api-key: <PLAYAUTO_API_KEY>` + `Authorization: Token <PLAYAUTO_AUTH_TOKEN>`
- `twokey`(셀링콕/푸시웹 정황): `APIKey: <PLAYAUTO_API_KEY>` + `SecretKey: <PLAYAUTO_AUTH_TOKEN>`
- 401/403이면 두 모드를 바꿔가며 시도하고 부가서비스·승인 상태도 확인한다.

## 출처 (공식 URL)

- API 연동 부가서비스(플랜 요건·가격·조회 전용·작업카운트): https://www.plto.com/additional/API/
- 개발자센터 신청([사용신청] 버튼 → 약관동의 → 신청정보)·2~5일 승인 심사·API 문서 열람·부가서비스 구매 경로(결제>이용료 결제>구독형 부가서비스>[API 연동]): https://www.plto.com/customer/HelpDesc/gmp/13711/
- 인증키 발급 위치(마스터계정 → 설정 > 환경설정 > API 사용설정 > [솔루션 인증키]; 대행사 경유 시 "API 대행사 설정" 사용함): https://www.plto.com/customer/HelpDesc/gmp/13304/
- 개발자센터(로그인/승인 게이트, 신청 직링크): https://developers.playauto.io/
- 보조 출처(헤더·경로·레이트리밋 — 비공식 1곳): https://ecomautonote.com/플레이오토-api-실전-연동-주문-수집부터-커스텀-주문/

## 미확인 / 주의

- **인증 헤더 원문 미확인:** 헤더명(`x-api-key` vs `APIKey`)·토큰 prefix(`Token`)·sol_no 동반 여부·2키(Secret/Secure) 여부가 공식 문서로 확정 불가(개발자센터 로그인/승인 게이트). 보조출처 1곳만 단일 `x-api-key`+JWT를 시사하고, 셀링콕/푸시웹 안내는 2키 체계를 시사 → 구현은 `PLAYAUTO_AUTH_MODE`로 전환 가능하게 함.
- **JWT 토큰 발급 위치·만료 정책 미확인:** 토큰을 콘솔 어디서 발급하는지, 만료 주기·재발급 절차가 공식 미확인. 자동 리프레시 정황 없음 → 만료 시 콘솔에서 수동 재발급 후 `PLAYAUTO_AUTH_TOKEN` 갱신 필요.
- **베이스 URL 미검증:** `https://openapi.playauto.io`는 도메인 실재(403 반환)는 확인됐으나 공식 문서로 베이스 URL 확정 불가. 루트는 HTTP 403. 코드 하드코딩 금지 → env로 외부화.
- **버전 경로 미확인:** `/v1`·`/v2` 같은 경로 버저닝 미확인 → 경로에 버전 세그먼트를 가정하지 말 것.
- **엔드포인트·파라미터·레이트리밋 수치 미검증:** 전부 보조출처 1곳에서만 확인. 런타임 가드 전제.
- **OAuth scope 미확인:** 명시적 scope 흐름 없음(부재 정황). 단정 보류.
- **작업카운트 과금 주의:** 각 호출이 작업카운트를 차감할 수 있음 → 무분별한 반복 호출 시 비용 발생.
- **추가 확인 경로:** 위 미확인 항목(헤더 원문·베이스URL·경로·레이트리밋·에러코드·작업API 신청 범위)은 모두 로그인/승인 게이트 뒤에 있어, 고객사의 개발자센터 승인 계정 협조 없이는 추가 확인 불가.
