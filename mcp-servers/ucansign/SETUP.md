# 키 발급 가이드 — 유캔싸인 (ucansign)

> 출처 기준일: 2026-06-26 (`_workspace/ucansign/02_validation.md` 검증본). 모든 사실은 아래 "출처" 섹션의 공식 URL에 근거하며, 확인되지 않은 항목은 "미확인 / 주의"에 명시한다.

## 연동 유형 / 인증 방식
- **연동 유형**: 로컬 빌드 stdio MCP 서버 (공식 원격 MCP 커넥터 없음 → route=build).
  - 근거: 개발자센터 전 메뉴에 MCP/Remote MCP/Claude connector 언급 0건. (https://app.ucansign.com/developer)
- **인증 방식**: **API KEY → accessToken(Bearer) 교환** (서버가 자기 계정으로 운영하는 단일 키 방식, MCP 권장).
  - `POST https://app.ucansign.com/openapi/user/token` 바디 `{"apiKey": "<API KEY>"}` → 응답 `result.accessToken` (유효 30분).
  - 이후 모든 호출에 `Authorization: Bearer {accessToken}`. 토큰 만료(에러 441) 시 서버가 재발급 후 1회 재시도.
  - 토큰 발급/갱신은 서버 내부에서 처리하므로 사용자는 **API KEY 하나만** 발급받으면 된다.
  - 근거: https://app.ucansign.com/developer/auth
- **OAuth2**(다중 사용자 위임): 셀프 등록(`개발자 > 클라이언트`)으로 가능하나, 토큰 교환 HTTP 메서드가 공식문서 2곳에서 상충(GET vs POST)하여 1차 범위에서 **보류**. 이 가이드의 발급 대상이 아니다.

## 발급 절차 (스텝)
1. **유캔싸인 회원가입 / 로그인** — https://app.ucansign.com (기존 회원이면 로그인만).
2. **개발자 기능 진입** — 로그인 후 개발자센터(https://app.ucansign.com/developer) 접속.
   - "유캔싸인 회원이라면 누구나 개발자 기능 이용" — 별도 신청/심사 없음.
3. **API KEY 확인** — `개발자 메뉴 > API KEY` 탭에서 자동 발급된 키를 확인/복사한다.
   - "개발자로 등록된 계정은 API 키를 자동으로 발급". (https://app.ucansign.com/developer/auth)
4. **환경변수 주입** — 복사한 키를 `UCANSIGN_API_KEY`로 설정한다(아래 환경변수 매핑 참조). 서버가 이 키로 accessToken을 자동 교환한다.
5. **(선택) 무과금 테스트 모드** — `UCANSIGN_TEST_MODE=true` 설정 시 발송 계열 호출에 `x-ucansign-test: true`가 강제되어 과금 없이 테스트(워터마크 부착, 법적효력 없음).

## 전제조건
- **사업자 심사 / 제휴 심사**: **없음**. 회원이면 누구나 API KEY 자동 발급. (https://app.ucansign.com/developer, https://app.ucansign.com/developer/auth)
- **발신번호 사전등록**: **API KEY 발급 자체에는 불필요**. 단, 카카오 알림톡 발송(`signingMethodType:kakao`) 시 발신프로필 사전등록 요건 여부는 개발자문서에 명시 없음 → "미확인 / 주의" 참조.
- **포인트 선충전(운영 전제, 발급과는 별개)**: 서명요청 1건 = 1포인트 차감. 0포인트 + 자동충전 OFF면 에러 1039로 발송 실패. 발송 전 `get_point_balance`로 잔액 점검 권장. (https://app.ucansign.com/developer/sign-request, https://app.ucansign.com/developer/error-code)
- **템플릿 선행 생성(운영 전제)**: API만으로 서명문서를 발송하려면 관리자페이지에서 템플릿을 미리 생성해야 한다. 템플릿의 사인/도장 '추가내용 입력하기' 필드가 비어 있으면 에러 7003. (https://app.ucansign.com/developer/sign-request)

## 스코프 / 권한
- **스코프 체계 없음(명시적 부재)**. API KEY로 교환한 accessToken은 해당 회원 **계정 전체 권한**을 가진다. 별도 scope 파라미터/권한 등급 지정 불가.
  - 근거: https://app.ucansign.com/developer/oauth (전문에 scope 언급 0건).
- 따라서 발급 시 권한 범위를 선택하는 단계는 없으며, 키 노출 = 계정 전체 권한 노출이므로 백엔드 보관 필수.

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `UCANSIGN_API_KEY` | 관리자/개발자 메뉴 > **API KEY** 탭에서 자동 발급된 키 (https://app.ucansign.com/developer/auth). 백엔드에만 보관, 평문 커밋 금지 | 필수 |
| `UCANSIGN_BASE_URL` | 베이스 URL 오버라이드용. 기본값 `https://app.ucansign.com/openapi` (변경 불필요 시 미설정) | 선택 |
| `UCANSIGN_TEST_MODE` | 상수 `true`. 설정 시 발송 계열에 `x-ucansign-test:true` 강제(무과금·워터마크·법적효력 없음) | 선택 |

## 출처 (공식 URL)
- 개발자센터 홈: https://app.ucansign.com/developer
- 인증 & 인가(토큰 발급·API KEY 자동발급·30분 유효): https://app.ucansign.com/developer/auth
- OAuth(클라이언트 셀프등록·scope 부재): https://app.ucansign.com/developer/oauth
- 서명요청(테스트 헤더·1포인트·템플릿 선행): https://app.ucansign.com/developer/sign-request
- 에러코드(441 토큰만료, 1039 포인트부족, 7003 서명이미지 미입력): https://app.ucansign.com/developer/error-code
- 콘솔(회원가입/로그인): https://app.ucansign.com

## 미확인 / 주의
- **OAuth 토큰 교환 메서드 상충**: 가이드 cURL은 `GET /openapi/user/oauth/auth`, Postman 컬렉션은 `POST` — 공식 2소스 불일치. OAuth 발급/연동은 보류이며, 사용할 경우 GET 우선·POST 폴백 가드 필요.
- **카카오 알림톡 발신 요건 미확인**: `signingMethodType:kakao`는 파라미터로 동작하나, 발신프로필/발신번호 사전등록을 유캔싸인이 대행하는지 사용자 측 등록이 필요한지 개발자문서에 명시 없음. 카카오 발송 실패 시 응답 `msg` 확인.
- **레이트리밋 수치 미공개**: 공식 numeric 한도 명시 없음. 5xx/일시 오류는 지수 백오프 재시도로 대응(4xx는 재시도 금지).
- **accessToken 30분 만료**: 서버가 토큰 캐시 + 만료(441) 자동 재발급을 처리. 사용자가 직접 다룰 항목은 아니나, 장시간 호출 없던 세션은 첫 호출에서 재발급이 일어날 수 있음.
- **SSL 루트 인증서**: 2024.07.09 SSL R10→R6 갱신. 구형 HTTP 클라이언트는 루트 인증서 갱신이 필요할 수 있음. (https://app.ucansign.com/developer/updatelog)
