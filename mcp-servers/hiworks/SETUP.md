# 키 발급 가이드 — 하이웍스 (hiworks)

> 근거: `_workspace/hiworks/02_validation.md`(공식 API 레퍼런스 원본 JSON + 가비아 공식 매뉴얼 본문 대조) 및 `mcp-servers/hiworks/README.md`. 추측 없이 출처 있는 사실만 정리했다. 출처 URL은 맨 아래 참조.

## 연동 유형 / 인증 방식

- **연동 유형**: REST API (공식 원격 MCP 커넥터 없음 → 로컬 MCP 서버로 빌드).
- **인증**: **Office Token (HTTP Bearer) 단일 방식**.
  - 일반 API(`https://api.hiworks.com`): `Authorization: Bearer {OFFICE_TOKEN}`.
  - 세금계산서 API(`https://bill-api.hiworks.com`)는 **별도 이중 인증 헤더**: `Office-Authorization` + `Bill-Access-Authorization` (Bearer 미사용).
- **폐기**: OAuth(Access Token)는 **신규 발급 중단**(공식 원문 재확인). OAuth 흐름·`/user/v2/me`는 사용 금지.
- **토큰 성격**: 슈퍼관리자가 콘솔에서 발급한 **정적 토큰**을 env에 주입한다. OAuth 교환/리프레시 없음.

## 발급 절차 (스텝) — Office Token

1. **하이웍스 오피스(유료 그룹웨어)에 가입**하고 **전체관리자(슈퍼관리자) 계정**으로 로그인한다.
   - 로그인 URL: `login.office.hiworks.com/{회사도메인}`
2. 콘솔에서 다음 경로로 이동한다:
   **오피스관리 > 서비스 연동(API) > 오피스 연동 > 서비스 연동**
3. **[오피스 토큰 생성]** 버튼을 누른다. (별도 사업자/제휴 심사 없이 **즉시 자체 발급**됨.)
4. 토큰 생성 화면에서 **사용할 도구에 대응하는 Scope를 반드시 체크**한다(아래 "스코프/권한" 참조). 허용된 Scope에서만 API 호출이 가능하다.
5. (권장) **API 허용 IP**를 등록한다 — MCP 서버 호스트 IP. 설정된 IP에서만 호출 가능하도록 제한해 보안을 높인다.
6. 생성된 **Office Token 값**을 복사해 env(`HIWORKS_OFFICE_TOKEN`)에 넣는다.
7. 발급/확인은 **[API 관리 > 서비스 연동]** 메뉴에서 다시 확인할 수 있다.

> ⚠️ **토큰 재발급 시 기존에 연동된 API는 더 이상 이용할 수 없다.** 토큰 회전 시점에 주의한다.

### 세금계산서 도구용 토큰 (선택, 2차)

세금계산서 3종(`check_business_status`/`validate_business`/`get_tax_invoice`)은 `bill-api.hiworks.com`의 **이중 인증 헤더**(`Office-Authorization` + `Bill-Access-Authorization`)를 쓴다. **`Bill-Access-Authorization` 토큰의 정확한 발급 절차는 미확인**(아래 "미확인/주의" 참조). 두 값이 모두 설정된 경우에만 해당 도구가 활성화되며, 미설정 시 도구는 안내 메시지를 반환한다(서버는 정상 기동).

## 전제조건

1. **하이웍스 오피스(유료) 가입 + 슈퍼관리자 계정.** 무료/개인 계정은 사용 불가(전자결재·메신저 알림·공용 문자는 오피스 회원만 제공).
2. **Scope 사전 체크.** 토큰 생성 시 사용할 도구의 Scope를 켜지 않으면 런타임 401/403.
3. **SMS 발신번호 사전등록.** `sender`는 사전등록(전기통신사업법 §84-2 발신번호 사전등록제) 완료된 번호만 사용 가능.
4. **알림톡 사전 준비.** 카카오 비즈니스 채널 연동 + **템플릿 사전승인**(이름·제목·본문 일치). 불일치 시 발송 실패. 하이웍스 내 발신프로필 등록 절차 세부는 **미확인**.
5. **(메일 도구) 이메일 Scope 활성화.** 현행 컬렉션은 `sendMail`을 Office Token + "이메일" Scope로 호출. (과거 2019 가이드의 "애플리케이션 등록 + 약 5영업일 사전검수"는 폐기된 OAuth 앱 경로 한정이며, Office Token 경로에는 적용되지 않음.)
6. **사업자/제휴 심사: 기본 발급에는 불필요(없음).** 슈퍼관리자가 콘솔에서 즉시 발급. 제휴 문의는 한도 확대/광범위 연동 시에만 선택적.

## 스코프 / 권한

토큰 생성 시 체크하는 Scope 목록(허용된 Scope에서만 API 호출 가능):

| Scope | 대응 MCP 도구(예) |
|-------|------------------|
| 인사 | `hiworks_list_users`, `hiworks_list_organizations` |
| 이메일 | `hiworks_send_mail` |
| 알림 | `hiworks_send_notification` |
| 전자결재 | `hiworks_create_approval_document`, `hiworks_get_approval_document` |
| 문자 | `hiworks_send_sms`, `hiworks_get_sms_count/result/usage` |
| 카카오 알림톡 | `hiworks_send_alimtalk`, `hiworks_get_alimtalk_balance/result` |
| 전자세금계산서 | `hiworks_check_business_status`, `hiworks_validate_business`, `hiworks_get_tax_invoice` |
| 회계지원 | (1차 계약 제외 — ERP성 회계 도구) |

> **레이트리밋(확정)**: Office Token 이용 시 **오피스당 1,000건/일(성공 기준)**. 토큰을 공유하는 모든 호출이 같은 한도를 사용한다. 초과 시 이용 제한.
> **Office Token = 오피스 전체 권한**이 로컬에 상주한다. 도구 범위(Scope)는 최소한으로만 켤 것.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `HIWORKS_OFFICE_TOKEN` | 콘솔 [오피스 토큰 생성]에서 발급한 Office Token (일반 API Bearer) | 필수 |
| `HIWORKS_API_BASE_URL` | 일반 API 베이스. 기본 `https://api.hiworks.com` (오버라이드용) | 선택 |
| `HIWORKS_DEFAULT_USER_ID` | SMS/메일/알림톡 호출의 `user_id` 기본값(오피스 사용자 아이디) | 선택 |
| `HIWORKS_BILL_OFFICE_AUTHORIZATION` | 세금계산서 API `Office-Authorization` 헤더값 | 세금계산서 도구 사용 시 필수 |
| `HIWORKS_BILL_ACCESS_AUTHORIZATION` | 세금계산서 API `Bill-Access-Authorization` 헤더값 (발급 절차 미확인) | 세금계산서 도구 사용 시 필수 |
| `HIWORKS_BILL_API_BASE_URL` | 세금계산서 베이스. 기본 `https://bill-api.hiworks.com` | 선택 |

> 시크릿은 절대 하드코딩/로그/URL에 노출하지 않는다. `.env`는 커밋하지 않는다.

## 출처 (공식 URL)

- 개발자센터 문서 허브: https://developers.hiworks.com/docs
- 공식 API 레퍼런스(Postman): https://documenter.getpostman.com/view/6863253/S1TVWcri
- 컬렉션 원본 JSON(검증에 사용): https://documenter.getpostman.com/api/collections/6863253/S1TVWcri
- 오피스 토큰 생성/Scope 매뉴얼(가비아 공식): https://customer.gabia.com/manual/hiworks/3403/4280
- 시작 가이드: https://developers.hiworks.com/support/article/1 , /article/2 , /article/3

## 미확인 / 주의

- **`Bill-Access-Authorization`(세금계산서) 발급 절차 미확인.** Office Token과 별개 인증이며, 발급 위치/방법이 공식 근거로 확인되지 않았다 → 세금계산서 3종은 2차로 보류. 두 토큰이 설정된 환경에서만 활성.
- **알림톡 발신프로필 등록 절차 세부 미확인.** 카카오 비즈채널 연동 + 템플릿 사전승인이 필요하다는 사실은 확인되나, 하이웍스 화면 내 등록 절차 단계는 미확인.
- **메일 인증 경로 불일치 주의.** 2019 가이드(애플리케이션 등록 + 5영업일 검수)와 현행 Postman(`Authorization: Bearer {office_token}` + "이메일" Scope)이 불일치. 현행은 Office Token 경로로 보되, 401/403 시 "이메일 Scope/앱 활성화 필요"로 가드.
- **에러 스키마·페이지네이션·`sms_type` enum 미문서화.** 구현은 HTTP status 우선 분기 + 응답 바디 원문 표면화로 처리.
- **토큰 재발급 = 기존 연동 전면 중단.** 재발급은 신중히.
