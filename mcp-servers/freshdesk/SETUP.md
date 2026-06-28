# 키 발급 가이드 — Freshdesk (freshdesk)

## 연동 유형 / 인증 방식
- **연동 유형**: Freshdesk REST API v2 (HTTPS 전용, JSON). 베이스 URL `https://<domain>.freshdesk.com/api/v2`.
- **인증 방식**: HTTP Basic Authentication.
  - 사용자명 = 개인 **API key**, 비밀번호 = 더미 문자 `X`.
  - 서버가 `Authorization: Basic base64("<API_KEY>:X")` 헤더를 자동 구성한다.
  - 토큰 교환·만료 단계 없음(장수명 API key 직접 사용).
- **OAuth2 아님**: REST v2는 OAuth를 쓰지 않는다(OAuth는 마켓플레이스 앱 한정, 본 연동과 무관).

## 발급 절차 (스텝)
공식 도움말 기준 (support.freshdesk.com — "How to find your API key" / "Where can I find my API key?"):

- **발급 페이지 직링크**: 없음(직링크 없음). API key는 워크스페이스/계정 도메인에 종속되어 공용 발급 URL이 존재하지 않는다. 본인 계정 포털(`https://<domain>.freshdesk.com`)에 로그인한 상태에서 아래 메뉴 경로로 접근한다.
- **메뉴 > 버튼 경로**: 로그인 → 우측 상단 **프로필 사진** 클릭 → **Profile Settings** → 우측 패널의 **[View API key]** 버튼 클릭 → **CAPTCHA 인증** 완료 → 표시된 API key 복사.

1. 본인 Freshdesk 계정에 로그인한다(`https://<domain>.freshdesk.com`).
2. 우측 상단의 **프로필 사진**을 클릭하고 **Profile Settings** 를 선택한다.
3. **Profile Settings** 페이지의 **우측 패널**에서 **[View API key]** 버튼을 클릭한다.
4. **CAPTCHA 인증**을 완료하면 API key가 표시된다 — 복사한다.

> 주의: API key는 상담원(agent)이 **verified 상태**일 때만 표시된다. 키 표시 후에는 안전하게 보관할 것(재설정(Reset) 시 해당 키를 쓰는 모든 연동이 끊긴다).
> 참고: 일부 구(舊) UI 계정은 [View API key] 버튼/CAPTCHA 없이 우측 패널에 API Key가 바로 노출될 수 있다. 키가 보이지 않으면 브라우저 캐시/쿠키 삭제 후 다른 브라우저로 재로그인.
> 이 키가 Basic Auth의 사용자명이 된다. 비밀번호 자리에는 더미 `X`를 쓴다.

## 전제조건
- **플랜**: REST API v2 사용에 별도 유료 플랜·심사·앱 등록 불필요(유효한 Freshdesk 계정의 상담원/관리자 프로필만 있으면 됨).
  - 단, 레이트리밋은 플랜별 차등(Free 0 / Growth 200 / Pro 400 / Enterprise 700 / Trial 50 calls/min — 분당 체계 단계적 롤아웃 중). Free 플랜은 분당 한도 0으로 표기됨에 유의.
- **사업자 심사 / 실명확인 / 발신번호 사전등록**: 없음(글로벌 SaaS, 한국 특화 규제 무관).
- **공식 원격 MCP 커넥터(부록)** 를 직접 연결하려는 경우에 한해: Enterprise 플랜 + EAP 승인 필요(아래 부록 참조). 본 로컬 빌드 경로에는 해당 없음.

## 스코프 / 권한
- 별도 스코프/권한 토글 없음 — **API key는 발급한 사용자의 역할 권한을 그대로 상속**한다.
- 운영 권장: 과권한 방지를 위해 작업에 필요한 최소 권한의 **상담원(agent) 키** 사용. 관리자 키 남용 지양.
- 키는 고권한 단일 시크릿이므로 .env/시크릿 스토어에만 보관하고 코드·로그·응답에 노출 금지.

## 환경변수 매핑
| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `FRESHDESK_DOMAIN` | 계정 서브도메인 — `<domain>.freshdesk.com`의 `<domain>` 부분(예: 포털이 `acme.freshdesk.com`이면 `acme`) | 예 |
| `FRESHDESK_API_KEY` | Profile settings 페이지 "Your API Key"에서 복사한 개인 API key | 예 |

## 출처 (공식 URL)
- API 레퍼런스 / 인증 / API key 위치: https://developers.freshdesk.com/api/
- API key 발급 위치(메뉴/버튼 경로): https://support.freshdesk.com/support/solutions/articles/215517-how-to-find-your-api-key
- API key 발급 위치(보조 확인): https://support.freshdesk.com/support/solutions/articles/225435-where-can-i-find-my-api-key-
- 레이트리밋(플랜별 분당 한도): https://support.freshdesk.com/support/solutions/articles/225439-what-are-the-rate-limits-for-the-api-calls-to-freshdesk-
- 공식 원격 MCP 커넥터(EAP, Enterprise 한정): https://support.freshdesk.com/support/solutions/articles/50000012670-model-context-protocol-mcp-integration-in-freshdesk-eap-

## 미확인 / 주의
- **멀티리전 호스트명(미확인)**: 리전별 별도 호스트 부재의 1차 근거 미확보. 항상 계정 `<domain>.freshdesk.com`을 사용하는 것으로 가정한다. 401/연결 실패 시 `FRESHDESK_DOMAIN`을 먼저 점검.
- **레이트리밋 최신성**: 근거 문서 Last updated 2024-03-14, "분당 체계 단계적 롤아웃" 명시. 일부 구계정은 여전히 시간당 한도(예: 5,000/hr)일 수 있음 → 고정 수치 가정 금지, 429 수신 시 응답 `Retry-After`(초) 동적 대응.
- **전용 인증 article 로그인 벽**: API key 기반 인증 전용 article(`.../50000003346-...`)은 로그인 리다이렉트로 직접 확인 불가했으나, 인증 메커니즘은 공개 REST 레퍼런스(developers.freshdesk.com/api/)로, **키 발급 메뉴/버튼 경로는 공개 도움말 article 2건**(215517 / 225435)으로 교차 검증 완료.
- **주의(혼동 금지)**: `https://mcp.freshworks.dev/mcp`(Bearer)는 별개 서버(마켓플레이스 앱 배포용, 티켓 데이터 접근 아님). 본 가이드의 키와 무관.

---

## 부록 — 공식 원격 MCP 커넥터 키(Enterprise + EAP 한정)
> 빌드 경로와 **별개**. 대상이 Enterprise 플랜 + EAP 승인 고객이면 서버를 빌드하지 않고 공식 원격 커넥터를 직접 연결할 수 있다.
- **인증**: API key only(OAuth 미지원). 위 동일한 Profile settings의 API key를 `Authorization` 헤더에 사용.
- **원격 URL**: `https://<your-freshdesk-domain>/mcp` (HTTP streamable transport).
- **가용성**: Beta + EAP, Enterprise 한정. 신청: technical account manager 또는 support@freshdesk.com.
- 출처: https://support.freshdesk.com/support/solutions/articles/50000012670-model-context-protocol-mcp-integration-in-freshdesk-eap-
