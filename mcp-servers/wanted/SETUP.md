# 키 발급 가이드 — 원티드(Wanted) (wanted)

> 출처 검증본: `_workspace/wanted/02_validation.md` (검증일 2026-06-26) 및 `README.md`.
> 추측 없이 출처가 있는 사실만 기재한다. 출처가 불명확한 항목은 "미확인"으로 표기.

## 연동 유형 / 인증 방식

- **연동 유형**: 공식 MCP/Claude 커넥터 없음 → 원티드 공개 REST(OpenAPI)를 로컬 stdio MCP 서버로 빌드(route = build).
- **인증 방식**: API Key(헤더 방식). OAuth 아님 — 토큰 교환/리프레시 단계 없음.
- **필수 헤더 2종**(모든 호출에 부착):
  - `wanted-client-id: <client_id>`
  - `wanted-client-secret: <client_secret>`
- **공통 헤더**: `accept: application/json`
- **선택 헤더(가드)**: `Authorization: <값>` — 스펙상 `PermissionsDependency`로 존재하나 필수성 미증명. 환경변수 `WANTED_AUTHORIZATION`가 설정된 경우에만 부착(미확인).

## 발급 절차 (스텝)

원티드 OpenAPI는 즉시 self-serve가 아니다. 포털에서 신청 후 메일로 발급된다.

1. **포털 접속**: https://openapi.wanted.jobs/ 접속.
2. **이용 목적 확인 → 신청**: 포털 안내에 따라 이용 목적을 확인하고 인증정보 발급을 신청한다. (포털 명시 흐름: 이용목적 → 신청 → 발급 → 사용, 4단계)
3. **메일 수신(발급)**: 신청 접수일 기준 **3 영업일 이내** 메일로 `client_id` / `client_secret` 수신. 이때 **테스트 서비스용**과 **실 서비스용** 인증 정보가 함께 발급된다.
4. **환경변수 주입**: 발급된 값을 아래 환경변수에 주입하고 서버를 구동한다.
   - 테스트키: 스테이징 베이스 `https://stg-openapi.wanted.jobs` 사용(`WANTED_API_BASE`로 전환).
   - 운영키: 운영 베이스 `https://openapi.wanted.jobs`(기본값).

## 전제조건

- **플랜/유료 계약(공개 읽기 도구 한정)**: **없음**. 본 커넥터의 11개 공개 읽기 도구는 `client_id`/`client_secret`만으로 호출 가능.
- **사업자 심사**: 미확인 — 포털은 "이용 목적 확인" 후 발급이라고만 명시. 명시적 사업자 심사 문구 없음. 개인 개발자 가능 여부도 명시 없음(신청으로 실증 필요).
- **발신번호 사전등록 등**: 해당 없음(메시징 툴 아님).
- **발급 리드타임**: 접수 후 3 영업일 — 즉시 발급 아님(일정에 반영).
- **참고(이번 커넥터 범위 밖 전제조건)**: ATS(`/ats/*`, `/recruit-company/*`)는 `x-wanted-dashboard-service-key` 필수(원티드 채용 대시보드=기업/유료 상품 보유 전제), AI 예측(`/ai/*`)은 별도 유료 계약(team-ml@wantedlab.com). 본 빌드에서는 제외.

## 스코프 / 권한

- **명시적 scope 체계 없음**(OAuth scope 부재). 권한은 발급 범위로 결정되며 공식 매핑표 없음 — **미확인**.
- 본 커넥터 범위: **공개 읽기 엔드포인트만**(채용 포지션·기업·태그 조회). 위 키만으로 호출 가능.
- 권한 밖(미부여 시 접근 불가): ATS/지원자 관리, `/stat/*`, AI 예측 — 별도 전제조건 필요(위 "전제조건" 참조).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `WANTED_CLIENT_ID` | 발급 메일의 `client_id` → 헤더 `wanted-client-id` 값 | 필수 |
| `WANTED_CLIENT_SECRET` | 발급 메일의 `client_secret` → 헤더 `wanted-client-secret` 값 | 필수 |
| `WANTED_API_BASE` | 베이스 호스트. 기본 `https://openapi.wanted.jobs`(운영키), 테스트키 시 `https://stg-openapi.wanted.jobs` | 선택 |
| `WANTED_AUTHORIZATION` | (가드) `Authorization` 헤더 값. 설정 시에만 부착(필수성 미확인) | 선택 |

> 시크릿 실값을 `.mcp.json`에 직접 쓰지 말고 `${VAR}` 확장 또는 셸 환경/`.env`(커밋 금지)에 둔다.

## 출처 (공식 URL)

- 발급 포털(이용목적→신청→발급→사용 4단계, 3영업일, 테스트·운영 동시 발급): https://openapi.wanted.jobs/
- 가이드(베이스 URL, client_id/secret 헤더 사용 curl 예시, 스테이징 호스트): https://openapi.wanted.jobs/guides/
- V1 OpenAPI 스펙(securitySchemes `ClientIdHeader`/`ClientSecretHeader`/`PermissionsDependency`): https://openapi.wanted.jobs/v1/openapi.json
- V2 OpenAPI 스펙: https://openapi.wanted.jobs/v2/openapi.json
- 공식 MCP/Claude 커넥터 부재 확인: https://claude.com/docs/connectors/directory

## 미확인 / 주의

- **`Authorization` 헤더 필수성 미증명**: 스펙상 `PermissionsDependency`로 존재하나, 가이드 공식 curl 예시는 `wanted-client-id`+`wanted-client-secret` 2개만 사용. 선택일 가능성 크나 런타임 미검증 → `WANTED_AUTHORIZATION` 설정 시에만 부착.
- **사업자 심사 / 개인 개발자 발급 자격**: 포털에 명시 문구 없음 — 신청으로 실증 필요.
- **스코프/권한 매핑표 부재**: 이용 목적별로 열리는 엔드포인트 공식 매핑 없음.
- **레이트리밋/쿼터 수치 미상**: 스펙·가이드·포털 어디에도 없음(429/RateLimit 헤더 정의 자체 없음). 발급 메일/약관에서 분당·일별 쿼터 확인 권장.
- **키 갱신/만료 절차 미기재**: 문서에 없음 — 미확인.
- **개인정보 주의**: `get_company_insight`의 `biz_number`는 사업자등록번호(기업 통계용)이며 개인 식별정보가 아니나, 입력이 민감 정보임에 유의.
