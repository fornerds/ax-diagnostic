# 키 발급 가이드 — 네이버 밴드 (band)

> 출처 기준일: 2026-06-26 (검증본 `_workspace/band/02_validation.md`)
> 모든 항목은 검증본/README에 출처 URL이 있는 사실만 기재한다. 출처가 없는 발급 디테일은 "미확인 / 주의"에 둔다.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 MCP 서버 빌드(route=build). Claude 공식 커넥터 디렉터리에 BAND 미등재 → 원격 커넥터 없음.
  - 출처: https://claude.com/docs/connectors/directory , https://support.claude.com/en/articles/14503689-mcp-connectors
- **인증 방식**: OAuth 2.0 Authorization Code Grant. 모든 리소스 호출에 `access_token`을 쿼리 파라미터로 전달하며 HTTPS 필수.
  - 출처: https://developers.band.us/develop/guide/api/get_authorization_code_from_user
- **토큰 호스트**: `https://auth.band.us` (`/oauth2/authorize`, `/oauth2/token`)
- **리소스 호스트**: `https://openapi.band.us`
- **두 가지 토큰 경로**:
  1. **단일 사용자(본인 밴드) — 권장**: BAND 개발자센터 > My Apps의 **"Connect BAND account"**로 access_token을 즉시 발급받아 env에 주입. 리다이렉트 구현 불필요. 단, 본인(앱 소유 계정) 계정에만 권한이 부여됨.
     - 출처: https://developers.band.us/develop/guide/api/get_authorization_code_from_user ("only you (the person who owns the account) can be given permission")
  2. **표준 2단계(타 사용자 대상)**: `GET https://auth.band.us/oauth2/authorize?response_type=code&client_id=…&redirect_uri=…` → code 수신 → `GET https://auth.band.us/oauth2/token?grant_type=authorization_code&code=…` + 헤더 `Authorization: Basic base64(client_id:client_secret)` → access_token 수신.
     - 출처: 〃

## 발급 절차 (스텝)

> 본 MCP 서버는 기본적으로 **경로 1(Connect BAND account)** 의 access_token만 있으면 동작한다(`client_id`/`client_secret`은 표준 OAuth 재발급 시에만 선택 사용).

**경로 1 — Connect BAND account (단일 사용자, 권장)**
1. BAND 개발자센터(`https://developers.band.us`)에 BAND 계정으로 로그인한다.
2. **앱 등록 폼 직링크** `https://developers.band.us/develop/myapps/api/form` 으로 이동해 서비스(앱)를 등록한다. 이때 redirect_uri는 임의 값이라도 입력해 두면 된다(경로 1에서는 실제 리다이렉트 미사용).
3. **My Apps 목록 직링크** `https://developers.band.us/develop/myapps/list` 로 이동 → 방금 등록한 **앱 이름을 클릭**해 "API 수정(앱 설정)" 페이지로 들어간다.
4. 그 페이지에서 **[밴드계정 연동](Connect BAND account)** 버튼을 클릭하면 본인 계정 access_token이 화면에 표시된다.
   - 메뉴 경로 요약: 로그인 → `myapps/api/form`(앱 등록) → `myapps/list` → **앱 이름 클릭** → API 수정 페이지 → **[밴드계정 연동 / Connect BAND account]** 버튼.
5. 표시된 access_token을 env `BAND_ACCESS_TOKEN`에 주입한다.
   - 토큰 응답 필드: `access_token`, `token_type=bearer`, `refresh_token`, `expires_in=315359999`(약 10년), `scope`, `user_key`.
   - 출처: https://developers.band.us/develop/guide/api/get_authorization_code_from_user (인증/토큰 응답), 발급 화면 경로·버튼명은 https://github.com/Happysttim/band-api , https://github.com/jungwuk-ryu/PMMP-bandAPI (콘솔 URL `myapps/api/form`·`myapps/list`·"밴드계정 연동" 버튼 명시) 교차 확인.

**경로 2 — 표준 OAuth(타 사용자 대상, 필요 시에만)**
1. 앱 등록 폼(`https://developers.band.us/develop/myapps/api/form`)에서 redirect_uri 도메인을 등록한다(요청 시에는 full URL 전달).
2. 사용자에게 `GET https://auth.band.us/oauth2/authorize?response_type=code&client_id=…&redirect_uri=…` 동의 화면을 띄워 `code`를 받는다.
3. `GET https://auth.band.us/oauth2/token?grant_type=authorization_code&code=…` 호출 + 헤더 `Authorization: Basic base64(client_id:client_secret)` → access_token 수신.
4. access_token을 `BAND_ACCESS_TOKEN`에, 필요 시 `BAND_CLIENT_ID`/`BAND_CLIENT_SECRET`을 env에 설정.
   - 출처: https://developers.band.us/develop/guide/api/get_authorization_code_from_user

## 전제조건

- **플랜/유료 요건**: 출처에 별도 언급 없음 → **확인된 요건 없음**(미확인 아님이 아니라 문서 미기재. "미확인 / 주의" 참조).
- **사업자 심사**: 검증본상 별도 심사 단계 언급 없음. 개인 개발자 본인 계정 토큰은 즉시 발급으로 보임(프로덕션 쿼터 상향 심사 유무는 미확인).
  - 출처/판정: `02_validation.md` 미확인#3
- **발신번호 사전등록 등**: 해당 없음(BAND OpenAPI는 SMS/발신번호 개념 없음) → **없음**.
- **선행 의존**: 거의 모든 작업이 `band_key`를 요구 → 첫 흐름은 `list_bands`로 band_key 확보.
  - 출처: `02_validation.md` 구현 제약

## 스코프 / 권한

- **공식 스코프 목록(10개, 토큰 응답 scope와 정확히 일치)**:
  `READ_MY_PROFILE`, `READ_BAND_AND_USERS_LIST`, `READ_POST`, `WRITE_POST`, `DELETE_POST`, `READ_COMMENT`, `CREATE_COMMENT`, `READ_BAND_PERMISSION`, `READ_ALBUM`, `READ_PHOTO`.
  - 출처: https://developers.band.us/develop/guide/api/get_authorization_code_from_user
- **작업별 매핑**:
  - 읽기: READ_BAND_AND_USERS_LIST / READ_POST / READ_COMMENT / READ_BAND_PERMISSION / READ_MY_PROFILE / READ_ALBUM / READ_PHOTO
  - 쓰기: WRITE_POST / CREATE_COMMENT
  - 삭제: DELETE_POST (글·댓글 공통, **`contents_deletion` 권한과 결합**)
- **주의 — 별도 댓글삭제 스코프 없음**: `DELETE_COMMENT` 스코프는 공식 목록에 **존재하지 않음**. 댓글 삭제는 `DELETE_POST` 스코프 + `contents_deletion` 권한으로 처리.
  - 출처: 〃 (`02_validation.md` 수정됨 항목)
- **권한 선행 체크**: 쓰기/삭제 전 `check_permissions(band_key, "posting,commenting,contents_deletion")`로 확인 권장. `contents_deletion`은 게시자 본인 여부를 검사하지 않으므로(권한만 있으면 타인 글도 삭제) 파괴적 동작은 신중히.
  - 출처: https://developers.band.us/develop/guide/api/get_post_permission

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `BAND_ACCESS_TOKEN` | BAND 개발자센터 > My Apps > **"Connect BAND account"** 로 발급한 access_token(본인 계정 한정). 모든 리소스 호출에 쿼리로 자동 부착됨. 없으면 서버는 시작 시 즉시 종료. | 예 |
| `BAND_CLIENT_ID` | My Apps의 앱 등록 정보. 표준 OAuth 토큰 교환/재발급 시 Basic 헤더 구성에 사용(현재 서버 미사용, 예약). | 아니오 |
| `BAND_CLIENT_SECRET` | 〃 (앱 등록 정보). **절대 하드코딩 금지** — env로만 관리. | 아니오 |

> 출처: `02_validation.md` 환경변수 목록, `README.md` 환경변수 표.

## 출처 (공식 URL)

- 인증/OAuth/스코프/Connect BAND account: https://developers.band.us/develop/guide/api/get_authorization_code_from_user
- 권한 확인(`contents_deletion` 등): https://developers.band.us/develop/guide/api/get_post_permission
- 에러코드: https://developers.band.us/develop/guide/api/handle_errors
- 개발자 가이드 개요: https://developers.band.us/develop/guide/api
- Claude 커넥터 디렉터리(BAND 미등재 확인): https://claude.com/docs/connectors/directory , https://support.claude.com/en/articles/14503689-mcp-connectors

## 미확인 / 주의

- **My Apps 콘솔 정확한 URL·앱 등록 화면 단계**: `developers.band.us`가 개발자센터이고 My Apps에서 앱 등록 후 "Connect BAND account"로 토큰을 발급한다는 사실은 검증됨. 단 **앱 등록 폼의 정확한 경로 URL과 화면별 입력 항목**은 사용 가능한 출처(검증본/README)에 명시되지 않음. 공식 사이트 재확인 시도(WebFetch)는 `developers.band.us` 도메인 차단으로 실패 → **미확인**(추후 browse 스킬로 콘솔 직접 확인 권장).
- **refresh_token 교환 엔드포인트**: `grant_type=refresh_token` 절차가 OAuth 가이드에 없음. `expires_in`이 약 10년이라 단기 영향 적음. 토큰 만료(10401/invalid_token) 시 **재로그인 유도**로 처리, 자동 refresh는 보류.
- **사업자/심사·프로덕션 쿼터 상향 요건**: 문서상 "앱 등록 후 access token 발급"만 기술. 별도 심사 단계 언급 없음 → 상향 심사 유무는 **미확인**.
- **레이트리밋 수치/윈도우**: 코드 1001(App)/1002(User)/1003(Cool down)의 분·일당 한도·쿨다운 초는 공식문서 미기재 → 코드 기반 지수 백오프로 대응(수치 하드코딩 금지).
- **타인 콘텐츠 삭제 위험**: `contents_deletion` 권한은 본인 글 여부를 검사하지 않음 → 파괴적 도구(`delete_post`/`delete_comment`)는 키 확인·권한 체크 동반.
  - 출처: https://developers.band.us/develop/guide/api/get_post_permission
