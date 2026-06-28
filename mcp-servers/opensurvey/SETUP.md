# 키 발급 가이드 — 오픈서베이 / Opensurvey (opensurvey)

> 오픈서베이 Dataspace API **V3** 연동용 자격증명 발급 가이드.
> 모든 사실은 `_workspace/opensurvey/02_validation.md`(검증일 2026-06-26)의 출처 있는 검증 항목에서 가져왔습니다. 추측 없음.

## 연동 유형 / 인증 방식
- **연동 유형**: REST API (베이스 URL `https://api.opensurvey.io`, 버전 `/v3`).
- **인증 방식**: API Key 기반 **HTTP Basic 인증**.
  - 헤더: `Authorization: Basic <base64NoPadding(SPACE_ID + ":" + API_KEY)>`
  - **base64는 no-padding 사용** — 표준 base64 인코딩 후 trailing `=` 패딩을 제거해야 합니다.
  - **HTTPS + TLS 1.2 이상만** 지원.
  - OAuth / 토큰 교환 없음 (사전 발급된 정적 키 2종을 환경변수로 주입).
- 출처: https://developer.opensurvey.io/ko/articles/8330185-api-인증-방식-authentication , https://developer.opensurvey.io/ko/articles/8504647-api-연동-quick-start

## 발급 절차 (스텝)
자격증명은 2종입니다: **Space ID**(스페이스당 자동 발급) + **API 인증키**('developer' 권한으로 발급).

- **발급 페이지 직링크**: **없음(직링크 부재)**. 오픈서베이 콘솔은 워크스페이스(스페이스)별 로그인 게이트 화면이라 공용 발급 직링크 URL이 공식 문서에 공개되어 있지 않습니다. → 아래 **메뉴 경로**로 진입하세요.
- **정확한 메뉴 경로**: **로그인 → [설정] → [연동] → [API]** (공식 인증 문서에 `[설정 > 연동 > API]`로 명시). 이 화면에서 Space ID 확인과 API 인증키 발급/재발급을 모두 수행합니다.

1) **Team 권한 계정으로 멤버를 스페이스에 초대**한다.
2) **Admin 권한 계정이 해당 멤버에게 'developer' 권한을 부여**한다. (키 발급은 developer 권한만 가능)
3) 오픈서베이 웹 콘솔에 로그인 후 **[설정] → [연동] → [API]** 메뉴로 이동한다. (공식 표기: `[설정 > 연동 > API]`)
4) 해당 화면에서 **Space ID를 확인**(자동 발급됨, 스페이스 소속이면 누구나 조회 가능)하고, **API 인증키를 발급/재발급**한다. (발급/재발급은 'developer' 권한 계정만 가능)
5) 발급된 Space ID와 API 인증키를 아래 환경변수에 주입한다.

> 별도의 운영키 발급 절차나 오픈서베이 측 심사·승인 절차는 **없습니다**(자가 발급). 개발 → 운영 즉시 전환.
> 출처: https://developer.opensurvey.io/ko/articles/8330185-api-인증-방식-authentication (메뉴 경로 `[설정 > 연동 > API]`·developer 권한 명시) , https://developer.opensurvey.io/ko/articles/8330193-연동-절차 (권한 절차·심사 불요)

## 전제조건
- **요금제**: 오픈서베이 **Enterprise 플랜** 필요. API 연동은 Enterprise 플랜에서만 동작하며, 비-Enterprise 계정은 키가 있어도 모든 호출이 `402 Payment Required`로 실패합니다(코드 버그 아님 — 플랜/계약 상태 문제).
  - 출처: https://support.opensurvey.io/ko/articles/8508642-데이터스페이스-api-연동-가이드
  - 주의(문서 간 불일치): 개발자센터 "연동 절차" 문서에는 플랜 요건이 명시되어 있지 않아, **키 발급까지는 성공하고 실제 호출 단계에서 402로 막힐 수 있습니다.**
- **조직 권한 체계**: 멤버 초대(Team) → 권한 부여(Admin) → 키 발급(developer)의 사내 권한 협조가 선행되어야 합니다.
  - 출처: https://developer.opensurvey.io/ko/articles/8330193-연동-절차
- **사업자 심사 / 발신번호 사전등록 / 오픈서베이 승인**: 없음 (자가 발급).

## 스코프 / 권한
- API 키 자체에는 **리소스별 스코프 체계가 없습니다.** 접근은 **스페이스(Space) 단위**로 이루어집니다.
- 타 스페이스의 리소스에 접근하면 `403`이 반환됩니다.
- 본 MCP 서버가 사용하는 V3 API는 **전부 읽기 전용(GET) 6종**입니다 (설문 생성/수정/발송 write API는 존재하지 않음).
- 출처: https://developer.opensurvey.io/ko/articles/8330185-api-인증-방식-authentication , https://developer.opensurvey.io/api/index.html

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `OPENSURVEY_SPACE_ID` | 콘솔 [설정 > 연동 > API]의 **Space ID**(자동 발급). HTTP Basic의 username 부분. | 필수 |
| `OPENSURVEY_API_KEY` | 콘솔 [설정 > 연동 > API]에서 **발급한 API 인증키**(developer 권한 필요). HTTP Basic의 password 부분. | 필수 |
| `OPENSURVEY_BASE_URL` | 베이스 URL 오버라이드. 기본값 `https://api.opensurvey.io` (보통 미설정). | 선택 |

> 시크릿은 코드/`.mcp.json`에 실값으로 쓰지 마세요. `.env`(gitignore됨) 또는 환경변수 확장(`${VAR}`)으로 주입합니다.

## 출처 (공식 URL)
- 인증 방식(HTTP Basic / Space ID·API키 / TLS / **발급 메뉴 경로 `[설정 > 연동 > API]`** / developer 권한): https://developer.opensurvey.io/ko/articles/8330185-api-인증-방식-authentication
  - 검증(2026-06-28): 위 인증 문서가 메뉴 경로 `[설정 > 연동 > API]`와 'developer' 권한을 명시. **콘솔 발급 직링크 URL은 공식 문서 어디에도 미공개**(인증 문서·연동 절차·Quick Start 3건 모두 확인) → "직링크 없음 + 메뉴 경로"로 확정.
- Quick Start(헤더 형식 / base64 no-padding): https://developer.opensurvey.io/ko/articles/8504647-api-연동-quick-start
- 연동 절차(Team→Admin→developer 권한, 심사·승인 불요): https://developer.opensurvey.io/ko/articles/8330193-연동-절차
- Enterprise 플랜 요건: https://support.opensurvey.io/ko/articles/8508642-데이터스페이스-api-연동-가이드
- API 레퍼런스 포털(베이스 URL·V3·엔드포인트): https://developer.opensurvey.io/api/index.html
- V1/V2 지원 종료(2026.12): https://developer.opensurvey.io/ko/articles/12582840-2026-12-api-통합-및-v1-v2-api-지원-종료-안내
- 제약 사항(레이트리밋·에러코드): https://developer.opensurvey.io/ko/articles/8504665-제약-사항

## 미확인 / 주의
- **base64 no-padding 함정**: 표준 라이브러리 기본 base64는 `=` 패딩을 붙입니다. 그대로 쓰면 인증 실패 가능 → 패딩 제거를 명시적으로 적용하고 실호출로 1회 검증하세요. (본 서버는 구현됨.)
- **V1/V2 폐기 + Quick Start의 v1 예시 함정**: 공식 Quick Start가 아직 `/v1/` 예시 URL을 노출합니다. 그대로 따라하면 2026.12 이후 깨집니다 → 구현/호출은 `/v3` 고정.
- **402 식별**: 키가 정상이어도 비-Enterprise 플랜이면 전 호출이 402로 실패합니다. "Enterprise 플랜/계약 상태 확인 필요"로 해석하세요(키 오류 아님).
- **서버사이드 공식 SDK(npm/pip)**: 미확인. 개발자센터에서 발견하지 못함(Postman 예시만 안내). 자체 `fetch` 기반 구현 권고.
- **API 인증키 만료/로테이션 정책**: 02_validation 범위 내 출처에 명시 없음 → **미확인**. 재발급은 콘솔 [설정 > 연동 > API]에서 가능(developer 권한).
