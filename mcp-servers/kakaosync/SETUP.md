# 키 발급 가이드 — 카카오싱크 (kakaosync)

> 이 서버는 **이미 발급된 사용자 액세스 토큰(또는 refresh_token)을 주입받아** 회원/동의 정보를 조회·관리(읽기)하는 범위입니다.
> 신규 로그인/최초 토큰 발급(`/oauth/authorize` → `/oauth/token` authorization_code 플로우)은 최종사용자 브라우저 리다이렉트 + `redirect_uri` 콜백 웹서버가 필요하므로 본 MCP 서버 밖에서 수행합니다.

## 연동 유형 / 인증 방식

- **연동 유형**: OAuth 2.0 Authorization Code Grant + Kakao REST API.
- **호출 인증 헤더**:
  - 기본: `Authorization: Bearer ${KAKAO_ACCESS_TOKEN}` (사용자 토큰)
  - 서버-서버 조회가 필요한 경우에 한해: `Authorization: KakaoAK ${KAKAO_ADMIN_KEY}` (어드민 키 — 민감, 미사용 권장)
- **베이스 URL**:
  - 인가/토큰: `https://kauth.kakao.com` (`/oauth/authorize`, `/oauth/token`)
  - API: `https://kapi.kakao.com` (버전은 경로 prefix, 예 `/v2/user/me`)
- **토큰 수명**: 액세스 토큰 약 12시간(43,199초), 리프레시 토큰 60일(5,184,000초). 액세스 만료 시 refresh_token으로 갱신.

## 발급 절차 (스텝)

1. **Kakao Developers 콘솔 접속** — 앱 관리 페이지 `https://developers.kakao.com/console/app` 에서 애플리케이션을 생성(또는 기존 앱 선택).
2. **REST API 키 확인** — [앱] > [플랫폼 키] > [REST API 키]. 이 값이 OAuth `client_id`(= `KAKAO_REST_API_KEY`)입니다.
3. **(조건부) 클라이언트 시크릿 확인** — [앱] > [플랫폼 키] > [REST API 키] > [클라이언트 시크릿]. 보안(client_secret) 사용이 ON이면 토큰 발급/갱신에 함께 전송해야 합니다(= `KAKAO_CLIENT_SECRET`).
4. **리다이렉트 URI 등록** — [앱] > [플랫폼 키] > [REST API 키] > [리다이렉트 URI]에 OAuth 콜백 URL 등록(최초 로그인/토큰 발급 시 필요).
5. **카카오 로그인 활성화 + 동의항목 설정** — [카카오 로그인] > [사용 설정] ON → [동의항목]에서 필요한 [개인정보]/[접근권한] 항목을 [설정]으로 구성.
6. **(카카오싱크 전용 기능 사용 시) 비즈 앱 전환 + 비즈니스 채널 심사 + 앱-채널 연결** — 아래 "전제조건" 참고.
7. **사용자 토큰 발급(서버 밖)** — 등록한 redirect_uri로 인가코드를 받아 `POST kauth.kakao.com/oauth/token`(grant_type=authorization_code)으로 액세스 토큰·리프레시 토큰을 발급. 발급된 토큰을 본 MCP 서버에 env 또는 도구 인자로 주입.

## 전제조건

> **개인/미사업자는 카카오싱크 고유 기능을 사용할 수 없습니다.** 일반 카카오 로그인 기본 동의항목만 사용한다면 비즈앱 없이도 가능.

카카오싱크 고유 기능(간편가입 동의창·서비스약관·채널추가)은 **비즈 앱 + 비즈니스 채널**이 필수입니다.

- **비즈니스 채널 심사**: 영업일 기준 3~5일
- **앱-채널 연결 심사**: 영업일 기준 3~5일 (앱·채널의 사업자번호 일치 검증)
- **누적 리드타임**: 최소 6~10영업일 + 추가 동의항목 심사
- **제출물**: 사업자등록증, 대표 신분증 등 (비즈니스 채널/정보 심사용)
- **민감 동의항목(CI·실명·전화번호·배송지)**: 추가 "비즈니스 정보 심사" 통과 전까지 응답에 내려오지 않음 → 해당 도구는 `*_needs_agreement: true` 또는 빈 값 반환(구현에서 가드 필요).
- **권한**: 앱 = Owner 또는 Editor / 채널 = 매니저(Manager) 또는 마스터(Master).
- **레이트리밋/쿼터**: 액세스 토큰 발급 20회/10분, 리프레시 토큰 발급 30회/60분(사용자당). 앱 단위 월 한도 3,000,000 requests/month(무료 공유 풀, 사용자정보/동의 조회는 이 풀에서 차감).

## 스코프 / 권한

도구별 필요 동의항목(scope):

| 데이터 | scope | 비고 |
|--------|-------|------|
| 프로필(닉네임·이미지) | `profile_nickname` / `profile_image` | 기본 동의항목 |
| 이메일 | `account_email` | |
| 전화번호 | `phone_number` | 비즈앱 + 추가 심사 전제 |
| 실명 | `name` | 비즈앱 + 추가 심사 전제 |
| CI(연계정보) | `account_ci` | 비즈앱 + 추가 심사 전제 |
| 배송지 | `shipping_address` | 비즈앱 + 추가 심사 전제 |

- 동의항목은 콘솔 [카카오 로그인] > [동의항목]에서 활성화/설정.
- 민감 항목은 비즈 심사 통과 전까지 응답에서 `*_needs_agreement`로 표시되고 값이 비어 있을 수 있음.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `KAKAO_REST_API_KEY` | 콘솔 [앱] > [플랫폼 키] > [REST API 키] (= OAuth `client_id`) | refresh 사용 시 필수 |
| `KAKAO_CLIENT_SECRET` | 콘솔 [앱] > [플랫폼 키] > [REST API 키] > [클라이언트 시크릿] | 조건부(보안 client_secret ON 시 필수) |
| `KAKAO_ACCESS_TOKEN` | OAuth 토큰 발급(서버 밖, `POST kauth.kakao.com/oauth/token` grant_type=authorization_code) 결과의 `access_token` | 필수(또는 도구 인자 `access_token`으로 주입) |
| `KAKAO_REFRESH_TOKEN` | 위 토큰 발급 결과의 `refresh_token` | 선택(자동 갱신 사용 시 필수) |
| `KAKAO_ADMIN_KEY` | 콘솔 [앱] > [플랫폼 키] > [어드민 키] (서버-서버 조회용, 민감) | 선택(미사용 권장) |
| `KAKAO_AUTH_BASE_URL` | 코드 기본값 오버라이드(기본 `https://kauth.kakao.com`) | 선택 |
| `KAKAO_API_BASE_URL` | 코드 기본값 오버라이드(기본 `https://kapi.kakao.com`) | 선택 |

**인증 우선순위**: 도구 인자 `access_token` → `KAKAO_ACCESS_TOKEN`(Bearer) → (둘 다 없으면) `KAKAO_ADMIN_KEY`(KakaoAK).
**보안**: 모든 토큰/키는 env로만 주입. 코드·`.mcp.json`·로그에 토큰 평문 출력 금지.

## 출처 (공식 URL)

- 콘솔(앱 관리·키 확인): https://developers.kakao.com/console/app
- 카카오 로그인 REST API(인증·엔드포인트·스코프·토큰 수명·에러): https://developers.kakao.com/docs/ko/kakaologin/rest-api
- 카카오싱크 개발 가이드(동의항목·서비스약관): https://developers.kakao.com/docs/ko/kakaosync/dev-guide
- 쿼터/레이트리밋: https://developers.kakao.com/docs/ko/getting-started/quota
- 카카오싱크 도입(비즈앱·채널 심사·권한·심사 일수): https://kakaobusiness.gitbook.io/main/tool/kakaosync/phase-in

## 미확인 / 주의

- **CI/실명/전화 "비즈니스 정보 심사" 소요 일수**: 공식 문서에 일괄 수치 미명시. 심사 신청 화면에서 안내되는 값을 따를 것. — **미확인**
- **`service_terms` 조회 엔드포인트 버전 prefix**: 본 스펙은 `GET /v1/user/service_terms` 기준으로 확정했으나, 일부 캐시 페이지에서 `/v2/` 패턴이 추론됨. 첫 호출 전 라이브 레퍼런스로 1회 재확인 권고. — **주의**
- **client_secret 활성화 여부**: 콘솔에서 보안 client_secret을 ON으로 두었는지에 따라 `KAKAO_CLIENT_SECRET` 필수 여부가 달라짐. ON이면 토큰 발급/갱신에 반드시 포함.
- **에러 분기**: API(`kapi`) 응답 본문의 음수 `code` 기준(`-401` 토큰 만료, `-402` 동의 부족, `-10` 쿼터 초과, `-3` 앱키/권한 미허용). `KOExxx`(예 KOE237)는 인가(`kauth`) 단계 에러로 별개 체계. `msg` 텍스트 의존 금지.
