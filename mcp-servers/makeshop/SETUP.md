# 키 발급 가이드 — 메이크샵 (makeshop)

> 대상: 한국 메이크샵(makeshop.co.kr) **Open API** (구 API/API3는 폐기됨, 신규는 Open API만 / 주문은 2.0만).
> 모든 사실은 `_workspace/makeshop/02_validation.md`(검증일 2026-06-26)와 `mcp-servers/makeshop/README.md`의 출처 있는 항목만 사용. 추측 없음.

## 연동 유형 / 인증 방식

- **연동 유형:** REST (조회 GET / 처리 POST). OAuth/토큰교환 **아님** — 정적 API Key형.
- **인증:** 모든 요청에 HTTP 헤더 **2종을 동시** 전달.
  ```
  Shopkey: {MAKESHOP_SHOPKEY}
  Licensekey: {MAKESHOP_LICENSEKEY}
  ```
- **베이스 URL:** 조회 `{상점도메인}/list/open_api.html`, 처리 `{상점도메인}/list/open_api_process.html`.
- 두 키 모두 **상점 단위**로 발급된다(상점 도메인 + Shopkey + Licensekey 세트).

## 발급 절차 (스텝)

키 발급은 **B2B 등록 전제**다(개인 즉시 발급 불가). 운영자 계정 또는 운영자 협조가 필수다.

1. **파트너 포털 라이선스 신청** — `https://partner.makeshop.co.kr/` 에서 **메이크샵 운영자 계정**으로 라이선스 업체 신청. 사업자등록·정산정보를 제출한다. (라이선스 신청 진입점: `https://openapi.makeshop.co.kr/licenses/` → 파트너 포털로 리다이렉트)
   - 파트너 **심사 5~10영업일** 소요.
2. **라이선스 키 발급** — 심사 통과 후 키 발급(약 1~5일).
3. **대상 상점 관리자 화면에서 오픈 API 메뉴 진입** — 메뉴 경로는 관리자 버전/디자인에 따라 **2가지** 존재(둘 다 안내):
   - ① **부가서비스 > 부가서비스 > 오픈 API** (공식 헬프센터 기준)
   - ② **연동관리[상단] > 오픈API[좌측]** (벤더 가이드 기준)
4. **업체 추가 + 권한·동의 처리** — 오픈 API 메뉴에서 라이선스 업체를 **검색·추가**한 뒤:
   - 필요한 **권한 전체 체크**
   - **서비스 이용동의** 체크
   - **제3자 제공 동의** 체크 후 신청
   - (라이선스 업체로 등록되어 있지 않으면 상점 검색 자체가 안 됨)
5. **Shopkey 확인** — 추가된 업체의 **Shopkey [보기]** 로 상점키를 확인한다.
6. **허용 IP 등록** — 라이선스 관리화면에서 호출 서버의 **고정 IP**를 화이트리스트에 등록한다(미등록 시 전 호출이 `9009`로 차단).

> 위 단계 중 하나라도 빠지면 `9001~9005 / 9009 / 9101 / 9102 / 9103 / 9104` 등으로 전 호출이 차단된다.

## 전제조건

- **사업자 심사:** 필요 — 파트너 포털에서 사업자등록·정산정보 제출 후 심사(5~10영업일).
- **플랜:** README/검증본에 특정 플랜 요구 사실 **없음** → "없음(미확인)".
- **발신번호 사전등록:** 해당 없음(SMS 토큰은 미확인으로 계약 제외).
- **운영자 계정:** 필요 — 메이크샵 운영자 계정으로 라이선스 신청·상점 메뉴 동의 수행.
- **고정 IP:** 사실상 필요 — 허용 IP 화이트리스트 기반(동적 IP/서버리스(람다 등)는 부적합, 고정 IP 서버 권장).

## 스코프 / 권한

- 권한은 **키에 종속된 작업별 Permission**이다. 발급(상점 메뉴에서 권한 체크) 시 부여되며, 코드/런타임이 협상하지 않는다.
- 권한·동의 미비 시: `9007`, `9008`, `9101`, `9102`, `9103`, `9104`.
- 검증된 작업 범위: 상품 조회(`type=product`), 주문 2.0 조회(`type=order`), 상품 등록(`type=product&process=store`).
  - 회원/송장/배송상태변경/재고/적립금/쿠폰/게시판/SMS 등 개별 토큰은 **공식 미확인** → 전용 권한 단정 불가(가드형 패스스루로만 접근).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `MAKESHOP_BASE_URL` | 대상 상점 베이스 URL (예: `https://shop.example.com` 또는 `shop.example.com`) | 예 |
| `MAKESHOP_SHOPKEY` | 상점 관리자 오픈 API 메뉴의 **Shopkey [보기]** 값 (HTTP 헤더 `Shopkey`) | 예 |
| `MAKESHOP_LICENSEKEY` | 파트너 포털에서 발급된 **라이선스 키** (HTTP 헤더 `Licensekey`) | 예 |
| `MAKESHOP_FORCE_HTTPS` | `true`(기본)=https만 / `false`=https 실패 시 http 폴백 | 아니오 |
| `MAKESHOP_TIMEOUT_MS` | 요청 타임아웃(ms, 기본 15000) | 아니오 |

> 시크릿은 코드/`.mcp.json`에 실값으로 쓰지 말고 환경변수로 주입. `.env`는 커밋 금지.

## 출처 (공식 URL)

- 공식 가이드: `https://openapi.makeshop.co.kr/` , `https://openapi.makeshop.co.kr/guide`
- 라이선스 신청(→ 파트너 포털 리다이렉트): `https://openapi.makeshop.co.kr/licenses/` → `https://partner.makeshop.co.kr/`
- 헬프센터(메뉴 경로·동의·라이선스 업체 등록): `https://help.makeshop.co.kr/faq/undefined-5/undefined-3/api`
- 구 API/API3 폐기 공지: `https://help.makeshop.co.kr/faq/undefined-5/undefined-3/api-api3`
- 주문 1.0 미제공 공지: `https://www.makeshop.co.kr/newmakeshop/home/notice_view.html?t=note&uid=78`
- (보조 교차검증) 메뉴 ②경로·동의 단계: `https://guide.linkd.kr/start/install/makeshop/step1`

## 미확인 / 주의

- **심사·발급 소요일**: 파트너 심사 5~10영업일 / 라이선스 키 발급 1~5일 — 검색결과 기반 보완치(상황에 따라 변동 가능).
- **메뉴 경로 2종**: 관리자 버전/디자인에 따라 ①부가서비스>오픈API / ②연동관리>오픈API로 갈린다. 실제 화면에서 둘 중 보이는 경로 사용.
- **허용 IP 등록 UI 세부**는 라이선스 관리화면 내에 있으나 벤더 가이드에 단계별 캡처 없음 → 화면에서 확인 필요.
- **HTTPS 강제 여부 미확인**: 공식 예제는 http. 구현은 https 우선 → 실패 시 http 폴백(옵션).
- **샌드박스/테스트 상점 제공 여부 미확인**: 개발 테스트가 실상점 + 발급 키로만 가능할 수 있음.
- **레이트리밋**: 조회 시간당 500회(주문/상품/회원/게시판 **합산**), 처리 권한당 500회. 초과 시 `9006` — 대량 동기화 부적합.
- **개별 작업 토큰(mode/type/process)**: 상품 조회/주문 조회/상품 등록 외 작업은 공식 토큰 미확인 → 발급 권한이 있어도 정확 토큰은 메이크샵 가이드에서 직접 확인해야 함.
