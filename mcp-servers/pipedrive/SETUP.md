# 키 발급 가이드 — Pipedrive (pipedrive)

> 검증 출처: `_workspace/pipedrive/02_validation.md`(검증일 2026-06-26), `mcp-servers/pipedrive/README.md`
> 보강: API 토큰 UI 경로는 공식 문서 1회 재확인(2026-06, web_checked)으로 해소.

## 연동 유형 / 인증 방식

Pipedrive 공개 REST API(v2 표준 + 일부 v1) 연동. 인증은 두 방식 중 택1.

- **방식 A — API 토큰 (권장, 단일 테넌트):** 모든 요청에 `x-api-token: {API_TOKEN}` 헤더. 베이스URL = `https://{회사도메인}.pipedrive.com`. 환경변수 2개로 동작 — 최단 경로.
- **방식 B — OAuth 2.0 (멀티테넌트/마켓플레이스):** `Authorization: Bearer {access_token}`. 베이스URL = 토큰 응답의 `api_domain` 필드값. 본 서버는 토큰 교환/리프레시를 내장하지 않으므로 액세스 토큰을 외부에서 발급해 주입.
- 출처: https://pipedrive.readme.io/docs/core-api-concepts-authentication , https://pipedrive.readme.io/docs/marketplace-oauth-authorization

## 발급 절차 (스텝)

### 방식 A — API 토큰
1. Pipedrive 웹앱에 로그인.
2. 우측 상단 **계정 이름(account name)** 클릭 → **Company settings** → **Personal preferences** → **API**.
   (출처: https://pipedrive.readme.io/docs/how-to-find-the-api-token , 2026-06 재확인)
3. 표시되는 개인 **API token** 값을 복사 → `PIPEDRIVE_API_TOKEN`.
4. 브라우저 주소창의 회사 서브도메인 확인 → `PIPEDRIVE_COMPANY_DOMAIN`.
   예: 주소가 `https://acme.pipedrive.com` 이면 값은 `acme`.

> 주의: 사용자당 활성 API 토큰은 **1개**다. 재발급/교체하면 기존 연동이 즉시 끊긴다(검증 #3). 사용자는 회사마다 별도 토큰을 가진다.

### 방식 B — OAuth 2.0 (외부에서 토큰 발급)
1. Developer Hub에서 앱 등록(private/단일계정 앱은 심사 없이 즉시 가능 — 검증 #58/누락섹션). `client_id`/`client_secret` 확보.
2. 인가: `GET https://oauth.pipedrive.com/oauth/authorize?client_id=...&redirect_uri=...&state=...`
3. 토큰 교환: `POST https://oauth.pipedrive.com/oauth/token` + `Authorization: Basic base64(client_id:client_secret)`, body `grant_type=authorization_code&code=...&redirect_uri=...` (authorization_code는 **5분 만료** — 검증 #6).
4. 갱신: 동일 엔드포인트, `grant_type=refresh_token&refresh_token=...`.
5. 응답의 `access_token` → `PIPEDRIVE_ACCESS_TOKEN`, `api_domain` → `PIPEDRIVE_API_DOMAIN`.
   (`api_domain` = "베이스URL path, including the company_domain" — 추측 불필요, 이 필드 사용. 검증 #7)

## 전제조건

- **사업자 심사·제휴 승인 불필요.** 글로벌 SaaS로 한국식 제약(공동인증서·발신번호 사전등록 등) **없음**(검증 누락섹션).
- 플랜: API 자체는 모든 유료 플랜에서 사용 가능. 단 **레이트리밋 한도는 플랜 배수에 비례**(Lite 1 / Growth 2 / Premium 5 / Ultimate 7 — 검증 #26). 발급 자체에는 특정 플랜 강제 조건 미확인.
- 방식 B의 **마켓플레이스 public 앱 배포 심사** 절차는 미확인이나, **private/단일테넌트 앱은 심사 없이** 가능(검증 #38). 단일 회사 사용이면 방식 A로 전제조건 사실상 없음.

## 스코프 / 권한

- **방식 A(API 토큰):** 별도 스코프 선택 없음 — 토큰 소유 사용자의 권한을 그대로 따른다.
- **방식 B(OAuth) 필수 스코프(1차 도구 31개 기준):**
  `base`, `deals:read`, `deals:full`, `contacts:read`, `contacts:full`, `activities:read`, `activities:full`, `leads:read`, `leads:full`, `search:read`.
  - 파이프라인/단계 조회는 `deals:read` 범위에 포함 — 별도 스코프 불필요.
  - 확장(Products 등) 시 `products:*` 추가.
  - 출처: https://pipedrive.readme.io/docs/marketplace-scopes-and-permissions-explanations

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `PIPEDRIVE_API_TOKEN` | 웹앱 계정명 > Company settings > Personal preferences > API 의 토큰값 | 방식 A 필수 |
| `PIPEDRIVE_COMPANY_DOMAIN` | 웹앱 주소 `https://{이값}.pipedrive.com` 의 서브도메인 | 방식 A 필수 |
| `PIPEDRIVE_ACCESS_TOKEN` | OAuth 토큰 응답의 `access_token` | 방식 B 필수 |
| `PIPEDRIVE_API_DOMAIN` | OAuth 토큰 응답의 `api_domain`(베이스URL) | 방식 B 필수 |
| `PIPEDRIVE_CLIENT_ID` | Developer Hub 앱의 client id | 방식 B(외부 토큰 교환·리프레시 시) |
| `PIPEDRIVE_CLIENT_SECRET` | Developer Hub 앱의 client secret | 방식 B(외부 토큰 교환·리프레시 시) |

> 시크릿 하드코딩 금지. `.env`/환경변수로만 주입(`.env`는 커밋 금지). 방식 A가 설정돼 있으면 서버가 방식 A를 우선 사용.

## 출처 (공식 URL)

- 인증(API 토큰 / x-api-token / 활성 토큰 1개): https://pipedrive.readme.io/docs/core-api-concepts-authentication
- API 토큰 찾는 법(UI 경로): https://pipedrive.readme.io/docs/how-to-find-the-api-token
- OAuth 2.0 인가/토큰 교환/갱신: https://pipedrive.readme.io/docs/marketplace-oauth-authorization
- OAuth 스코프/권한: https://pipedrive.readme.io/docs/marketplace-scopes-and-permissions-explanations
- 레이트리밋(플랜 배수): https://pipedrive.readme.io/docs/core-api-concepts-rate-limiting

## 미확인 / 주의

- **마켓플레이스 public 앱 배포 심사 절차(#38):** 미확인. 단일테넌트 빌드(방식 A)와는 무관.
- **방식 A 우선 권장:** 단일 회사·단일 사용자라면 API 토큰 2개 변수만으로 즉시 동작. OAuth는 토큰스토어/갱신 운영이 추가로 필요.
- **토큰 교체 위험:** API 토큰 재발급 시 기존 연동 전부 중단(활성 토큰 1개 제약).
- **에러 바디 필드명(#36):** 공식 미확정 — 구현은 방어적 파싱. 키 발급 자체와는 무관.
