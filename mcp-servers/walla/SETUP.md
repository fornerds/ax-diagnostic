# 키 발급 가이드 — 왈라 (walla)

왈라(Walla) Open API 키를 발급해 MCP 서버(`mcp-walla`)에 연결하기 위한 가이드. 아래 내용은 모두 왈라 공식 문서에 출처가 있는 사실만 정리했다(추측 없음). 불명확한 항목은 "미확인"으로 표기했다.

## 연동 유형 / 인증 방식

- **연동 유형**: REST API + API Key (OAuth2/토큰 교환/SDK 없음).
- **인증**: 발급한 API 키를 모든 요청 헤더에 부착.
  - 헤더: `X-WALLA-API-KEY: <발급한 키>`
  - Basic/Bearer/clientId 방식 아님.
- **인증 실패 시 응답**: `403` (401이 아님).

## 발급 절차 (스텝)

1. `https://app.walla.my` 에 로그인한다.
2. `https://app.walla.my/open-api/doc` 로 이동한다. (콘솔/Swagger/키 발급용 사람 페이지)
3. 페이지 상단 API 키 섹션에서 키를 생성하고 복사한다.

> 주의: 위 `app.walla.my`는 키 발급·콘솔·Swagger용(사람용)이다. 실제 **API 호출 베이스 URL은 `https://api.walla.my/open-api/v1`** 이다(아래 "주의" 참조).

## 전제조건

- 로그인 가능한 왈라 계정.
- **플랜/사업자 심사/발신번호 사전등록 등 별도 전제조건**: 공식 문서에 명시 없음 → "미확인"(아래 "미확인 / 주의" 참조). 발신번호·사업자 심사 같은 절차는 문서상 요구되지 않음(해당 없음).

## 스코프 / 권한

- 별도 OAuth 스코프 체계 없음 — **단일 키**.
- 키가 닿는 워크스페이스/폼 범위에 따라 접근이 제한될 수 있으며, 권한 밖 리소스 접근 시 `403`이 반환된다(런타임 가드).
- 노출 API는 전부 읽기(GET 조회) + POST export. 폼/응답 생성·수정·삭제 같은 쓰기 권한은 공개 API에 없음.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `WALLA_API_KEY` | `app.walla.my/open-api/doc` 상단 API 키 섹션에서 생성한 키. `X-WALLA-API-KEY` 헤더 값으로 사용 | 예 |
| `WALLA_API_BASE_URL` | API 호출 베이스. 미설정 시 기본값 `https://api.walla.my/open-api/v1` 사용. ONPREM 등 다른 호스트 오버라이드용 | 아니오 |

> `WALLA_API_KEY`는 하드코딩 금지. `.env.example`을 `.env`로 복사해 채우고 `.env`는 커밋하지 않는다(`.gitignore` 포함).

## 출처 (공식 URL)

- 인증(헤더·키 발급 경로·베이스 URL·레이트리밋): https://docs.walla.my/ko/docs/developer-docs/authentication
- 응답 API: https://docs.walla.my/ko/docs/help-center/integrate-forms/response-api
- Parquet 내보내기: https://docs.walla.my/ko/docs/developer-docs/responses/parquet
- 키 발급/콘솔 페이지(사람용): https://app.walla.my/open-api/doc

## 미확인 / 주의

- **[주의] 호출 베이스 URL 혼동**: 키 발급은 `app.walla.my`에서 하지만, 실제 API 호출 베이스는 **`https://api.walla.my/open-api/v1`** 이다. `app.walla.my`를 호출 베이스로 쓰면 안 된다(공식 curl 예시는 모두 `api.walla.my` 사용 — 출처 인증/Parquet 문서).
- **[미확인] 플랜별 Open API 허용 여부**: 무료/유료 플랜에 따라 Open API 사용이 허용되는지 공식 문서에 명시 없음. 사용자 키로 실제 호출 시 확정되며, 차단 시 `403`으로 나타난다.
- **[미확인] 레이트리밋 응답 헤더**: `Retry-After`/`X-RateLimit-*` 존재 여부 공식 문서 미명시 → 헤더에 의존하지 말고 고정/지수 백오프로 처리.
- **[정보] 레이트리밋**: API 키당 시간당 6,000건(1시간 롤링 윈도우, export 포함 공유). 초과 시 `429 Too Many Requests`. (출처: 인증 문서)
