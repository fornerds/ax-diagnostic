# 키 발급 가이드 — Looker Studio (looker)

> 대상은 무료 셀프서비스 BI인 **Looker Studio**(구 Google Data Studio)다.
> 별개 제품인 엔터프라이즈 **Looker**(LookML, Google Cloud core/original)와 혼동 금지.
> API 호스트·스코프는 제품명이 Looker Studio여도 `datastudio` 이름을 그대로 쓴다.
> 검증 시점: 2026-06.

## 연동 유형 / 인증 방식

- **연동 유형**: 공식 원격 MCP 없음 → 공개 REST API(`https://datastudio.googleapis.com/v1`) 기반 로컬 stdio MCP 서버.
- **인증**: OAuth 2.0 Bearer 토큰(Google). 모든 호출에 `Authorization: Bearer <access_token>`.
- **토큰 흐름 (둘 중 하나)**:
  - **방식 A (권장)**: 도메인 전체 위임된 **서비스 계정**으로 조직 사용자를 임퍼스네이션해 토큰 발급.
  - **방식 B (대안)**: 표준 3-legged OAuth(설치형 동의)로 refresh token 확보.

## 발급 절차 (스텝)

### 방식 A — 서비스 계정 + 도메인 전체 위임 (권장)
1. **GCP 콘솔 접속** → https://console.cloud.google.com/ 에서 프로젝트 선택(또는 생성).
2. **API 사용 설정**: Looker Studio(Data Studio) API를 사용 설정한다.
3. **서비스 계정 생성 + JSON 키 발급**: IAM 및 관리자 → 서비스 계정에서 계정 생성 후 JSON 키를 내려받는다. 이 파일의 **절대 경로**가 `GOOGLE_APPLICATION_CREDENTIALS` 값이 된다.
4. **도메인 전체 위임 승인 (Workspace 관리자)**: Admin Console(https://admin.google.com) → 보안 → API 제어 → 도메인 전체 위임에서 서비스 계정의 **클라이언트 ID** + 스코프(아래 "스코프/권한")를 등록한다.
5. **임퍼스네이션 대상 지정**: 호출을 대행할 조직 사용자 이메일을 `LOOKER_STUDIO_SUBJECT`로 지정한다.

### 방식 B — 3-legged OAuth refresh token (대안)
1. **GCP 콘솔** → API 및 서비스 → 사용자 인증 정보에서 **OAuth 2.0 클라이언트 ID** 생성(클라이언트 ID/시크릿 확보).
2. 동의 흐름을 `access_type=offline`로 진행해 **refresh token** 확보.
3. 값 3개(`GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`, `GOOGLE_OAUTH_REFRESH_TOKEN`)를 환경변수로 주입.

## 전제조건

- **[필수] Google Workspace 또는 Cloud Identity 조직 계정.** 개인 무료 Gmail로는 인증 자체 불가(401/403). 원문: "only available to users that belong to an organization with Google Workspace or Cloud Identity".
- **[필수, 방식 A] 도메인 전체 위임 관리자 사전 승인.** Workspace 관리자가 Admin Console에서 클라이언트 ID + 스코프를 사전 등록해야 함.
- 사업자 심사·발신번호 사전등록 등 별도 심사: **없음**.
- 플랜 제약: Looker Studio 자체는 무료지만, 위 조직 계정 전제 때문에 개인 계정 사용자는 사용 불가.

## 스코프 / 권한

| 용도 | 스코프 |
|------|--------|
| 읽기(search, get_permissions) | `https://www.googleapis.com/auth/datastudio.readonly` |
| 쓰기(update, addMembers) | `https://www.googleapis.com/auth/datastudio` |
| 동의 흐름 보조(선택) | `https://www.googleapis.com/auth/userinfo.profile` |

- 토큰 1개로 통일 시 full `…/auth/datastudio`(읽기 포함) 사용. 최소권한을 원하면 readonly 분리.
- 읽기 도구는 readonly 스코프로 동작. 쓰기 도구(patch/addMembers)는 full `datastudio` 필요.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `GOOGLE_APPLICATION_CREDENTIALS` | 방식 A에서 발급한 서비스 계정 키 JSON 파일의 **절대 경로** | 방식 A 필수 |
| `LOOKER_STUDIO_SUBJECT` | 임퍼스네이션할 조직 사용자 이메일 (예: `user@your-domain.com`) | 방식 A 필수 |
| `GOOGLE_OAUTH_CLIENT_ID` | 방식 B의 OAuth 클라이언트 ID(GCP 사용자 인증 정보) | 방식 B 필수 |
| `GOOGLE_OAUTH_CLIENT_SECRET` | 방식 B의 OAuth 클라이언트 시크릿 | 방식 B 필수 |
| `GOOGLE_OAUTH_REFRESH_TOKEN` | 방식 B의 refresh token(`access_type=offline` 동의로 확보) | 방식 B 필수 |
| `LOOKER_STUDIO_SCOPE` | 사용할 스코프. 기본 `…/auth/datastudio`(쓰기 포함), 읽기 전용은 `…/auth/datastudio.readonly` | 선택 |
| `LOOKER_ENABLE_DESTRUCTIVE` | 파괴적 도구(`revoke_all_asset_permissions`) 활성화 게이트. 정확히 `true`일 때만 노출 | 선택 |

> 시크릿(키 파일·client secret·refresh token)은 **하드코딩 금지**, env로만 주입. `.env.example`에는 더미값만 둔다.

## 출처 (공식 URL)

- 인증·스코프·조직 전제·도메인 위임: https://developers.google.com/looker-studio/integrate/api
- API 레퍼런스(베이스 URL `…/v1`): https://developers.google.com/looker-studio/integrate/api/reference
- 타입(AssetType/Role/Permissions/Member): https://developers.google.com/looker-studio/integrate/api/reference/types
- changelog(2020-08-17 GA): https://developers.google.com/looker-studio/integrate/api/changelog
- (참고) Looker용 MCP — Looker Studio 대상 아님: https://docs.cloud.google.com/looker/docs/mcp

## 미확인 / 주의

- **레이트리밋·쿼터 수치**: `datastudio.googleapis.com` 전용 공개 쿼터 문서 없음. Cloud Console 프로젝트 쿼터가 적용될 것으로 추정(미확인). 클라이언트는 429/5xx를 지수 백오프로 방어적 처리.
- **에러 응답 형식**: 각 메서드 페이지에 에러 스키마 미기재. Google 표준(HTTP 4xx/5xx + JSON `error`)으로 추정(미확인). `status.message` 원문을 사용자에게 전달.
- **가치 한계**: 본 API는 자산(리포트·데이터소스) **메타데이터 검색 + 공유 권한 관리** 전용. 대시보드 데이터 값·리포트 내용·차트·필터/섹션/차원 조회 및 리포트 생성/편집은 **불가**. 데이터 조회형 BI 연동이 목표면 엔터프라이즈 Looker(공식 MCP 보유)를 검토.
- **파괴적 작업**: `revoke_all_asset_permissions`(전 멤버 회수)는 기본 비활성. `LOOKER_ENABLE_DESTRUCTIVE=true` + 실행 시 확인 토큰(`confirm: "REVOKE"`)으로만 노출.
- 개인 Gmail 사용자(한국 소상공인·개인 다수 포함)는 인증 자체 불가 — 도입 전 조직 계정·도메인 위임 가능 여부를 반드시 확인.
