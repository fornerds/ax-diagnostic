# 키 발급 가이드 — 노티플라이 (notifly)

> 출처 기준: `_workspace/notifly/02_validation.md`(검증일 2026-06-26) 및 `mcp-servers/notifly/README.md`. 추측 없이 출처 있는 사실만 기재. 콘솔 로그인 URL은 2026-06-28 실제 페이지 확인으로 확정(`console.notifly.tech`).

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 MCP 서버 빌드(운영용 공식 원격 커넥터 없음 — 공식 MCP는 문서/SDK 검색 전용).
- **인증 방식**: 2단계 토큰 방식.
  1. `POST https://api.notifly.tech/authenticate` 바디 `{ "accessKey": "...", "secretKey": "..." }`
  2. 응답 `data` 필드 = 인증 토큰(JWT, **1시간 유효**)
  3. 이후 모든 호출에 `Authorization: Bearer <token>` + `Content-Type: application/json`
- 토큰은 MCP 서버 내부에서 자동 발급/캐시(1시간)된다. 사용자가 직접 토큰을 발급하지 않는다. 사용자가 준비할 것은 **accessKey / secretKey / projectId** 3가지뿐이다.

## 발급 절차 (스텝)

**발급 위치 한 줄 요약**: 노티플라이 콘솔(`console.notifly.tech`) 로그인 → 프로젝트 선택 → **프로젝트 "설정" 페이지**에서 `accessKey` / `secretKey` / `projectId` 확보.

1) 노티플라이 콘솔 로그인 페이지로 접속해 로그인한다.
   - **로그인 직링크**: https://console.notifly.tech/ko/auth/login (영문 `…/en/auth/login`) — 2026-06-28 실제 페이지 확인(제목 "계정에 로그인해주세요", 이메일/비밀번호 폼).
   - 계정이 없으면 회원가입: https://console.notifly.tech/ko/auth/register
2) 로그인 후 콘솔 메인(`https://console.notifly.tech/ko/console/products`)에서 **대상 프로젝트(제품)를 선택**한다.
3) 선택한 프로젝트의 **"설정" 페이지**로 이동한다.
   - 이 "설정" 페이지가 키/ID의 단일 출처다(공식 문서 표현: "노티플라이 프로젝트 설정 페이지에서 확인"). 같은 페이지에 카카오톡 채널 등 채널 설정 섹션도 함께 있다.
4) 설정 페이지에서 **Access Key(`accessKey`)와 Secret Key(`secretKey`)**를 확인/확보한다. **프로젝트당 Access Key 1개**가 제공된다.
5) 같은 설정 페이지에서 **projectId**(거의 모든 API 작업의 필수값)를 확인한다.
6) 확보한 3개 값을 아래 "환경변수 매핑"대로 환경변수에 주입한다.

> **발급 직링크에 대하여**: 로그인 직링크는 위 1)로 확정. 단 *특정 프로젝트의 설정 페이지로 바로 가는 직링크*는 콘솔이 로그인+프로젝트 선택 후 동작하는 SPA이고 경로가 프로젝트별이라 범용 직링크가 없다 → "프로젝트별 직링크 없음 + 위 메뉴 경로(로그인 → 프로젝트 선택 → 설정)"로 접근.
>
> 별도 사업자 심사·제휴 승인 단계는 문서상 없음 → 가입 고객이면 즉시 키 확보 가능.

## 전제조건

키 발급 자체에는 추가 전제조건이 **없음**(별도 사업자 심사/제휴 승인 단계 문서상 없음).

단, **키만으로는 실발송이 안 되는 채널이 있다.** 아래는 발급이 아닌 "실발송"의 선행 절차다(콘솔에서 먼저 완료해야 함):

- **문자(SMS/LMS/MMS)**: 콘솔에서 **발신번호 등록** + 문자 발송 설정 완료해야 API 발송 가능. 미설정 시 `400 "Sender info is not registered."` (출처: send-text-message)
- **카카오 알림톡**: **발신프로필(카카오 채널) 등록** + **템플릿 검수 `approved`**(통상 1~2영업일) 선행. approved templateId만 발송 가능. (출처: alimtalk-guide, send-kakao-alimtalk)
- **080 광고 수신거부**: API는 **조회만** 제공(등록/삭제는 콘솔에서). 마케팅 발송 시 정보통신망법 대응 필요.

## 스코프 / 권한

- 세분 스코프 없음. 키 1개 = **프로젝트 단위 전체 권한**으로 보임.
- 작업별 권한 구분은 공식 문서에서 확인되지 않음(미확인이나 사용 차단 요인은 아님).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `NOTIFLY_ACCESS_KEY` | 콘솔 → 프로젝트 설정의 Access Key (`accessKey`) | 예 |
| `NOTIFLY_SECRET_KEY` | 콘솔 → 프로젝트 설정의 Secret Key (`secretKey`) | 예 |
| `NOTIFLY_PROJECT_ID` | 콘솔 → 프로젝트 설정의 projectId (기본 프로젝트; 각 도구의 `projectId` 인자로 override 가능) | 예(실질) |
| `NOTIFLY_BASE_URL` | 베이스 URL override (기본 `https://api.notifly.tech`) | 아니오 |

> `NOTIFLY_ACCESS_KEY` / `NOTIFLY_SECRET_KEY` 가 없으면 MCP 서버는 시작 시 즉시 종료한다. 시크릿은 코드/`.mcp.json`에 하드코딩하지 말고 환경변수로만 주입한다.

## 출처 (공식 URL)

- **콘솔 로그인(발급 위치 진입점)**: https://console.notifly.tech/ko/auth/login (회원가입 https://console.notifly.tech/ko/auth/register) — 2026-06-28 실제 페이지 확인
- **콘솔 메인(프로젝트 선택)**: https://console.notifly.tech/ko/console/products
- 키/ID 출처가 "프로젝트 설정 페이지"임을 명시한 공식 문서: https://docs.notifly.tech/api-reference/authenticate , https://docs.notifly.tech/ko/api-reference/getting-started
- API 개요(베이스 URL, 인증/Authorization 헤더): https://docs.notifly.tech/ko/api-reference/getting-started
- 인증 엔드포인트(`POST /authenticate`, 토큰 1시간): https://docs.notifly.tech/api-reference/authenticate
- 문자 발송(발신정보 미등록 400): https://docs.notifly.tech/api-reference/send-text-message
- 카카오 알림톡(템플릿 승인 선행): https://docs.notifly.tech/api-reference/send-kakao-alimtalk
- 공식 MCP(문서/SDK 검색 전용, 운영 아님): https://docs.notifly.tech/ko/devtools/notifly-mcp-server , https://github.com/notifly-tech/notifly-mcp-server

## 미확인 / 주의

- **콘솔 로그인 URL 확정(2026-06-28)**: `https://console.notifly.tech/ko/auth/login`(실제 페이지 확인). 발급 위치 = 로그인 → 프로젝트 선택 → 프로젝트 "설정" 페이지. ~~이전 "콘솔 URL 미확인"은 해소됨.~~
- **설정 페이지 내부 하위 탭/버튼명 부분 미확인**: 콘솔이 로그인 게이트 뒤의 SPA라 "설정" 페이지 안에서 accessKey/secretKey가 노출되는 정확한 하위 섹션/탭 명칭과 프로젝트별 직링크는 비로그인 상태로 스크랩 불가. 공식 문서가 "프로젝트 설정 페이지에서 확인"이라고 명시한 수준까지는 확정. 정확한 탭명은 로그인 후 직접 확인 권장.
- **작업별 스코프/권한 구분 미확인**: 세분 스코프 문서 미발견.
- **레이트리밋(RPS/일일 쿼터) 미확인**: 전역 레이트리밋 문서 미발견. 배치 상한(track ≤500, 그 외 ≤1,000)만 확정.
- **플랜별 API 가용 범위 미확인**: 문자/알림톡이 유료 플랜·채널 활성화에 종속될 가능성 있으나 플랜별 매트릭스 문서 미발견.
- **유료 도구 주의**: `create_campaign_event_export`는 **요청당 과금**.
- **파괴적 도구 주의**: `delete_users`(DELETE /users)는 비가역. 식별자 없는 빈 조건 호출은 거부됨.
