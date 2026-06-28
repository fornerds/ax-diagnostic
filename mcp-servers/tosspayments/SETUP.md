# 키 발급 가이드 — 토스페이먼츠(PG) (tosspayments)

> 출처: `_workspace/tosspayments/02_validation.md`(공식 문서 1:1 검증본) + 공식 개발자센터 1회 확인(2026-06-27, API Key 메뉴/발급 절차 보강).
> 원칙: 추측 금지. 검증된 사실만 기재하고 출처 URL을 단다. 불명확한 항목은 "미확인"으로 남김.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 MCP 서버(stdio) ↔ 토스페이먼츠 REST API. OAuth 없음, 정적 시크릿 키 방식.
- **인증 방식**: HTTP Basic. 시크릿 키를 username으로, password는 빈 값으로 두고(`{SECRET_KEY}:`) **BOM 없는 UTF-8**로 base64 인코딩한다.
  - 헤더: `Authorization: Basic base64("{TOSS_SECRET_KEY}:")`
  - 토큰 교환·갱신 단계 없음. MCP 서버가 매 요청마다 헤더를 자동 생성한다.
  - 출처: https://docs.tosspayments.com/reference/using-api/authorization
- **사용하는 키**: 서버 REST용 **시크릿 키**(`sk`/`gsk` 계열)만 사용. 클라이언트 키(`ck`/`gck`)는 브라우저 SDK 전용이라 서버 MCP에서 사용하지 않는다.

## 발급 절차 (스텝)

1. **개발자센터 가입·로그인** — https://developers.tosspayments.com/
2. **전자결제 신청** — 신청을 진행한다. (테스트 키는 승인 전에도 일부 제공됨)
3. **상점 선택** — 대시보드에서 발급 대상 상점을 고른다.
4. **API 키 메뉴 진입** — https://developers.tosspayments.com/my/api-keys 에서 키 확인.
5. **키 확인·복사** — 키 쌍(클라이언트 키 + 시크릿 키)이 표시된다. 이 중 **시크릿 키**(`test_sk`/`test_gsk` 또는 `live_sk`/`live_gsk`)를 복사해 환경변수에 넣는다.

- 출처: https://docs.tosspayments.com/reference/using-api/api-keys

## 전제조건

- **테스트 키**(`test_sk`/`test_gsk`): 별도 심사 없이 **즉시 사용 가능**. 개발·데모·통합검증 가능(가상 승인, 실제 출금 없음).
- **라이브(실결제) 키**(`live_sk`/`live_gsk`): 전자결제 신청 → 사업자등록증 제출 → 신청서/계약 심사 → **카드사 심사** 완료 후 발급. 즉 사업자/가맹 계약 없이는 실결제 불가.
- 발신번호 사전등록 등 통신 관련 전제조건: 해당 없음(PG 결제 API).
- 출처: https://docs.tosspayments.com/reference/using-api/api-keys , 교차 확인 https://help.portone.io/content/tosspayments-contract (라이브 키 심사 절차 — 3자 보조 출처)

## 스코프 / 권한

- 별도 OAuth 스코프 개념 없음. **시크릿 키 자체가 전체 권한**(결제 승인·취소·조회·정산·가상계좌·빌링·현금영수증)을 가진다.
- test/live 구분은 키 prefix로 결정된다.
- 시크릿 키 prefix 집합: `test_sk_`, `test_gsk_`, `live_sk_`, `live_gsk_` 모두 유효한 서버 시크릿 키. (`gsk`는 위젯형, `sk`는 개별 API형 — 서버는 둘 다 허용)
- 출처: https://docs.tosspayments.com/reference/using-api/api-keys

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `TOSS_SECRET_KEY` | 개발자센터 API 키 메뉴(https://developers.tosspayments.com/my/api-keys)의 **시크릿 키**(`test_sk`/`test_gsk`/`live_sk`/`live_gsk`) | 필수 |
| `TOSS_API_BASE_URL` | 베이스 URL 오버라이드. 기본값 `https://api.tosspayments.com` — 보통 설정 불필요 | 선택 |
| `TOSS_ALLOW_DESTRUCTIVE` | `true`여야 `cancel_payment`(환불)·`pay_with_billing_key`(실과금)가 실제 호출됨. 미설정/`false`면 안전상 차단 | 선택 |

> 클라이언트 키(`ck`/`gck`)는 서버 MCP에서 사용하지 않으므로 env에 두지 않는다. 시크릿 키 실값을 코드/설정 파일에 쓰지 말고 환경변수로만 주입한다.

## 출처 (공식 URL)

- API 키 발급·종류: https://docs.tosspayments.com/reference/using-api/api-keys
- 개발자센터(가입·로그인): https://developers.tosspayments.com/
- API 키 메뉴(콘솔): https://developers.tosspayments.com/my/api-keys
- 인증(Basic, base64, BOM 금지): https://docs.tosspayments.com/reference/using-api/authorization
- 베이스 URL·API 가이드: https://docs.tosspayments.com/en/api-guide
- (교차 보조) 라이브 키 계약·심사 절차: https://help.portone.io/content/tosspayments-contract

## 미확인 / 주의

- **레이트리밋(요청/초·분, 429 임계치)**: 공식 문서에 명시 수치 **없음 → 미확인**. MCP 서버는 429 응답 감지 시 `Retry-After` 존중 + 지수 백오프(1s→2s→4s, 최대 3회)로 대응한다.
- **웹훅 서명검증·이벤트 종류 상세**: 본 검증 범위 외(수신형 API라 MCP 호출 대상 아님) → 운영 시 별도 확인.
- **라이브 키 심사 절차**의 1차 출처는 공식 페이지 직접 인용이 아니라 PortOne 3자 가이드로 교차 확인. 사실 자체는 일관하나, 정확한 서류·소요기간은 개발자센터/영업 안내를 따른다.
- **보안 주의**: 시크릿 키(`sk`/`gsk`)는 결제·환불 권한을 가진다. 로그·에러 메시지·응답·커밋에 절대 노출 금지. `.env`는 커밋하지 않는다.
- **파괴적 작업**: `cancel_payment`(환불)·`pay_with_billing_key`(실과금)는 되돌릴 수 없는 돈 이동. 라이브 키 사용 시 `TOSS_ALLOW_DESTRUCTIVE` 가드를 신중히 다룬다.
