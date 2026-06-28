# 키 발급 가이드 — 바로빌 (barobill)

> **주의: 아래 발급 경로(메뉴/버튼)는 미확인 — 정확하지 않을 수 있습니다.** 발급 콘솔이 로그인 게이트/계정별이라 외부에서 화면을 확인하지 못했습니다. 실제 UI와 다를 수 있으니 공식 콘솔에서 직접 확인하세요. (추적: `mcp-servers/_SETUP_GAPS.md`)

> 검증 기준: `_workspace/barobill/02_validation.md`(2026-06-26) + 공식 개발자센터 `dev.barobill.co.kr` 1회 재확인(2026-06-27).
> 원칙: 출처 있는 사실만. 발급 절차의 일부 디테일(운영 키 전환 정확 요건 등)은 공개 문서로 확인 불가 → "미확인"으로 명시.

## 연동 유형 / 인증 방식

- **연동 유형:** 공식 원격 MCP 커넥터 없음 → 로컬 stdio MCP 서버. 백엔드는 SOAP 1.1(ASMX) API.
- **인증 방식:** `CERTKEY`(연동 인증키, 정적 시크릿) 기반. OAuth/토큰 교환·리프레시 **없음**. scope 개념 **없음** — 키 1개로 가입 계정 권한 내 호출.
- **호출 시 본문 전달:** 모든 메서드의 1·2번 파라미터로 `CERTKEY` + `CorpNum`(호출 주체 사업자번호, '-' 없는 10자리). 일부 메서드는 추가로 `UserID`(바로빌 회원 담당자 ID) 필수.
- **호스트 분기:** 운영 `https://ws.baroservice.com`, 테스트(샌드박스) `https://testws.baroservice.com`. **SOAP 네임스페이스/SOAPAction은 두 환경 모두 `http://ws.baroservice.com/` 로 동일**(호스트만 다름).
- 출처: https://dev.barobill.co.kr , https://ws.baroservice.com/CORPSTATE.asmx?op=GetCorpStateEx

## 발급 절차 (스텝)

1. **회원가입 (직링크):** https://dev.barobill.co.kr/accounts/signup — 이메일 또는 소셜 계정으로 가입. (로그인 직링크: https://dev.barobill.co.kr/accounts/login)
2. 가입(파트너 등록) 완료 시 **API 키(CERTKEY)가 자동 발급**된다("간단한 회원가입 후, API 키를 자동 발급합니다"). 테스트키는 가입 즉시 무료이며, 파트너 등록 시 **테스트포인트 10,000P가 자동 지급**된다.
3. **대시보드 확인 (직링크):** 로그인 후 https://dev.barobill.co.kr/cs/dashboard 에서 발급된 CERTKEY를 확인한다.
   - **메뉴 경로/버튼명은 미확인:** 대시보드 내 CERTKEY의 정확한 화면 위치·메뉴 라벨·복사 버튼명은 로그인 영역(공개 문서 미노출)이라 확인 불가 → 로그인 후 직접 확인 필요. (확인 어려울 시 1:1 기술지원 070-4040-5617 / https://dev.barobill.co.kr/cs/inquiry/technical 문의)
4. 테스트 호출은 `testws.baroservice.com` 호스트로 수행(샌드박스 운영 파리티 확인됨).
5. 운영(실발행/실발송) 전환: 충전식 과금 + 운영 키 전환 절차가 있음. 공식 안내상 **"테스트용 연동인증키를 운영용 연동인증키로 교체"** 하는 방식으로, 별도 구조 변경 없이 전환된다고 명시. 단 **정확한 심사/충전 요건(최소금액·승인 게이트)은 미확인** → 운영 적용 전 바로빌에 별도 확인.

- 출처: https://dev.barobill.co.kr , https://dev.barobill.co.kr/accounts/signup , https://dev.barobill.co.kr/cs/dashboard , https://www.barobill.co.kr/partner/info.asp

## 전제조건

연동 자체(키 발급·조회 도구 사용)는 가입 외 별도 전제 없음. 단, **특정 도구 사용 시** 아래 사전 셋업이 필요(모두 미충족 시 음수 에러코드 반환):

- **SMS/LMS 발송(`send_sms`, `send_lms`):** **발신번호 사전등록 필수**. 발송 전 `check_sms_from_number`로 등록 여부 확인. (출처: https://ws.baroservice.com/SMS.asmx?op=CheckSMSFromNumber)
- **카드 내역 조회(`list_card_logs`):** 바로빌 콘솔에서 대상 카드 **사전 등록** 필요(등록 시 카드사 WebId/WebPwd 등 기관 로그인 자격 입력). (출처: https://ws.baroservice.com/CARD.asmx?op=RegistCardEx)
- **계좌 내역 조회(`list_bank_account_logs`):** 콘솔에서 대상 계좌 **사전 등록** 필요(은행 로그인 자격 입력). (출처: https://ws.baroservice.com/BANKACCOUNT.asmx?op=GetPeriodBankAccountLogEx2)
- **운영 충전 잔액:** 실발송/실발행은 충전식 과금 — 잔액 부족 시 음수 코드 반환. (출처: https://www.barobill.co.kr/partner/info.asp)
- **공동인증서 등록(발행 계열, 2차 범위):** 세금계산서 실발행은 공동인증서 사전 등록 선행(국세청 규정). 현재 서버 1차 범위(조회 중심)에서는 비활성. (출처: https://www.barobill.co.kr/csc/faq_v.asp?DocSEQ=14763)

> 위 카드/계좌/발행 계열은 현재 서버의 **2차 범위**라 도구 미포함. 1차 범위(조회/헬스/유틸/문자발송)만 사용한다면 가입+키 발급(+ SMS 시 발신번호 등록)만으로 동작.

## 스코프 / 권한

- OAuth scope 개념 **없음**. CERTKEY 1개가 가입 계정의 모든 권한을 포괄(서비스별 별도 키 분리 없음).
- 동일 CERTKEY로 TI(세금계산서)·CASHBILL(현금영수증)·CORPSTATE(휴폐업조회)·SMS·CARD·BANKACCOUNT·FAX 서비스 호출.
- 출처: https://dev.barobill.co.kr , https://ws.baroservice.com/CORPSTATE.asmx?op=GetCorpStateEx

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `BAROBILL_CERTKEY` | 개발자센터 가입 시 자동 발급되는 연동 인증키. 대시보드(https://dev.barobill.co.kr/cs/dashboard)에서 확인 | 필수 |
| `BAROBILL_CORP_NUM` | 호출 주체(본인 사업장) 사업자번호, '-' 없는 10자리. 가입 시 등록한 사업자 정보 | 필수 |
| `BAROBILL_USER_ID` | 바로빌 회원 담당자 ID(가입 시 본인이 설정한 회원 ID). 목록/상태 일부 도구(`list_sales_tax_invoices`, `get_cashbill_state`, `list_cashbill_sales`)에서 요구 | 해당 도구 사용 시 필수 |
| `BAROBILL_ENV` | `prod`→ws / `test`(또는 미설정)→testws 호스트 분기. 값은 본인이 직접 지정(발급 정보 아님). 기본 `test` 권장 | 선택 |

## 출처 (공식 URL)

- 개발자센터: https://dev.barobill.co.kr
- 회원가입(직링크): https://dev.barobill.co.kr/accounts/signup
- 로그인(직링크): https://dev.barobill.co.kr/accounts/login
- 대시보드(키 확인, 직링크): https://dev.barobill.co.kr/cs/dashboard
- 1:1 기술지원 문의: https://dev.barobill.co.kr/cs/inquiry/technical (전화 070-4040-5617)
- 파트너/요금 안내: https://www.barobill.co.kr/partner/info.asp
- 공동인증서 등록 FAQ: https://www.barobill.co.kr/csc/faq_v.asp?DocSEQ=14763
- 운영 서비스(예): https://ws.baroservice.com/CORPSTATE.asmx?op=GetCorpStateEx
- 테스트 서비스(예): https://testws.baroservice.com/CORPSTATE.asmx?op=GetCorpStateEx

## 미확인 / 주의

- **대시보드 내 CERTKEY 표시 위치/메뉴 라벨/복사 버튼명**: 대시보드가 로그인 영역이라 공개 문서로 미확인(공식·3rd party 가이드 모두 "가입 시 자동 발급"까지만 기술, 화면 경로 미노출). 로그인 후 직접 확인 필요 — 어려울 시 1:1 기술지원(070-4040-5617) 문의.
- **운영 키 전환 정확 요건**(사업자 심사 절차·충전 최소금액·승인 게이트): 미확인. 테스트키 즉시 무료 발급만 확인됨. 운영 전 바로빌에 별도 문의.
- **레이트리밋 수치**: 공개 문서에 호출 제한 수치 명시 없음(미확인). 사전 가드 없이 호출하되 음수 에러코드 기반 처리.
- **에러코드 표(개별 수치)**: 미확인. 런타임 `GetErrString(CERTKEY, ErrCode)`로 메시지 변환(서버 내장).
- **공식 npm/pip SDK 없음**: 범용 SOAP 또는 수동 envelope 사용. **팝빌(Popbill)과 혼동 금지 — 별개 회사**.
- **보안 주의**: 카드/계좌 등록 시 기관 WebId/WebPwd(민감 자격)가 필요하므로, 등록은 MCP 평문 수집을 피하고 바로빌 콘솔에서 사전 수행 권장(현재 서버는 등록 도구 미포함).
- **호스트 분기 함정**: 테스트 환경(testws)에서도 SOAP 네임스페이스/SOAPAction은 `ws.baroservice.com` 그대로. testws로 네임스페이스를 바꾸지 말 것.
