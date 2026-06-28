# 키 발급 가이드 — 스티비 (stibee)

> 모든 내용은 `_workspace/stibee/02_validation.md`(공식 OpenAPI v2.0.0 + help.stibee.com 대조 검증본)와 `README.md`에서 가져왔습니다. 추측은 배제했고, 불명확한 항목은 "미확인"으로 표기했습니다.

## 연동 유형 / 인증 방식

- **연동 유형**: REST API v2 (베이스 URL `https://api.stibee.com/v2`, OpenAPI `version: 2.0.0`).
- **인증 방식**: API Key. OAuth 없음 / 토큰 교환·갱신 단계 없음.
- 모든 요청에 HTTP 헤더 `AccessToken: {발급한 키}` 를 넣습니다. 본문이 있는 요청은 `Content-Type: application/json`(UTF-8).
- 출처: openapi.yml `components.securitySchemes.AccessToken` (type:apiKey, in:header, name:AccessToken), `info.description` 인증 섹션.

## 발급 절차 (스텝)

1. 스티비에 로그인한 뒤 **워크스페이스 설정**으로 이동합니다.
2. **API 키** 메뉴를 선택합니다.
3. **새로 만들기**를 눌러 키를 생성합니다.
4. 생성된 키를 **복사**해 안전한 곳(환경변수)에 저장합니다.

- 워크스페이스당 API 키는 **최대 10개**까지 생성 가능합니다.
- 발급은 콘솔에서 사용자가 직접 하는 **자가 발급**이며, 사업자/실명 인증 절차는 발급 요건으로 명시돼 있지 않습니다.
- 출처: https://help.stibee.com/api-webhook/api ("워크스페이스 설정 > API 키 > 새로 만들기 > 복사", "최대 10개까지 생성").

## 전제조건

- **스탠다드 이상 유료 요금제 필요.** 무료(스타터) 요금제에서는 API 키 발급/사용이 불가하며, 모든 도구가 인증 단계에서 실패합니다. 출처: https://help.stibee.com/api-webhook/overview ("스탠다드, 프로, 엔터프라이즈 요금제에 해당").
- **2025-01-21 이후 발급한 키만 유효.** 그 이전 키는 `Errors.Authorization.NoToken` 으로 실패하므로 새로 발급해야 합니다. 출처: openapi.yml `info.description` 인증 섹션.
- **발신번호 사전등록**: 해당 없음(스티비는 이메일 서비스). 단, **이메일 발송**(send/reserve 계열)은 발신자 이메일이 **사전 인증**돼 있어야 하고, gmail.com·naver.com 등 공용 도메인은 발신자로 사용할 수 없습니다(`NotPermittedSenderDomain`). 출처: openapi.yml `verifyListSender`, `AddSenderErrorResponse`.
- 사업자 심사: 키 발급 자체에는 불필요(자가 발급). 단, 유료 결제 단계에서 사업자정보 입력이 필요한지는 **미확인**(아래 참조).

## 스코프 / 권한

스티비 API 키에는 개별 스코프 선택 개념이 없고, **워크스페이스 요금제별로 사용 가능한 기능이 게이팅**됩니다. 요금제가 낮으면 해당 도구가 `Errors.Authorization.PermissionDenied`(httpStatusCode 400)로 실패합니다.

| 기능 영역 | 필요 요금제 |
|---|---|
| 구독자 관리, 그룹 할당/해제 | 스탠다드 이상 |
| 그룹 CRUD, 이메일 | 프로 이상 |
| 주소록·발신자 주소 | 엔터프라이즈 |

출처: openapi.yml `info.description` 요금제별 권한 표 + 각 operation의 `x-badges`/`tags`. (그룹 할당/해제는 스탠다드+, 그룹 CRUD는 프로+로 구분됨 — 02_validation 수정 항목 반영.)

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `STIBEE_API_KEY` | 워크스페이스 설정 > API 키 > 새로 만들기 로 발급한 키(2025-01-21 이후 발급분). `AccessToken` 헤더 값으로 사용 | 예 |
| `STIBEE_BASE_URL` | 베이스 URL 오버라이드. 미설정 시 기본값 `https://api.stibee.com/v2` 사용 | 아니오 |

## 출처 (공식 URL)

- API 키 발급 절차·개수 제한: https://help.stibee.com/api-webhook/api
- API 사용 가능 요금제 개요: https://help.stibee.com/api-webhook/overview
- 공식 OpenAPI 스펙(v2.0.0, 진실원본): https://raw.githubusercontent.com/stibee/api-docs/main/openapi.yml

## 미확인 / 주의

- **유료 결제 단계의 사업자정보 요구 여부 — 미확인.** 키 발급 자체는 자가 발급이나, 유료 결제 시 사업자정보 입력이 필요한지는 공식 결제/요금제 문서에서 별도 확인 권장(연동 자체에는 영향 없음). 출처 미발견. (02_validation 리스크 8)
- **레이트리밋 초과 시 HTTP 상태/헤더 — 미확인.** 분당/초당 한도(대량추가 10/분, 목록조회 100/분, 그 외 1000/분)는 명시돼 있으나 429·Retry-After 제공 여부는 스펙에 없음 → 클라이언트가 보수적으로 준수. (02_validation 리스크 6)
- **자동(트리거) 이메일 v1 폐기 일정 — 미확인.** 자동이메일은 레거시 호스트 `https://stibee.com/api/v1.0/auto/...` 로만 안내되며 공식 폐기 고지가 없음. 이 커넥터는 v2만 사용하므로 영향 없음.
- 키는 발급 시 1회 복사해 보관하세요. 분실 시 재발급이 필요할 수 있습니다(공식 재노출 절차 명시 없음 — 콘솔에서 확인).
