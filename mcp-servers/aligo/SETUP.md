# 키 발급 가이드 — 알리고 (aligo)

> 출처 기준: `_workspace/aligo/02_validation.md`(검증일 2026-06-26) + 알리고 공식 문서. 출처 URL 없는 사실은 적지 않음. 발급 디테일이 불명확한 부분은 "미확인 / 주의"에 명시.

## 연동 유형 / 인증 방식
- **연동 유형**: 공식 원격 MCP 커넥터 없음 → 로컬 stdio MCP 서버(build).
- **인증 방식**: **API Key (정적 키)**. OAuth/세션이 아니며 토큰 교환·만료가 없다. 발급된 키를 매 요청 POST 본문(`application/x-www-form-urlencoded`)에 동봉한다.
- **인증 파라미터가 문자/카카오로 다름**:
  - 문자(SMS/LMS/MMS) API: 본문에 `key`(API Key) + `user_id`(회원 ID).
  - 카카오 알림톡 API: 본문에 `apikey`(API Key) + `userid`(회원 ID).
- 출처: https://smartsms.aligo.in/admin/api/spec.html , https://smartsms.aligo.in/smsapi.html , https://smartsms.aligo.in/alimapi.html

## 발급 절차 (스텝)
1. **회원가입** — 알리고(https://smartsms.aligo.in) 가입. 문자 API는 **"별도의 신청절차 없이 회원가입만 하면 바로 연동"** 가능(SMS 심사 불필요). 단, 사업자등록이 필요(개인사업자/법인, 사업자등록증 제출). 출처: https://smartsms.aligo.in/smsapi.html , https://docs.acaflow.co.kr/aligo-service-settings
2. **API 발급/인증 페이지로 이동** — 로그인 후 아래 둘 중 하나로 동일 화면에 도달한다.
   - **직링크**: **`https://smartsms.aligo.in/admin/api/auth.html`** (로그인 상태에서 바로 진입)
   - **메뉴 경로**: 알리고 상단 메뉴 **`문자API`** → 하위 탭 **`신청/인증`** (이 탭이 곧 /admin/api/auth.html). 상단 `문자API` 하위 탭 구성은 `소개` / `신청/인증` / `API SPEC` / `예제 다운로드`.
   - 그 외 진입 버튼: 문자API 소개(info.html)의 **`문자API 신청하기`** 버튼, smsapi.html의 **`지금바로 서버 연동하기`** 버튼이 모두 이 auth.html로 연결된다.
   - 출처: https://smartsms.aligo.in/smsapi.html (`지금바로 서버 연동하기`→/admin/api/auth.html 링크 확인), https://smartsms.aligo.in/admin/api/info.html (`문자API` 상단 메뉴 + `문자API 신청하기` 버튼 + `소개/신청·인증/API SPEC/예제 다운로드` 탭 확인)
3. **API Key 발급/확인** — `신청/인증`(auth.html) 화면에서 **`API KEY 발급신청`** 버튼을 클릭하면 즉시 사용 가능한 API Key가 발급·표시된다. 이미 발급한 경우 같은 화면에서 키를 재확인한다. 발급된 키는 가입 시 사용한 **회원 ID(user_id)**와 함께 보관한다. 출처: https://manual.themango.co.kr/aligo_api (버튼명 `API KEY 발급신청`·즉시발급·진입 URL 확인)
4. **(권장) 발송 서버 IP 등록** — 동일 `신청/인증`(auth.html) 화면의 **`발송 서버 IP 추가`** 항목에서 발송 서버 IP를 등록한다. 복수의 외부 연동 가이드가 "발송 서버 IP를 등록해야 정상 사용 가능"으로 안내하므로 발급 직후 함께 처리할 것. 단 알리고 공식 SPEC 본문에는 강제 여부가 명시되지 않음(아래 "미확인" 참조). 출처: https://manual.themango.co.kr/aligo_api (`발송 서버 IP 추가` 항목 확인), https://docs.acaflow.co.kr/aligo-service-settings
5. **카카오 알림톡 키** — 카카오 API 인증값(`apikey`/`userid`)도 같은 알리고 계정 체계에서 확보한다. 문자 키와 동일할 수 있으나, 본 구현은 분리 변수로 관리한다(아래 환경변수 매핑). 출처: https://smartsms.aligo.in/alimapi.html
6. **발신번호 사전등록** — 알리고 사이트에서 발신번호를 미리 등록한다(전제조건 참조). 미등록 번호로는 발송 불가.

## 전제조건
- **발신번호 사전등록 (필수, 전기통신사업법)** — "발신번호는 사이트내에서 미리 등록된 번호만 사용하실 수 있습니다." 미등록 번호로는 발송이 런타임 실패한다. 출처: https://smartsms.aligo.in/smsapi.html
- **선불 충전 (발송 시 필수)** — 선불 충전형 과금. 잔여 건수가 없으면 발송 실패. 발송 전 `get_sms_remain`(`/remain/`) / `get_alimtalk_remain`(`/akv10/heartinfo/`)로 확인. 출처: https://smartsms.aligo.in/admin/api/spec.html , https://smartsms.aligo.in/alimapi.html
- **문자(SMS) API 심사: 없음** — 회원가입만으로 즉시 연동(별도 신청절차 불필요). 출처: https://smartsms.aligo.in/smsapi.html
- **카카오 알림톡 전제 (무거움)** — 카카오 비즈니스 채널 개설 → 채널 인증(`profile/auth`) → 발신프로필(`senderkey`) → 템플릿 등록·검수 승인(약 4~5영업일). 승인된 `senderkey`·`tpl_code`를 사전 확보해야 `send_alimtalk`이 동작한다(`list_alimtalk_profiles`/`list_alimtalk_templates`로 조회). 출처: https://smartsms.aligo.in/friendapi.html , https://smartsms.aligo.in/alimapi.html

## 스코프 / 권한
- **스코프 체계 없음.** key + user_id(카카오는 apikey + userid) 1쌍으로 발송·조회 전 엔드포인트를 호출한다. OAuth 스코프/권한 구분이 없다. 출처: https://smartsms.aligo.in/admin/api/spec.html

## 환경변수 매핑
| 변수 | 값을 어디서 | 전송 시 파라미터 | 필수 |
|------|-------------|------------------|------|
| `ALIGO_API_KEY` | 로그인 후 `문자API > 신청/인증`(https://smartsms.aligo.in/admin/api/auth.html)의 `API KEY 발급신청`으로 발급한 문자 API Key | `key` | O (문자 도구 사용 시) |
| `ALIGO_USER_ID` | 알리고 가입 회원 ID | `user_id` | O (문자 도구 사용 시) |
| `ALIGO_KAKAO_API_KEY` | 카카오 API 인증 키(문자 키와 같을 수 있으나 분리 관리) | `apikey` | O (알림톡 도구 사용 시) |
| `ALIGO_KAKAO_USER_ID` | 알리고/카카오 연동 회원 ID | `userid` | O (알림톡 도구 사용 시) |
| `ALIGO_DEFAULT_SENDER` | 사이트에 사전등록한 발신번호 | `sender`(미지정 시 기본값) | X (권장) |

> 부가(선택, 기본값 있음): `ALIGO_SMS_BASE_URL`(기본 `https://apis.aligo.in`), `ALIGO_KAKAO_BASE_URL`(기본 `https://kakaoapi.aligo.in`). 키 하드코딩 금지 — 시크릿은 전부 env. 문자만 쓰면 카카오 변수는 비워둬도 동작(도구별 인증 가드).

## 출처 (공식 URL)
- 문자 API 안내·키 발급/심사 불필요·발신번호 등록: https://smartsms.aligo.in/smsapi.html
- 문자API 소개·"문자API 신청하기" 진입 버튼: https://smartsms.aligo.in/admin/api/info.html
- API 연동/인증(키 발급·확인, `API KEY 발급신청`·`발송 서버 IP 추가` 항목) 페이지 = `문자API > 신청/인증` 탭: https://smartsms.aligo.in/admin/api/auth.html (로그인 필요)
- 문자 API SPEC(인증 파라미터·스코프 부재·과금): https://smartsms.aligo.in/admin/api/spec.html
- 카카오 알림톡 API(인증 파라미터·전제): https://smartsms.aligo.in/alimapi.html
- 친구톡/채널·템플릿 검수 안내: https://smartsms.aligo.in/friendapi.html
- 발급 절차·버튼명·IP 등록(외부 통합 가이드, 메뉴/버튼명 교차확인): https://manual.themango.co.kr/aligo_api , https://docs.acaflow.co.kr/aligo-service-settings

## 미확인 / 주의
- **로그인 후 키 발급 경로 확인됨** — 메뉴 경로 `문자API > 신청/인증`(=https://smartsms.aligo.in/admin/api/auth.html)에서 `API KEY 발급신청` 버튼으로 발급, `발송 서버 IP 추가` 항목으로 IP 등록(2026-06 재확인). 단 auth.html은 **로그인 게이트**라 비로그인 상태에서는 로그인 폼만 노출되어 발급 후 키가 표시되는 정확한 필드 레이아웃까지는 직접 캡처 확인 불가(메뉴·버튼명은 공식 info.html 및 더망고 매뉴얼로 교차확인). 화면에서 못 찾으면 알리고 고객센터(평일 10:00~17:00, 02-511-4560, cs@alipeople.kr)에 문의. 출처: https://smartsms.aligo.in/admin/api/info.html , https://manual.themango.co.kr/aligo_api
- **IP 화이트리스트 강제 여부 미확인** — 공식 문서(SPEC/데모 SPEC/smsapi)에 IP 제한·허용 IP 등록 언급 없음. 강제가 아닐 가능성이 높으나 공식 확정 불가. 출처: https://smartsms.aligo.in/admin/api/spec.html
- **TPS/RPM(레이트리밋) 수치 미공개** — 공식 SPEC·데모·FAQ 어디에도 없음. 대량/반복 발송 시 사업자 측 차단 가능성 → 호출 간 백오프 권장. 출처: https://smartsms.aligo.in/admin/api/spec.html
- **카카오 키가 문자 키와 동일한지 여부 불명확** — 분리 변수로 관리. 두 키가 같다면 동일 값을 양쪽 변수에 넣어도 무방.
- **응답 코드 전체표 미공개** — 알려진 코드만 확인: `-101` 인증오류(키/ID 확인), `-804` 예약취소 5분 제한. 인증 실패 시 `-101`로 키·user_id를 점검한다.
- **테스트는 별도 URL 없음** — 운영 URL + 테스트 플래그(`testmode_yn=Y` 문자 / `testMode=Y` 카카오)로 과금·실발송 없이 검증. 출처: https://smartsms.aligo.in/admin/api/spec.html
