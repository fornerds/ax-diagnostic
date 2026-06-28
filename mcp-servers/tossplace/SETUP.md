# 키 발급 가이드 — 토스플레이스(POS) (tossplace)

> **주의: 아래 발급 경로(메뉴/버튼)는 미확인 — 정확하지 않을 수 있습니다.** 발급 콘솔이 로그인 게이트/계정별이라 외부에서 화면을 확인하지 못했습니다. 실제 UI와 다를 수 있으니 공식 콘솔에서 직접 확인하세요. (추적: `mcp-servers/_SETUP_GAPS.md`)

> 근거: `_workspace/tossplace/02_validation.md`(검증일 2026-06-26) + `mcp-servers/tossplace/README.md`. 출처 URL이 있는 사실만 기재한다. 발급 디테일이 문서에 없는 항목은 "미확인"으로 남긴다.

## 연동 유형 / 인증 방식
- 연동 유형: 공개 REST(Open API) 직접 호출. 공식 원격 MCP 커넥터 없음 → 로컬 stdio MCP 서버로 구현.
- 인증 방식: **API Key Pair**(OAuth 아님, 토큰 교환 단계 없음). 정적 키 페어를 env로 주입.
- 모든 요청 헤더:
  - `x-access-key: <Access Key>`
  - `x-secret-key: <Access Secret>`
  - `Content-Type: application/json`
- 베이스 URL: `https://open-api.tossplace.com/api-public/openapi/v1` (버전 `v1` 고정, 폐기/이동 없음)
- 출처: https://docs.tossplace.com/reference/open-api/common.html

## 발급 절차 (스텝)
1. 개발자센터에 접속한다. 콘솔: `https://developers.tossplace.com/`
   - (주의) 위 콘솔 URL은 구현 README 기준이며, 발급 **절차** 자체는 아래 개발 튜토리얼 문서로 검증됨. 콘솔 진입 URL의 공식 문서 직접 출처는 미확인 — 아래 "미확인 / 주의" 참조.
2. **API 타입 앱**을 생성한다 → 앱에서 **Access Key / Access Secret**을 확보한다.
   - **Access Secret은 생성 시에만 확인 가능**하다. 즉시 안전한 곳에 저장한다(재확인 불가).
3. **테스트 가맹점**에 앱을 연결한다(테스트 가맹점 상세에서 앱 **사용여부 ON**). 권한은 매장 단위로 결정된다.
4. 대상 가맹점의 `merchantId`(매장고유번호로 추정 — 아래 미확인 R5)를 확보한다.
5. (선택) 웹훅이 필요하면 개발자센터 내 애플리케이션 → 웹훅에서 셀프 등록한다. 서명 헤더 `x-toss-signature`(값 `v1=`+hex(HMAC-SHA256)).
- 출처(절차·시크릿 1회성·앱-가맹점 연동): https://docs.tossplace.com/guide/pos-integration/open-api/develop-tutorial.html
- 출처(웹훅): https://docs.tossplace.com/reference/open-api/webhook.html

## 전제조건
- 테스트 가맹점 연결까지: 공식 문서로 확인되는 한 별도 사업자 심사·플랜·발신번호 사전등록 요건 **명시 없음**.
- **운영(프로덕션) 사용**: 운영 키 발급에 사업자 심사/제휴 승인이 필요한지 공식 문서에 **명시 없음(미확인, R4)**. 단, 쓰기 API에 명시적 승인 게이트가 있어 운영 사용 시 제휴/승인이 사실상 전제일 가능성. 운영 전 토스 B2B 문의로 확인 권장: https://tossplace.com/contact/b2b
- **쓰기 API 승인 게이트(R1)**: 주문 생성/주문 상태변경(취소)은 "추후 공지가 있을 때까지 승인을 받은 앱에서만 사용 가능". 본 MCP 서버는 읽기 전용이라 직접 영향 없음(쓰기 도구 미구현).

## 스코프 / 권한
- OAuth식 scope 개념 없음. 접근 범위는 **앱-가맹점 연동 + 매장 단위**로 결정된다.
- 본 커넥터는 안정 등급 + 승인 불필요 + 검증 완료된 **읽기 14개 도구만** 사용(주문/결제/매장/카탈로그 조회). 쓰기·ALPHA(주문 생성·취소, 결제취소 기록, 적립 추가)는 의도적으로 제외.
- 출처: https://docs.tossplace.com/reference/open-api/common.html

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|---|---|---|
| `TOSSPLACE_ACCESS_KEY` | 개발자센터 API 타입 앱의 **Access Key** (헤더 `x-access-key`) | 필수 |
| `TOSSPLACE_SECRET_KEY` | 앱 **Access Secret** (헤더 `x-secret-key`, 생성 시 1회만 노출 → 즉시 저장) | 필수 |
| `TOSSPLACE_MERCHANT_ID` | 대상 가맹점 식별자 `merchantId`(매장고유번호로 추정, 테스트 가맹점 연결 후 확보) | 필수 |
| `TOSSPLACE_BASE_URL` | 베이스 URL 오버라이드(기본 `https://open-api.tossplace.com/api-public/openapi/v1`) | 선택 |

## 출처 (공식 URL)
- 공통(인증·베이스URL·레이트리밋·페이지네이션·에러): https://docs.tossplace.com/reference/open-api/common.html
- 키 발급·시크릿 1회성·앱-가맹점 연동: https://docs.tossplace.com/guide/pos-integration/open-api/develop-tutorial.html
- 시작/소개: https://docs.tossplace.com/guide/pos-integration/getting-started.html
- 웹훅: https://docs.tossplace.com/reference/open-api/webhook.html
- 운영 제휴/문의(B2B): https://tossplace.com/contact/b2b
- 개발자센터 콘솔(구현 README 기준): https://developers.tossplace.com/

## 미확인 / 주의
- **개발자센터 콘솔 진입 URL**: `https://developers.tossplace.com/`는 구현 README에 기재된 값으로, 공식 문서의 직접 출처로 1:1 확인되지는 않음. 절차(앱 생성→키 확보)는 develop-tutorial.html로 검증됨.
- **운영 발급 요건(R4)**: 운영 키 발급에 사업자 심사/제휴 승인이 필요한지 문서 미명시. B2B 문의로 사전 확인 권장.
- **merchantId 식별자(R5)**: `merchantId` ≡ `매장고유번호` 동의어 명시 문장 없음. 첫 호출(예: 매장정보 조회) 실패 시 식별자 형식 점검 필요.
- **Access Secret 1회성**: 분실 시 재확인 불가 — 재발급 절차는 문서에서 별도 확인 필요(미확인).
