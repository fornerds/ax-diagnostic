# 키 발급 가이드 — Shopify (shopify)

> 출처 기반(검증본 `_workspace/shopify/02_validation.md` / `README.md`). 추측 없음. 미확인 항목은 "미확인 / 주의"에 명시.

## 연동 유형 / 인증 방식
- **연동 유형:** 커스텀앱(custom app) — 단일 스토어 전용. (퍼블릭앱 OAuth 불요)
- **인증 방식:** 커스텀앱 **Admin API 액세스 토큰**(단일 스토어, 오프라인 토큰). 신규 Dev Dashboard 앱은 **Client ID/Secret → client credentials grant**로 토큰 교환(24h 만료, 갱신).
- **요청 인증:** 모든 GraphQL 요청 헤더에 `X-Shopify-Access-Token: {토큰}` + `Content-Type: application/json`.
- **엔드포인트(POST):** `https://{SHOPIFY_STORE_DOMAIN}/admin/api/{SHOPIFY_API_VERSION}/graphql.json` (REST 미사용 — 레거시).
- **API 버전:** `2026-04`(검증 시점 최신). 분기 버전은 약 1년 후 지원 종료 → 환경변수로 갱신 가능하게 핀.
- 출처: https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/generate-app-access-tokens-admin · https://shopify.dev/docs/api/admin-graphql

## 발급 절차 (스텝)

> **콘솔 진입 URL:** Dev Dashboard — **https://dev.shopify.com/dashboard/**
> (콘솔은 조직·앱·스토어별로 동적 생성되어 **발급 화면 개별 직링크 없음** → 아래 메뉴 경로로 진입.)
>
> **주의 1:** Shopify **Admin UI에서 신규 커스텀앱 생성은 더 이상 불가**하다(공식 원문: "You can no longer create new custom apps in the Shopify admin. Existing admin-created custom apps continue to work."). 신규는 **Dev Dashboard** 또는 **Shopify CLI**로 생성한다.
> **주의 2 (중요·변경됨):** Dev Dashboard로 만든 앱은 **액세스 토큰이 UI에 더 이상 노출되지 않는다**. 대신 **Client ID + Client secret**을 발급받아 **클라이언트 자격증명 그랜트(client credentials grant)**로 토큰을 **프로그램 방식으로 교환**한다. 발급된 토큰은 **24시간(`expires_in=86399`) 만료** → 코드에서 캐시·갱신.
> (참고: "설치 시 토큰 1회 노출 → `Reveal token once`"는 **2026-01-01 이전 Admin UI에서 만든 레거시 커스텀앱에만** 해당. 신규 빌드에는 적용 안 됨.)

### A. 신규 — Dev Dashboard 경로 (권장)
1. **콘솔 진입:** https://dev.shopify.com/dashboard/ 접속.
2. **커스텀앱 생성:** 좌측 내비게이션 **`Apps`** > 우측 상단 **`Create app`** > **`Start from Dev Dashboard`** > 앱 이름 입력 > **`Create`** 클릭.
3. **Admin API 스코프 설정:** 앱의 **`Versions`** 탭 > 앱 **scopes** 입력란에 사용할 스코프 지정(아래 "스코프 / 권한" 참조. 60일 초과 주문 분석 필요 시 `read_all_orders` 포함 시도) > **`Release`** 클릭.
4. **스토어에 설치:** 좌측 내비게이션 **`Home`** > 하단으로 스크롤 **`Install app`** > 대상 스토어 선택(또는 생성) > **`Install`** 클릭.
5. **자격증명 확보:** 좌측 내비게이션 **`Settings`** > **`Client ID`** + **`Client secret`** 복사(`.env`·`.gitignore` 보관, 커밋 금지).
6. **토큰 교환(프로그램):** 다음 엔드포인트에 POST하여 액세스 토큰을 받는다.
   - **엔드포인트:** `https://{SHOPIFY_STORE_DOMAIN}/admin/oauth/access_token`
   - **헤더:** `Content-Type: application/x-www-form-urlencoded`
   - **바디:** `grant_type=client_credentials` · `client_id={Client ID}` · `client_secret={Client secret}`
   - **응답:** `access_token`(→ `SHOPIFY_ACCESS_TOKEN`), `scope`, `expires_in`(=86399, 24h). 만료 시 재요청으로 갱신.
7. **스토어 도메인 설정:** 스토어 도메인(`*.myshopify.com`)을 `SHOPIFY_STORE_DOMAIN`에 설정.

### B. (레거시) Admin UI에서 만든 기존 커스텀앱만 해당
> 신규 생성 불가. 이미 Admin에서 만든 앱이 있을 때만 적용. 새 토큰이 필요하면 **앱을 제거(uninstall) 후 재설치(reinstall)**해야 함(공식 원문: "you need to uninstall and reinstall your app").
> 레거시 경로: Shopify Admin > **`Settings`** > **`Apps and sales channels`** > **`Develop apps`** > 해당 앱 선택 > **`API credentials`** 탭 > **`Admin API access token`** 섹션 > **`Reveal token once`**(1회만 표시).

- 출처: https://shopify.dev/docs/apps/build/dev-dashboard/create-apps-using-dev-dashboard · https://shopify.dev/docs/apps/build/dev-dashboard/get-api-access-tokens · https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/client-credentials-grant · https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/generate-app-access-tokens-admin

## 전제조건
- **플랜:** 별도 제한 없음(글로벌 SaaS). 단, 레이트리밋 한도는 플랜별 상이 — Standard 100 / Advanced 200 / Plus 1000 / Enterprise 2000 (점/초).
- **사업자 심사:** 없음. 한국 특화 규제(사업자/공동인증서 등) 없음. 단일 스토어 커스텀앱은 심사 불요(스토어 관리자 권한이면 발급).
- **발신번호 사전등록 등:** 없음(해당 없음).
- **스태프 권한:** "Manage and install apps and channels" + "Develop apps" + 리소스별 권한 필요.
- **(조건부) 60일 초과 주문 접근:** 기본 `read_orders`로는 최근 60일 주문만 조회 가능. 그 이전은 `read_all_orders` 스코프 필요 — 이 스코프는 **Shopify 승인(심사)** 대상. (커스텀앱 단일스토어에 자동 부여되는지는 미확인 — 아래 참조)
- 출처: https://shopify.dev/docs/apps/launch/protected-customer-data · https://shopify.dev/docs/api/usage/limits · https://shopify.dev/changelog/apps-now-need-shopify-approval-to-read-orders-older-than-60-days

## 스코프 / 권한
도구별 필요 스코프(Admin API access scopes):

| 용도 | 스코프 |
|------|--------|
| 주문 조회/분석 | `read_orders` (60일 초과는 `read_all_orders` 추가) |
| 주문 수정 | `write_orders` |
| 상품 조회 | `read_products` |
| 상품 생성·수정 | `write_products` |
| 고객 조회 | `read_customers` |
| 재고 조정 | `write_inventory` (+조회 시 `read_inventory`) |
| 임시(견적)주문 | `write_draft_orders` |
| 주문 이행(배송) | `write_fulfillments` (+`read_fulfillments`) |

- 보호고객데이터: 커스텀앱(단일 스토어)은 항상 접근 가능(심사 불요).
- 출처: https://shopify.dev/docs/api/usage/access-scopes

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `SHOPIFY_STORE_DOMAIN` | 대상 스토어 도메인(예: `your-store.myshopify.com`) — Shopify Admin / 스토어 설정 | 필수 |
| `SHOPIFY_ACCESS_TOKEN` | Admin API 액세스 토큰(`X-Shopify-Access-Token`). 신규 Dev Dashboard 앱: Settings의 Client ID/Secret로 `…/admin/oauth/access_token`에 client_credentials POST → `access_token`(24h, 갱신). 레거시 Admin 앱: `Reveal token once`로 1회 노출 | 필수 |
| `SHOPIFY_API_VERSION` | API 버전 핀(기본 `2026-04`) | 선택(기본값 제공) |

## 출처 (공식 URL)
- Dev Dashboard 콘솔 진입: https://dev.shopify.com/dashboard/
- Dev Dashboard 앱 생성(메뉴 경로 Apps>Create app>Start from Dev Dashboard>Create, 설치 Home>Install app>Install): https://shopify.dev/docs/apps/build/dev-dashboard/create-apps-using-dev-dashboard
- Dev Dashboard 토큰 취득(Settings>Client ID/secret → client credentials grant, 24h): https://shopify.dev/docs/apps/build/dev-dashboard/get-api-access-tokens
- client credentials grant(엔드포인트 `…/admin/oauth/access_token`, grant_type=client_credentials): https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/client-credentials-grant
- 액세스 토큰 발급(Admin UI 신규 불가 + 레거시 Reveal token once): https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/generate-app-access-tokens-admin
- 액세스 스코프 레퍼런스: https://shopify.dev/docs/api/usage/access-scopes
- Admin GraphQL API(엔드포인트·버전): https://shopify.dev/docs/api/admin-graphql
- 레이트리밋/플랜 한도: https://shopify.dev/docs/api/usage/limits
- 보호고객데이터: https://shopify.dev/docs/apps/launch/protected-customer-data
- 60일 주문 제한 변경공지: https://shopify.dev/changelog/apps-now-need-shopify-approval-to-read-orders-older-than-60-days

## 미확인 / 주의
- **`read_all_orders` 자동 부여 여부 (미확인):** 변경공지는 "레거시 private app은 자동으로 `read_all_orders`를 가진다"고 명시하나, 현행 **custom app**(단일 스토어)이 이를 승계하는지는 공식 명문으로 확정되지 않음. → 발급 시점에 스코프 포함 가능 여부를 직접 확인. 60일 경계 질의가 0건이면 권한/기간 제한일 수 있음.
- **토큰 취득 방식이 앱 생성 경로에 따라 다름:** 신규 **Dev Dashboard 앱**은 토큰이 UI에 노출되지 않고 **Client ID/Secret로 client credentials grant** 교환(24h 만료 → 코드에서 갱신). **레거시 Admin UI 앱**만 `Reveal token once`로 1회 노출. 어느 쪽이든 토큰·Secret은 즉시 안전 보관, `.env`·토큰 절대 커밋 금지(`.gitignore`).
- **버전 만료:** 분기 버전(`2026-04`)은 약 1년 후 지원 종료. `SHOPIFY_API_VERSION` 환경변수로 갱신 가능하게 유지.
- **온라인 토큰 제약:** 커스텀앱 토큰은 오프라인 토큰이므로 본 서버 용도에 적합. 온라인(per-user) 토큰 사용 시 `orderCreate` 등 일부 작업 불가(단, 본 서버는 `orderCreate` 제외).
