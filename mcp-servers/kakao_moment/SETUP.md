# 키 발급 가이드 — 카카오모먼트 (kakao_moment)

> 출처: `_workspace/kakao_moment/02_validation.md`(검증일 2026-06-26), `README.md`. 추측 없이 검증된 사실만 기재. 발급 디테일이 문서에 없는 항목은 "미확인"으로 표기.

## 연동 유형 / 인증 방식
- **연동 유형**: 로컬 MCP 서버 빌드(공식 Claude 커넥터 없음 — 재확인됨). 카카오모먼트 광고 오픈API(REST) 직접 호출.
- **인증 방식**: 카카오 **비즈니스 토큰(Bearer)**. OAuth 2.0 비즈니스 인가코드 → 비즈니스 토큰 교환.
  - 일반 카카오 로그인 access token으로는 호출 불가. `kauth.kakao.com/oauth/business/*` 경유의 비즈니스 토큰만 유효.
  - **refresh_token 없음**: 토큰 응답에 `refresh_token`·`expires_in`이 포함되지 않는다. 매번 새로 발급되며 장기 미사용 시 자동 만료. 만료 시 발급 절차(아래 1~2단계)를 사람이 재수행해야 함. 서버는 토큰을 발급/갱신하지 않고 env로 주입받은 토큰만 사용.
- **필수 요청 헤더**(모든 모먼트 API): `Authorization: Bearer {비즈니스 토큰}`, `adAccountId: {광고계정 번호}`.
  - 예외: 광고계정 목록 `GET /openapi/v4/adAccounts/pages`는 `adAccountId` 헤더 없이 호출 가능.

## 발급 절차 (스텝)
> 토큰 발급은 **서버 외부에서 1회 수행**한다(브라우저 리다이렉트 동의가 필요해 MCP stdio 도구로 자동화 불가). 발급된 `access_token`을 env에 주입한다.

> **콘솔 직링크**: 카카오 개발자 콘솔 앱 목록(=앱 관리 페이지) → **https://developers.kakao.com/console/app** (로그인 필요). 개별 앱·키·신청 화면은 이 콘솔 안에서 앱을 선택해 진입하며 워크스페이스/앱별 URL이라 보편 직링크는 없다. 아래 메뉴 경로로 이동.

0. **REST API 키(client_id)·리다이렉트 URI·client_secret 확보 (콘솔)**
   - 콘솔(https://developers.kakao.com/console/app)에서 앱 선택 → **[앱] > [플랫폼 키] > [REST API 키]** 에서 `REST_API_KEY`(=client_id) 확인.
   - 같은 화면 **[앱] > [플랫폼 키] > [REST API 키] > [리다이렉트 URI]** 에서 `REDIRECT_URI` 등록.
   - (선택) client_secret 사용 시 같은 **[앱] > [플랫폼 키] > [REST API 키]** 화면에서 client_secret 발급·활성화.

1. **전제조건 충족**(아래 "전제조건" 섹션) — 비즈 앱 전환 + 본인인증 + 사용 권한 승인. 미완 시 이후 모든 호출이 403.

2. **인가코드 요청 (브라우저)**
   ```
   GET https://kauth.kakao.com/oauth/business/authorize
     ?client_id={REST_API_KEY}
     &response_type=code
     &redirect_uri={REDIRECT_URI}
     &scope=moment_management            # 쉼표 구분으로 다중 지정 가능
     &resource_ids=moment_create:{AD_ACCOUNT_ID}   # moment_create scope 포함 시 필수
     &state={임의값}                      # 선택
   ```
   - 1차 도구(조회·보고서·상태/예산)는 `moment_management` 스코프로 충분하다.
   - `moment_create` 또는 `keyword_create` 스코프를 포함할 때만 `resource_ids`(형식 `ScopeGroup:ResourceId`, 즉 `moment_create:{adAccountId}`)가 필수.

3. **토큰 교환**
   ```
   POST https://kauth.kakao.com/oauth/business/token
     grant_type=authorization_code
     client_id={REST_API_KEY}
     code={2단계에서 받은 code}
     redirect_uri={REDIRECT_URI}
     client_secret={client_secret 활성화한 경우}
   ```
   - 응답: `access_token`, `token_type=bearer`, `scope`. **refresh_token·expires_in 없음.**

4. **토큰 주입**: 위 `access_token`을 `KAKAO_MOMENT_ACCESS_TOKEN`에, 대상 광고계정 번호를 `KAKAO_MOMENT_AD_ACCOUNT_ID`에 넣는다.

> 발급에 쓰는 `REST_API_KEY`(client_id)·`client_secret`·`redirect_uri`는 **서버 런타임에 불필요**하다. MCP 서버 본체는 발급된 비즈니스 토큰만 받는다. 별도 발급 헬퍼 스크립트를 둘 때만 `KAKAO_REST_API_KEY`·`KAKAO_CLIENT_SECRET`·`KAKAO_REDIRECT_URI`를 사용.

## 전제조건
키(토큰)만으로 동작하지 않는다. 다음이 선행되어야 하며, 미충족 시 모든 호출이 403이다.

1. **비즈 앱 전환 + 앱 소유자 본인인증.** 본인인증된 카카오계정이 해당 광고계정의 마스터/멤버 권한을 보유해야 함.
   - 본인인증 경로: 콘솔 로그인 후 **[로그인] > [카카오계정 ID] > [계정 설정] > [본인인증]**. (개인 개발자는 OWNER 계정 본인인증 후 비즈앱 전환 가능.)
   - 비즈앱 전환 경로: 콘솔(https://developers.kakao.com/console/app)에서 앱 선택 → **[앱 설정] > [앱] > [일반]** 의 비즈니스 정보에서 **[사업자 정보 등록]**(비즈니스 통합 서비스 약관 동의 필요).
2. **사용 권한 승인.** 콘솔(https://developers.kakao.com/console/app)에서 앱 선택 → **[앱] > [추가 기능 신청]** 에서 카카오모먼트 오픈API 사용 권한을 신청·승인받아야 함.
3. **비즈니스 토큰 사전 발급**(위 발급 절차).

- 플랜/요금제 요건: 문서에 명시 없음 → "없음(미확인)".
- 발신번호 사전등록: 해당 없음("없음"). (메시지광고는 있으나 1차 범위 밖)

> 승인 심사 기준·소요기간·일반 광고주 단독 승인 가능 여부는 카카오 문서에 명시되어 있지 않다(미확인). 도입 전 카카오에 확인 필요.

## 스코프 / 권한
정식 스코프 4종(출처: business-auth/common):

| 스코프 | 용도 | 1차 범위 |
|--------|------|----------|
| `moment_management` | 운영/조회·상태변경 | **사용**(조회·보고서·ON·OFF·일예산) |
| `moment_create` | 광고계정/소재 등 생성 | 미사용(2차 보류) |
| `moment_delete` | 삭제 | 미사용(2차 보류) |
| `moment_bizform_result_read` | 비즈폼+ 응답 조회 | 미사용(2차 보류) |

- `moment_create`/`keyword_create` 포함 시 인가 요청에 `resource_ids=moment_create:{adAccountId}` 필수.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `KAKAO_MOMENT_ACCESS_TOKEN` | 발급 절차 3단계 응답의 `access_token`(비즈니스 토큰) | O |
| `KAKAO_MOMENT_AD_ACCOUNT_ID` | 대상 광고계정 번호. `list_ad_accounts`(GET `/openapi/v4/adAccounts/pages`) 응답 `content[].id`로 확인 가능 | O |
| `KAKAO_MOMENT_BASE_URL` | API 베이스(기본 `https://apis.moment.kakao.com`) | X |
| `KAKAO_MOMENT_RATE_LIMIT_MS` | 호출 최소간격 가드 하한(기본 1000ms) | X |

> 발급 전용(서버 런타임 불필요) — 헬퍼 스크립트를 둘 경우에만: `KAKAO_REST_API_KEY`(client_id), `KAKAO_CLIENT_SECRET`, `KAKAO_REDIRECT_URI`.

## 출처 (공식 URL)
- 카카오 개발자 콘솔 앱 관리 페이지(키·신청 진입점): https://developers.kakao.com/console/app
- 비즈니스 인증(인가코드·토큰 교환, REST API 키/리다이렉트 URI/client_secret 위치 명시): http://developers.kakao.com/docs/ko/business-auth/rest-api
- 비즈니스 인증 공통(스코프·refresh 없음): http://developers.kakao.com/docs/ko/business-auth/common
- 카카오모먼트 레퍼런스(엔드포인트·베이스 URL): http://developers.kakao.com/docs/ko/kakaomoment/reference
- 카카오모먼트 캠페인(헤더·인증): http://developers.kakao.com/docs/ko/kakaomoment/campaign
- 카카오모먼트 공통(사용 권한 신청 경로·`AssetGroup` 폐기): http://developers.kakao.com/docs/ko/kakaomoment/common
- 카카오모먼트 에러코드: http://developers.kakao.com/docs/ko/kakaomoment/error-code
- (참고) 공식 Claude 커넥터 부재: https://support.claude.com/en/articles/11176164-pre-built-web-connectors-using-remote-mcp

## 미확인 / 주의
- **사용 권한 승인 심사 기준·소요기간·일반 광고주 단독 승인 가능 여부**: 문서 명시 없음 → 미확인. 도입 전 카카오 확인.
- **비즈니스 토큰 정확한 만료 시간(시간/일)**: 명시 없음 → 미확인. 런타임 401 발생 시 재인가(재발급) 안내로 가드.
- **토큰 자동 갱신 불가**: refresh_token 없음. 만료 시 사람이 발급 절차 재수행 필요.
- **콘솔 URL(개발자 콘솔 화면 주소)**: 확인됨 — 앱 관리 페이지 공통 진입점은 https://developers.kakao.com/console/app (출처: business-auth/rest-api, kakaomoment/common). 단 개별 앱·키·신청 화면은 워크스페이스/앱별 URL이라 보편 직링크 없음 → 위 콘솔에서 앱 선택 후 메뉴 경로([앱]>[플랫폼 키]>[REST API 키], [앱]>[추가 기능 신청])로 이동.
- **폐기 필드 금지**: 소재 `AssetGroup` 필드는 2025-09-30 이후 사용 불가(`slides`로 대체). 1차 범위(소재 생성/수정)에는 포함되지 않으나 향후 구현 시 주의.
- 시크릿(토큰)은 코드/`.mcp.json`에 하드코딩 금지. 모두 env로 주입.
