# 키 발급 가이드 — Redash (redash)

> 본 가이드는 검증본(`_workspace/redash/02_validation.md`, verdict=spec_ready, 검증 기준일 2026-06-26)과 `README.md`의 출처 있는 사실만 정리했다. 추측 금지. UI 클릭 경로 등 출처에서 확정되지 않은 디테일은 "미확인"으로 표기한다.

## 연동 유형 / 인증 방식

- 연동 유형: **REST API + API Key** (OAuth2/GraphQL/공식 SDK 없음). Redash의 OAuth(Google 등)는 사용자 로그인용일 뿐 API 호출 인증에는 쓰이지 않는다. API 호출 인증은 **API Key 단일**.
- 키 종류 2가지:
  - **User API Key** — 키 소유 사용자의 권한 범위 전체를 그대로 가진다(그 사용자가 보는 데이터소스/쿼리/대시보드).
  - **Query API Key** — 특정 단일 쿼리와 그 결과에만 한정. 더 안전. 공식 가이드 원문 권고: "Whenever possible we recommend using a Query API key".
- 키 전달(서버 채택 방식): HTTP 헤더 `Authorization: Key <REDASH_API_KEY>` — 접두사 `Key ` 고정(끝 공백 포함). **Bearer 아님.**
  - 대안으로 쿼리스트링 `?api_key=<KEY>`도 인스턴스가 지원하나, 본 서버는 로그/URL 노출 방지를 위해 헤더 방식만 사용.
- 베이스 URL: 인스턴스별 가변 `{REDASH_HOST}` (예: `https://redash.example.com`). 모든 경로는 `{REDASH_HOST}/api/...`. **버전 prefix 없음**(`/api/v1` 등 없음). 공식 가이드 기준 "as of V9".

## 발급 절차 (스텝)

사전 교환 단계(앱 등록·OAuth 동의·토큰 발급)가 없다. 인스턴스에 로그인한 사용자가 즉시 키를 확인/발급한다.

### A. User API Key (사용자 권한 전체)
1. 셀프호스트 Redash 인스턴스(`{REDASH_HOST}`)에 로그인한다.
2. **우상단 사용자(아바타) 메뉴 → "Edit Profile"** 을 클릭해 본인 프로필 페이지로 이동한다. (직링크: `{REDASH_HOST}/users/me` — 일부 버전은 `{REDASH_HOST}/users/me#apiKey` 로 API Key 섹션 직행 가능. 워크스페이스/인스턴스별 호스트가 가변이므로 글로벌 단일 직링크는 없음.)
   - 대안 경로: **Settings → Profile**(인스턴스 메뉴 구성에 따라). 두 경로 모두 동일한 프로필 페이지로 간다.
3. 프로필 페이지의 **"API Key"** 항목에 표시된 값을 복사한다. 필요 시 같은 화면의 **"Regenerate"** 버튼으로 키를 회전한다.
   - 메뉴 항목명("Edit Profile") 및 `/users/me` 경로는 버전별 라벨/테마 차이가 있을 수 있으나, 핵심은 우상단 사용자 메뉴 → 프로필 페이지의 "API Key" 항목이다.

### B. Query API Key (단일 쿼리 한정, 권장)
1. 인스턴스에 로그인한다.
2. 대상 쿼리를 연다(`{REDASH_HOST}/queries/<query_id>`). 워크스페이스/쿼리별로 경로가 달라 글로벌 단일 직링크는 없음 — 호스트 + 해당 쿼리 ID로 직행한다.
3. 쿼리 화면 **우측 상단의 쿼리(점 3개/케밥) 메뉴 → "Show API Key"** 를 클릭한다. 표시된 모달/패널에 그 쿼리 전용 API Key(및 결과 다운로드 URL)가 나온다.
4. 표시된 Query API Key를 복사한다. 이 키는 그 쿼리·결과에만 접근 가능(최소권한).
   - 메뉴 라벨("Show API Key")은 버전별 차이가 있을 수 있으나, 쿼리 메뉴 안의 API Key 노출 항목을 찾는다.

> 둘 다 사용 시 동일하게 `Authorization: Key <값>` 헤더로 전달한다. 본 MCP 서버는 다수 도구(쿼리 목록/대시보드/데이터소스 등 광범위 작업)를 노출하므로 실무상 User API Key가 현실적이나, **최소권한 사용자 계정의 키**를 쓰는 것을 권장한다.

## 전제조건

- 플랜/구독: **없음** (오픈소스 셀프호스트). 유료 플랜 불필요.
- 사업자 심사 / 제휴 승인 / 발신번호 사전등록: **없음** (출처: 공식 API 가이드 — 로그인 사용자가 즉시 발급).
- **셀프호스트 Redash 인스턴스가 반드시 필요.** 관리형 `app.redash.io`는 **2021-11-30 종료**(Hosted Redash EOL)되어 신규 가입 불가·데이터 삭제됨. 따라서 대상은 셀프호스트 전용. (출처: https://redash.io/help/faq/eol/)
- 관리자 전용 작업(`/api/users*` 등)을 호출하려면 **admin 권한 사용자의 키** 필요(비-admin은 403). 단, 본 서버는 사용자 관리(admin) 도구를 1차 범위에서 제외했으므로 일반 키로 충분.

## 스코프 / 권한

- 별도 OAuth 스코프 개념 없음. 권한은 **키 소유 사용자의 권한에 종속**.
  - User API Key = 그 사용자가 접근 가능한 데이터소스/쿼리/대시보드 전체.
  - Query API Key = 해당 단일 쿼리와 그 결과로 한정(최소권한).
- 관리자 전용 작업은 admin 키 필요(비-admin 403). (출처: 권한 모델 — https://www.stitchflow.com/user-management/redash/api , 소스 `redash/authentication/__init__.py`)
- 파괴적 작업 주의: `archive_query`(DELETE `/api/queries/<id>`), `archive_dashboard`(DELETE `/api/dashboards/<id>`)는 대상을 아카이브한다. 해당 권한이 있는 키로만 동작.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `REDASH_HOST` | 셀프호스트 인스턴스 베이스 URL (예: `https://redash.example.com`). 말미 슬래시는 서버가 자동 제거. 모든 경로는 `{REDASH_HOST}/api/...` | 예 |
| `REDASH_API_KEY` | 위 "발급 절차"에서 복사한 User API Key(프로필 페이지) 또는 Query API Key(쿼리 페이지). `Authorization: Key <값>` 헤더로 전송 | 예 |
| `REDASH_TIMEOUT_SEC` | 일반 HTTP 요청 타임아웃(초). 기본 30 | 아니오 |
| `REDASH_POLL_TIMEOUT_SEC` | 비동기 잡 폴링 최대 대기(초). 기본 60. `run_query_and_wait`의 `timeout_sec` 기본값 | 아니오 |

## 출처 (공식 URL)

- 공식 API 가이드(인증·API Key 종류·발급 위치, "as of V9"): https://redash.io/help/user-guide/integrations-and-api/api/
- UI 발급 동선(우상단 Users → Edit Profile에서 User API Key; 쿼리 메뉴 → "Show API Key"에서 Query API Key) — Redash v5 Quick Start Guide(O'Reilly): https://www.oreilly.com/library/view/redash-v5-quick/9781788996167/90ce460a-d9ad-4240-bc73-30cdd20ef6c3.xhtml
- 프로필 경로(Settings → Profile / `/users/me`에서 "API Key" 복사): https://www.stitchflow.com/user-management/redash/api
- 키 전달 방식(헤더 `Authorization: Key `, 쿼리스트링 `api_key`, 추출 우선순위) — 공식 소스: https://raw.githubusercontent.com/getredash/redash/master/redash/authentication/__init__.py
- REST 라우팅(사실상의 사양): https://raw.githubusercontent.com/getredash/redash/master/redash/handlers/api.py
- 관리형 호스팅 종료(Hosted Redash EOL = 2021-11-30): https://redash.io/help/faq/eol/
- 권한 모델(사용자 권한 종속, admin 전용 작업): https://www.stitchflow.com/user-management/redash/api

## 미확인 / 주의

- **UI 클릭 경로(확인됨, 버전별 라벨 차이 가능)**: User API Key = 우상단 사용자 메뉴 → "Edit Profile"(또는 Settings → Profile, `{REDASH_HOST}/users/me`) → 프로필의 "API Key" 항목. Query API Key = 쿼리 페이지(`/queries/<id>`)의 쿼리(케밥) 메뉴 → "Show API Key". (출처: O'Reilly Redash v5 Quick Start, Stitchflow) 메뉴 라벨/경로는 인스턴스 버전·테마에 따라 미세 차이가 있을 수 있다. 직링크는 호스트·쿼리 ID가 가변이라 글로벌 단일 직링크는 없으며, 위 패턴으로 직행한다.
- **인스턴스 버전 차이**: 공식 가이드 기준 V9. 인스턴스 버전에 따라 일부 동작 차이 가능(특히 대시보드 페이지네이션 `page>=2` 알려진 이슈: https://github.com/getredash/redash/issues/3694).
- **레이트리밋 공식 사양 부재**: API 레이트리밋 공식 명세 없음(인스턴스/프록시 레벨 가변). 429 수신 시 `Retry-After` 존중 + 재시도.
- **에러 응답 바디 정형 명세 부재**: 실패 시 `{"message": ...}` 패턴 관찰되나 공식 정형 명세 없음. HTTP 상태코드(401/403/404/400)는 신뢰, 구조화 파싱은 best-effort.
- **공식 OpenAPI/Swagger 미발견**: 기계판독 스펙은 공식 제공 없음(라우팅/스키마는 소스가 기준). 커뮤니티 문서(koooge.github.io/redash-api-doc)는 비공식.
