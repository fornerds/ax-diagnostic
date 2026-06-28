# 키 발급 가이드 — Toss Business (toss_business)

> "Toss Business(토스비즈니스)"는 공개 개발자 API가 없는 영업/온보딩 포털이므로,
> 실제 연동 표면은 **토스페이먼츠(Toss Payments) 코어 API**(`https://api.tosspayments.com/v1`)로 고정한다.
> 따라서 키 발급도 **토스페이먼츠 개발자센터**에서 이뤄진다.
> 검증 출처: `_workspace/toss_business/02_validation.md` (검증 시점 2026-06).

## 연동 유형 / 인증 방식

- 연동 유형: **REST + 시크릿 키 인증** (서버-투-서버). 토큰 교환·OAuth 플로우 없음.
- 인증 방식: **HTTP Basic**.
  - `Authorization: Basic base64("{TOSS_SECRET_KEY}:")`
  - **시크릿 키 뒤에 콜론(`:`)을 반드시 붙인 뒤** base64로 인코딩한다. 비밀번호 자리는 공백.
  - 콜론을 빠트리면 인증이 실패하니 주의.
- 키 1쌍 = 상점(MID) 단위 권한. 세분화된 OAuth scope 개념 없음.
- 출처:
  - https://docs.tosspayments.com/reference/using-api/authorization
  - https://docs.tosspayments.com/reference/using-api/api-keys

## 발급 절차 (스텝)

1. **개발자센터 접속·로그인:** https://developers.tosspayments.com
2. **API 키 메뉴 이동:** https://developers.tosspayments.com/my/api-keys
3. **키 확인·복사:** 해당 메뉴에서 시크릿 키(`test_sk_...` / `live_sk_...`)를 확인하고 복사한다.
   - 서버 MCP는 **시크릿 키**만 필요. 클라이언트 키(`*_ck_*`)는 사용하지 않는다.
4. **환경변수 주입:** 복사한 시크릿 키를 `TOSS_SECRET_KEY` 로 주입한다(아래 환경변수 매핑 참조).

> 데모/PoC: **테스트 키(`test_sk_...`)는 가입·사업자번호 없이 즉시 발급·사용 가능**하다.
> (출처: https://docs.tosspayments.com/blog/how-to-test-toss-payments)
> 단, API 로그·웹훅·가상계좌 모의입금 등 심화 테스트는 이메일/전화 가입 후 체험 상점에서 가능.

## 전제조건

- **테스트 키:** 전제조건 **없음**. 사업자 등록·심사 없이 즉시 사용.
- **라이브 키(`live_sk_...`):** 사업자 **전자결제(PG) 계약·심사**가 선행 전제.
  - 토스 심사 약 1~2일 + 카드사 심사 최대 14일 소요(공식 계약 FAQ 기준).
  - 사업자만 계약 가능. 비사업자/미계약 상태에서는 라이브 호출 불가.
  - 출처: https://toss.oopy.io/78932357-c9c3-4620-bb8e-1f311b2b14cc
- 발신번호 사전등록 등 별도 사전등록 요건: **해당 없음**(SMS/알림 발신 연동이 아님).
- 빌드·데모는 테스트 키로 무관하게 진행 가능.

## 스코프 / 권한

- 별도 OAuth scope 설정 **없음**. 시크릿 키 1쌍이 곧 해당 상점(MID)의 API 권한 전체.
- 테스트 키 ↔ 라이브 키는 환경 구분용. 세트(키/시크릿)를 섞거나 테스트/라이브를 혼용하면 `INVALID_API_KEY`.
- 출처:
  - https://docs.tosspayments.com/reference/using-api/authorization
  - https://docs.tosspayments.com/reference/using-api/api-keys

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `TOSS_SECRET_KEY` | 개발자센터 API 키 메뉴(https://developers.tosspayments.com/my/api-keys)의 **시크릿 키**(`test_sk_...` 또는 `live_sk_...`) | 필수 |
| `TOSS_API_BASE_URL` | 베이스 URL 오버라이드. 미지정 시 기본 `https://api.tosspayments.com` | 선택 |

> 시크릿 키는 외부에 노출되면 안 된다. 코드·공개 저장소·클라이언트 코드·`.mcp.json` 실값에 쓰지 말고,
> `.env` 또는 환경변수 확장(`${TOSS_SECRET_KEY}`)으로 주입한다.
> (출처: https://docs.tosspayments.com/reference/using-api/api-keys)

## 출처 (공식 URL)

- 개발자센터(키 발급 포털): https://developers.tosspayments.com
- API 키 메뉴(시크릿 키 확인/복사): https://developers.tosspayments.com/my/api-keys
- 인증(HTTP Basic·콜론·base64): https://docs.tosspayments.com/reference/using-api/authorization
- API 키 규칙(접두·세트·INVALID_API_KEY): https://docs.tosspayments.com/reference/using-api/api-keys
- 테스트 키 즉시 사용: https://docs.tosspayments.com/blog/how-to-test-toss-payments
- 라이브 키 계약·심사 전제: https://toss.oopy.io/78932357-c9c3-4620-bb8e-1f311b2b14cc
- Toss Business 포털(영업/온보딩): https://business.toss.im/

## 미확인 / 주의

- **레이트리밋(RPS/일 한도):** 공식 문서·FAQ에 수치 명시 **없음(미확인)**. 운영 도입 전 고객사 MID 기준 실측/문의 권장. 구현은 보수적 동시성 + 429/5xx 백오프로 가드.
- **웹훅:** 이벤트 카탈로그·서명 검증 상세 **미확정** → 본 MCP 서버 범위 밖(키 발급과 무관).
- **API 버전:** 코어 API는 날짜기반 버저닝으로 **상점 단위 계약 버전이 개발자센터에서 설정**된다. per-request 필수 버전 헤더는 없음. 운영 시 응답 안정성을 위해 상점 버전 고정 권장.
- **라이브 전환 시점:** 위 라이브 키 전제(전자결제 계약·심사)가 완료되어야 라이브 호출이 가능. PoC 단계에서는 테스트 키로 진행.
