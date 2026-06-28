# 키 발급 가이드 — CLOVA Studio / HyperCLOVA X (clovax)

> 대상은 **CLOVA Studio REST API**(네이버 클라우드 플랫폼, NCP)입니다. 소비자용 CLOVA X(clova-x.naver.com)의 비공식 래퍼는 약관 위반·차단 리스크가 있어 사용하지 않습니다. 공식 API만 사용합니다.

## 연동 유형 / 인증 방식

- **연동 유형**: API Key (Bearer). OAuth·토큰 교환 없음 — 콘솔에서 발급한 키를 HTTP 헤더에 직접 사용합니다.
- **필수 헤더**:
  - `Authorization: Bearer ${CLOVASTUDIO_API_KEY}` (키는 `nv-` 접두사)
  - `Content-Type: application/json`
- **선택 헤더**: `X-NCP-CLOVASTUDIO-REQUEST-ID: <uuid>` (요청 추적용 — 필수 아님. 서버가 미설정 시 호출마다 UUID 자동 생성).
- **키 종류**: 테스트 API 키(개발용) / 서비스 API 키(운영용). 둘 다 동일한 환경변수 하나로 수용합니다.
- 출처: https://api.ncloud-docs.com/docs/en/ai-naver-clovastudio-summary

## 발급 절차 (스텝)

1. **NCP 가입 + 결제수단 등록** — CLOVA Studio는 NCP 유료 서비스라 결제수단 등록이 선행되어야 합니다(과금 발생).
2. **NCP 콘솔 로그인** → 콘솔(`https://console.ncloud.com`) 우측 상단 **`Region & Platform`** 버튼에서 사용 리전·플랫폼 선택 후 **`[적용]`** 클릭 → 좌측 상단 **`Menu`** → **`Services > AI Services > CLOVA Studio`** 메뉴를 차례대로 클릭해 진입. 이후 **`My Product > [CLOVA Studio 바로가기]`** 버튼으로 CLOVA Studio 앱 화면(`https://clovastudio.ncloud.com/`)에 접속.
   - NCP 콘솔 진입점(로그인 필요): https://console.ncloud.com
   - CLOVA Studio 앱 도메인(로그인 필요): https://clovastudio.ncloud.com/
   - **직링크 없음**: NCP 콘솔은 로그인 게이트 뒤 SPA라 "API 키 화면"까지 곧장 가는 안정적 딥링크 URL이 공식 문서에 없음 → 콘솔 진입 후 위 메뉴 경로로 이동.
3. (앱 미생성 시) **테스트 앱**(개발용) 또는 **서비스 앱**(운영용)을 먼저 생성.
4. **API 키 발급** — 화면 **좌측 `API 키` 메뉴** 클릭 → API 키 화면에서 발급할 탭 선택 후 발급 버튼 클릭:
   - 개발용: **`[테스트]` 탭** → **`[테스트 API 키 발급]`** 버튼
   - 운영용: **`[서비스]` 탭** → **`[서비스 API 키 발급]`** 버튼
   - 발급된 키는 `nv-` 접두사. NCP 메인 계정 기준 테스트/서비스 각 **최대 10개**.
5. 발급한 키를 환경변수 `CLOVASTUDIO_API_KEY` 에 설정.

> **테스트 키 vs 서비스 키**: 개발은 테스트 키(낮은 QPM/TPM 한도)로 충분합니다. 운영 배포 시 **서비스 앱 등록 후 서비스 키로 교체**하면 한도가 올라갑니다(예: HCX-007 테스트 60QPM·60kTPM → 서비스 180QPM·300kTPM, 2025-07-17 기준).

## 전제조건

- **NCP 유료 계정 + 결제수단 등록**: 필수(코드로 자동화 불가, 사용자 책임).
- **사업자 심사**: 문서에 별도 사업자/제휴 "심사" 게이트는 **미기재**. NCP 콘솔에서 셀프 발급(테스트/서비스 각 10개) 가능. 단 "심사 불필요"를 단정하는 명시 문구도 없음 → 키 발급은 사용자 책임으로 진행하세요(미확인 참고).
- **발신번호 사전등록 등 기타 사전등록**: 없음(해당 없음 — 메시징 커넥터가 아님).
- **플랜 요건**: 별도 플랜 가입 게이트 없음. 사용량 기반 과금.

## 스코프 / 권한

- OAuth 스코프 개념 **없음**. 권한은 **키 단위**로 결정됩니다.
- 테스트 키: 테스트 앱 범위 호출. 서비스 키: 등록된 서비스 앱 호출용(높은 한도).

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `CLOVASTUDIO_API_KEY` | NCP 콘솔 > CLOVA Studio > 테스트 앱 또는 서비스 앱에서 발급한 API 키(`nv-` 접두사) | 예 |
| `CLOVASTUDIO_BASE_URL` | 베이스 호스트 오버라이드. 미설정 시 기본 `https://clovastudio.stream.ntruss.com` (정부망 등 예외 대응용) | 아니오 |
| `CLOVASTUDIO_REQUEST_ID` | 고정 요청 ID가 필요한 경우. 미설정 시 호출마다 UUID 자동 생성 | 아니오 |

> 시크릿 하드코딩 금지. `.env.example` 에는 `nv-...` 자리표시자만 둡니다. 실제 키는 절대 커밋하지 마세요(`.gitignore` 에 `.env` 포함).

## 출처 (공식 URL)

- 개요 / 인증 / 호스트 / 키 발급(테스트·서비스 각 10개): https://api.ncloud-docs.com/docs/en/ai-naver-clovastudio-summary
- OpenAI 호환(테스트/서비스 키 허용): https://api.ncloud-docs.com/docs/en/clovastudio-openaicompatibility
- 에러코드(40104 Invalid key 등): https://api.ncloud-docs.com/docs/en/clovastudio-troubleshoot
- 레이트리밋(테스트/서비스 한도, 2025-07-17 기준): https://guide.ncloud-docs.com/docs/en/clovastudio-ratelimiting
- 콘솔 진입(메뉴 경로 `Menu > Services > AI Services > CLOVA Studio`): https://guide.ncloud-docs.com/docs/clovastudio-start
- API 키 발급 절차(좌측 `API 키` 메뉴 → `[테스트]`/`[서비스]` 탭 → 발급 버튼, 각 최대 10개): https://api.ncloud-docs.com/docs/ai-naver-clovastudio-summary

## 미확인 / 주의

- **API 키 화면 직링크**: NCP 콘솔 진입점(`https://console.ncloud.com`)과 발급 메뉴 경로(`Menu > Services > AI Services > CLOVA Studio` → 좌측 `API 키` 메뉴 → `[테스트]`/`[서비스]` 탭 → 발급 버튼)는 공식 문서로 확인됨. 단 로그인 게이트 뒤 SPA라 "API 키 화면"으로 곧장 가는 안정적 딥링크 URL은 공식 문서에 없음 → **직링크 없음**(메뉴 경로로 진입). 출처: clovastudio-start, ai-naver-clovastudio-summary.
- **사업자/제휴 심사 필요 여부**: 별도 심사 게이트는 문서에 미기재(셀프 발급 가능). 다만 "심사 불필요"를 단정하는 명시 문구도 없음 → **미확인(완화)**. 키 발급은 사용자 책임.
- **키 부재/무효 시 동작**: 키가 없거나 유효하지 않으면(`status.code 40104` Invalid key) 서버가 명확한 에러를 반환합니다.
- **레거시 호스트 금지**: 구 `https://clovastudio.apigw.ntruss.com` + 이중 NCP 헤더(`X-NCP-CLOVASTUDIO-API-KEY` / `X-NCP-APIGW-API-KEY`) 방식은 지원 중단 예정이라 사용 금지. 단일 `Authorization: Bearer nv-...` 만 사용합니다.
