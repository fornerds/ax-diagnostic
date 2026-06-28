# 키 발급 가이드 — 네이버 클라우드 SENS (naver_sens)

> SENS(Simple & Easy Notification Service)는 NCP(NAVER Cloud Platform)의 알림 발송 서비스(SMS/LMS/MMS, 카카오 알림톡/친구톡)다.
> 이 커넥터는 OAuth가 아니라 **NCP API Gateway 공통 인증(IAM Access Key + HMAC 서명)** 으로 동작한다.
> 검증 출처: `_workspace/naver_sens/02_validation.md` (검증일 2026-06-26) + 공식 개발자 문서 1회 보강 확인.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 MCP 서버(stdio) → SENS REST API 직접 호출. (공식 원격 MCP 커넥터·전용 SDK 없음)
- **인증 방식**: NCP API Gateway 공통 인증. **OAuth 아님 / 스코프 개념 없음.**
- **요청마다 헤더 3종**(서버가 자동 부착):
  - `x-ncp-apigw-timestamp`: epoch milliseconds (매 요청 재생성, 서버 허용 오차 ±5분 — 초과 시 401)
  - `x-ncp-iam-access-key`: Access Key
  - `x-ncp-apigw-signature-v2`: `Base64( HMAC_SHA256( secretKey, StringToSign ) )`
    - `StringToSign = "{METHOD} {PATH+QUERY}\n{timestamp}\n{accessKey}"`
    - 서명에 쓴 경로(+쿼리)와 실제 요청 경로가 **동일**해야 함(불일치 시 401).
- 출처: https://api.ncloud-docs.com/docs/en/common-ncpapi

## 발급 절차 (스텝)

발급 대상은 두 가지다 — **(A) API 인증키(Access Key / Secret Key)** 와 **(B) SENS serviceId**.

### A. API 인증키 발급 (Access Key / Secret Key)

1. NAVER Cloud Platform 콘솔(https://console.ncloud.com) 로그인.
2. 우상단 **마이페이지(My Account) > 계정 및 보안 관리** 진입 후 재인증.
3. **보안 관리 > 접근 관리 > API 인증키 관리** 이동.
4. 기본 제공되는 **Access Key ID / Secret Key** 확인. 추가가 필요하면 **[신규 API 인증키 생성]** 클릭(계정당 최대 2개).
5. Access Key → `NCP_ACCESS_KEY`, Secret Key → `NCP_SECRET_KEY` 로 사용.
   - 출처: https://api.ncloud-docs.com/docs/en/common-ncpapi

> 참고: 02_validation은 운영상 **IAM 서브계정** 키 사용을 권장한다(권한 최소화). 서브계정 키 발급/권한 부여 화면은 콘솔의 IAM(Sub Account) 메뉴에서 진행하며, 서브계정에 SENS 사용 권한을 부여해야 한다. (정확한 권한 정책명은 미확인 — 아래 "미확인/주의" 참조.)

### B. SENS serviceId 발급

1. 콘솔 메뉴: **Services > Application Services > Simple & Easy Notification Service > Project** 이동.
2. **[프로젝트 생성하기]** 클릭.
3. 프로젝트 설정 입력:
   - 프로젝트 이름(소문자/숫자/하이픈/언더스코어, 2~24자, 고유)
   - 설명(선택, 128자 이내)
   - **서비스 타입**(SMS / 카카오 알림톡 등) 선택 — 발송에 쓸 채널을 선택.
   - 삭제 보호(선택)
4. **[생성하기]** 클릭. 프로젝트가 생성되며 **서비스 타입별로 OPEN API용 serviceId가 부여**된다.
5. 프로젝트 목록에서 채널별 serviceId를 확인해 환경변수에 매핑:
   - SMS → `SENS_SMS_SERVICE_ID`
   - 알림톡 → `SENS_ALIMTALK_SERVICE_ID`
   - 친구톡 → `SENS_FRIENDTALK_SERVICE_ID`
   - 출처: https://guide.ncloud-docs.com/docs/sens-project

## 전제조건

코드는 위 키만 있으면 빌드·기동되지만, **실제 발송**에는 아래 행정 전제가 필요하다(즉시 발송 불가).

1. **NCP 계정 + 결제수단 등록** — SENS는 발송 건당 과금. 결제수단 미등록 시 사용 불가.
2. **발신번호 사전등록 (영업일 3~4일)** — `from`에는 사전 등록·승인된 번호만 허용. 미등록 번호는 발송 불가. (전기통신사업법) 출처: https://guide.ncloud-docs.com/docs/sens-callingno
3. **카카오 알림톡/친구톡**: 카카오 채널 + 발신프로필 + 템플릿 검수(평균 2~3 영업일) 필요. 검수 완료된 `templateCode`만 알림톡 발송 가능. 출처: https://guide.ncloud-docs.com/docs/sens-sens-1-5
4. **광고성 발송 규제**: `contentType=AD`(SMS)·친구톡(광고성)은 정보통신망법상 수신동의·야간발송 제한·**080 무료수신거부 표기** 의무. 출처: https://guide.ncloud-docs.com/docs/en/sens-smspolicy

## 스코프 / 권한

- **스코프 개념 없음** — OAuth가 아니므로 OAuth scope 문자열을 발급/지정하지 않는다.
- 권한은 두 축으로 결정:
  - **계정/IAM 권한**: 키가 속한 (서브)계정에 SENS 사용 권한이 있어야 함.
  - **서비스(프로젝트) 단위**: 해당 serviceId 프로젝트에서 선택한 서비스 타입(SMS/알림톡/친구톡)만 호출 가능.
- 정확한 IAM 권한 정책명은 **미확인** (아래 참조).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `NCP_ACCESS_KEY` | 콘솔 > 마이페이지 > 계정 및 보안 관리 > API 인증키 관리 의 Access Key ID | 필수 |
| `NCP_SECRET_KEY` | 같은 화면의 Secret Key (노출 금지, HMAC 서명 키) | 필수 |
| `SENS_SMS_SERVICE_ID` | 콘솔 > SENS > Project 의 SMS 서비스 타입 serviceId | SMS 도구 사용 시 |
| `SENS_ALIMTALK_SERVICE_ID` | SENS Project 의 알림톡 serviceId | 알림톡 도구 사용 시 |
| `SENS_FRIENDTALK_SERVICE_ID` | SENS Project 의 친구톡 serviceId | 친구톡 도구 사용 시 |
| `SENS_API_BASE_URL` | 베이스 URL. 기본 `https://sens.apigw.ntruss.com` / 금융 `https://sens.apigw.fin-ntruss.com` / 공공 `https://sens.apigw.gov-ntruss.com` | 선택 |
| `SENS_DEFAULT_FROM` | 사전등록된 발신번호(콘솔에서 등록·승인) | 선택(권장) |
| `SENS_MAX_RECIPIENTS` | 1요청 수신자 수 가드(기본 100). 비용/스팸 안전장치 | 선택(권장) |

> `serviceId`는 환경변수 대신 각 도구의 `serviceId` 파라미터로 요청별 override도 가능.

## 출처 (공식 URL)

- 공통 인증/서명·API 인증키 발급 위치: https://api.ncloud-docs.com/docs/en/common-ncpapi
- SENS 프로젝트 생성·serviceId 발급: https://guide.ncloud-docs.com/docs/sens-project
- SENS 서비스 개요: https://guide.ncloud-docs.com/docs/sens-sens-1-1
- 발신번호 사전등록(정책): https://guide.ncloud-docs.com/docs/sens-callingno
- SMS 정책(광고/080/국제): https://guide.ncloud-docs.com/docs/en/sens-smspolicy
- 카카오 비즈메시지 검수: https://guide.ncloud-docs.com/docs/sens-sens-1-5
- SMS API 가이드: https://api.ncloud-docs.com/docs/ai-application-service-sens-smsv2
- 알림톡 API 가이드: https://api.ncloud-docs.com/docs/ai-application-service-sens-alimtalkv2
- 친구톡 API 가이드: https://api.ncloud-docs.com/docs/ai-application-service-sens-friendtalkv2

## 미확인 / 주의

- **IAM 서브계정 SENS 권한 정책의 정확한 명칭/발급 화면 경로**: 미확인. 02_validation도 "콘솔에서 SENS 권한 부여"까지만 확정. 서브계정 사용 시 콘솔 IAM 메뉴에서 SENS 권한을 부여하되 정확한 정책명은 콘솔에서 직접 확인 필요.
- **serviceId가 프로젝트 목록 화면의 정확히 어느 위치에 표시되는지**: 공식 문서는 "프로젝트 목록에 채널별 serviceId가 부여된다"까지만 명시. 세부 UI 위치는 콘솔에서 확인.
- **`console.ncloud.com` URL**: 공식 문서는 콘솔 메뉴 경로만 안내하고 정확한 딥링크 URL은 명시하지 않음. 위 URL은 NCP 콘솔 표준 진입점 기준.
- **API 인증키 화면이 계정(루트) 키 기준으로 서술됨**: 위 A절 절차는 공식 문서가 보여준 계정 단위 API 인증키 화면 기준. 서브계정 키 발급 절차는 별도(IAM)로, 본 문서에서 화면 단위까지 확정하지 않음.
- **시크릿 취급**: `NCP_SECRET_KEY`·serviceId는 노출 금지. serviceId 노출 시 재발급 권고(출처: sens-project). `.env`는 커밋 금지.
- **레이트리밋/1요청 최대 수신자/발송 에러코드 표**: 공식 수치 미확인 → 하드코딩 금지, 런타임 에러로 흡수(README/02_validation 참조).
- **Push v2·수신형 Webhook**: 경로/근거 미확인으로 이 커넥터 범위에서 제외.
