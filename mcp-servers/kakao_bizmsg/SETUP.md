# 키 발급 가이드 — 카카오 비즈메시지 (kakao_bizmsg)

> 기준: 카카오아이(디케이테크인) **Kakao i Connect Message — BizMessage API v2.0**
> 출처 검증본: `_workspace/kakao_bizmsg/02_validation.md` (검증일 2026-06-26)
> 핵심 전제: **셀프서비스 콘솔이 없습니다.** Client ID/Secret 은 **서비스 계약 시 발급**되며, 발송에 필요한 채널·템플릿·발신번호는 코드와 무관하게 사용자 측에서 사전에 준비·심사받아야 합니다. **"구현 완료 ≠ 즉시 발송 가능".**

---

## 연동 유형 / 인증 방식

- **연동 유형**: OAuth 2.0 + REST (로컬 MCP 서버가 공개 REST API 를 감쌈; 공식 원격 MCP 커넥터 없음).
- **인증 방식**: OAuth 2.0 **Client Credentials Grant** → Bearer access token.
  - 토큰 발급: `POST {BASE_URL}/v2/oauth/token`
    - 헤더: `Authorization: Basic {CLIENT_ID} {CLIENT_SECRET}` (**공백 구분 평문** — 표준 `base64(id:secret)` Basic 이 아님), `Content-Type: application/x-www-form-urlencoded`
    - 바디: `grant_type=client_credentials`
    - 응답: `access_token`, `token_type=bearer`, `expires_in`(초)
  - 이후 호출: `Authorization: Bearer {access_token}`
  - **refresh token 없음 / scope 없음**(확정). 만료 시 동일 엔드포인트로 재발급. (서버가 자동 발급·캐시)

---

## 발급 절차 (스텝)

발급은 셀프서비스가 아니라 **계약 + 심사** 절차입니다. 순서대로 모두 완료해야 발송이 가능합니다.

1. **서비스 계약 / 영업 문의** — 셀프 가입 콘솔이 없으므로 카카오아이(디케이테크인)에 직접 문의해 BizMessage 서비스 계약을 체결합니다.
   - **발급 페이지 직링크 없음** (셀프서비스 콘솔/가입 포털이 존재하지 않음 — 계약 경유로만 발급).
   - 디케이테크인 영업/문의: **contact.dkt@kakaocorp.com**
   - API 문서 진입점: https://docs.kakaoi.ai/kakao_i_connect_message/bizmessage/api/
   - 계약 주체가 카카오엔터프라이즈(레거시 v1.0) 또는 리셀러(SOLAPI·팝빌 등)일 수도 있으며, 그 경우 발급 위치·스펙이 달라집니다(아래 "미확인/주의" 참조).
2. **Client ID / Client Secret 수령** — 계약 체결 시 발급됩니다(셀프 발급 화면 없음 — 계약 담당자가 전달). → `KAKAO_BIZMSG_CLIENT_ID`, `KAKAO_BIZMSG_CLIENT_SECRET`.
3. **카카오톡 비즈니스 채널 개설 + 사업자/대표자 인증 심사** — 카카오톡채널 관리자센터에서 진행합니다. 승인 시 메시지 플랫폼(계약 콘솔)에서 **`sender_key`(발신 프로필 키)** 발급. → `KAKAO_BIZMSG_SENDER_KEY`.
   - 발급 콘솔 직링크: **https://center-pf.kakao.com/** (로그인 후 작업)
   - 메뉴 경로: 로그인 → (채널 미생성 시) **[새 채널 만들기]** 로 채널 개설 → 좌측 **채널** → **비즈니스 심사** → **[심사 신청하기]** 버튼 (※ 비즈니스 심사 신청은 **채널 마스터** 권한만 가능).
   - 필요 서류: 개인사업자 대표=본인 명의 카카오톡(전자증명서), 법인사업자 대표=사업자등록증 + 대표자 휴대폰 본인인증. 심사 기간: 영업일 약 3~5일.
   - 비즈니스 심사 승인 후, 메시지 발송용 발신프로필 키(`sender_key`)는 계약한 메시지 플랫폼(디케이테크인 콘솔) 측에서 채널을 연동해 발급받습니다(계약 콘솔 화면 직링크는 계약 계정에서 확인).
4. **알림톡 템플릿 사전심사** — 발송할 메시지 템플릿을 등록·심사받습니다. 승인된 **`template_code`** 만 발송 가능(임의 텍스트 발송 불가, 알림톡=정보성 한정). 템플릿 등록·검수 요청은 계약한 메시지 플랫폼(디케이테크인 콘솔)의 템플릿 관리 화면에서 진행하며, 검수 요청을 해야만 사용 가능하고 승인된 템플릿은 수정 불가(수정 시 신규 등록). → 각 발송 호출의 `template_code`.
5. **발신번호(`sender_no`) 사전등록** — 전기통신사업법 제84조의2 의무. AT/FT/XMS 모두 `sender_no` 필수이며, 등록되지 않은 번호로는 발송 불가. 발신번호 등록은 계약한 메시지 플랫폼(디케이테크인 콘솔)의 발신번호 관리 화면에서 진행하고, 통신사 발신번호 증명서 첨부 등록을 선행합니다. → `KAKAO_BIZMSG_SENDER_NO`.
6. **베이스 URL 확정** — 계약 주체에 맞는 host 를 `KAKAO_BIZMSG_BASE_URL` 로 설정(기본값 디케이테크인 v2.0).

> **발급 위치 요약**: (a) Client ID/Secret = 계약 시 발급(셀프 콘솔/직링크 없음, `contact.dkt@kakaocorp.com` 경유). (b) 채널 개설 + 비즈니스 심사 = `https://center-pf.kakao.com/` → 채널 > 비즈니스 심사 > [심사 신청하기]. (c) `sender_key`·`template_code`·`sender_no` = 계약한 메시지 플랫폼(디케이테크인) 콘솔의 발신프로필/템플릿/발신번호 관리 화면(계약 콘솔 직링크는 워크스페이스별이라 계약 계정에서 확인).

> 미발급/미승인 상태에서는 발송이 **400(인증오류)** 또는 **410(입력값오류)** 으로 실패하며, 도구가 그 코드와 함께 안내 문구를 반환합니다.

---

## 전제조건

발송 가능 상태가 되기 위한 사용자 측 선행 조건(코드와 무관하게 필요):

- **서비스 계약 필수** — 셀프서비스 아님. 계약 시 Client ID/Secret 발급.
- **사업자/대표자 인증 심사 필수** — 카카오톡 비즈니스 채널 개설 + 발신 프로필 심사(영업일 약 1~3일).
- **알림톡 템플릿 사전심사 필수** — 승인된 `template_code` 만 발송 가능.
- **발신번호 사전등록 필수** — 전기통신사업법 제84조의2 의무. AT/FT/XMS 공통으로 `sender_no` 필수.
- **플랜 요구사항**: 별도 명시 없음(계약 단위 권한). 요금제별 TPS 등은 미확인(아래 참조).

---

## 스코프 / 권한

- **OAuth scope: 없음** — client_credentials grant 라 scope 개념이 없습니다(확정). 권한은 **계약 단위**로 부여됩니다.
- 발송 가능 메시지 종류(알림톡/친구톡/SMS 등)는 계약·채널 심사 범위에 따릅니다.

---

## 환경변수 매핑

| 변수 | 값을 어디서 받나 | 필수 |
|------|----------------|------|
| `KAKAO_BIZMSG_CLIENT_ID` | 서비스 계약 시 발급 (디케이테크인/카카오엔터프라이즈) | 예 |
| `KAKAO_BIZMSG_CLIENT_SECRET` | 서비스 계약 시 발급 | 예 |
| `KAKAO_BIZMSG_BASE_URL` | 계약 주체별 host. 기본값 `https://web1.dktechinmsg.com`(디케이테크인 v2.0) | 예(기본값 제공) |
| `KAKAO_BIZMSG_SENDER_KEY` | 카카오 비즈니스 관리자센터 채널/발신 프로필 심사 승인 후 발급(발신 프로필 키) | 권장(도구별 override 가능) |
| `KAKAO_BIZMSG_SENDER_NO` | 사전등록한 발신번호(전기통신사업법 제84조의2 등록 완료 번호) | 권장(도구별 override 가능) |

### 계약 주체별 베이스 URL

- 디케이테크인 v2.0(표준, 기본값): `https://web1.dktechinmsg.com`
- 디케이테크인 스테이징: `https://stg-bizmsg-web.dktechinmsg.com` — **주의: 스테이징도 실제 메시지가 발송됩니다.**
- 금융권(별도 계약): `https://bizmsg-bank.kakaoenterprise.com`
- 카카오엔터프라이즈 레거시 v1.0: `https://bizmsg-web.kakaoenterprise.com` — **본 서버는 v2.0 기준. v1 계약이면 별도 대조 필요.**
- 리셀러(SOLAPI·팝빌 등): **API 스펙이 완전히 달라 본 서버와 호환되지 않습니다**(리셀러 자체 문서 기준 재구현 필요).

---

## 출처 (공식 URL)

- BizMessage API 개요 / 계약 시 Client ID·Secret 발급, 셀프 콘솔 없음, 영업 문의 `contact.dkt@kakaocorp.com`: https://docs.kakaoi.ai/kakao_i_connect_message/bizmessage/api/
- OAuth 토큰(`POST /v2/oauth/token`, client_credentials, Basic 공백구분, refresh/scope 없음): https://docs.kakaoi.ai/kakao_i_connect_message/bizmessage/api/api_reference/oauth/
- 채널 개설 + 사업자/발신 프로필 심사 전제(`sender_key`): https://kakaobusiness.gitbook.io/main/ad/infotalk/audit
- 카카오톡채널 관리자센터(채널 개설·비즈니스 심사 신청 화면): https://center-pf.kakao.com/
- 비즈니스 심사 메뉴 경로(채널 > 비즈니스 심사 > 심사 신청하기, 채널 마스터 권한·서류·심사기간): https://kakaobusiness.gitbook.io/main/channel/start , https://kakaobusiness.gitbook.io/main/channel/run/mychannel
- 알림톡=정보성 한정, 템플릿 사전심사 필수(`template_code`): https://kakaobusiness.gitbook.io/main/ad/infotalk
- 발신번호 사전등록 의무(전기통신사업법 제84조의2): https://www.smsko.co.kr/info_law/law_callback.html , https://solapi.com/guides/kakao-ata-guide

---

## 미확인 / 주의

- **인증 헤더 함정**: 토큰 발급 헤더는 공식 표기 그대로 `Authorization: Basic {CLIENT_ID} {CLIENT_SECRET}` (공백 구분 **평문**) 입니다. 표준 `base64(id:secret)` Basic 으로 인코딩하면 인증 실패. 실제 계약 콘솔 안내와 1회 대조 권고.
- **계약 주체에 따라 발급 위치·스펙이 다름**: 본 가이드는 디케이테크인 v2.0 기준입니다. (a) 카카오엔터프라이즈 v1.0(레거시) (b) 금융권 host (c) 리셀러 계약 중 무엇이냐에 따라 발급 콘솔·host·API 계약이 달라집니다. 리셀러 계약이면 본 서버 비호환.
- **셀프서비스 가입 포털 URL 없음(확인)**: Client ID/Secret 발급용 자동 콘솔/직링크는 존재하지 않으며 영업 문의(`contact.dkt@kakaocorp.com`) 경유로만 발급됨(공식 문서 기준). 채널 개설·비즈니스 심사 신청 화면은 카카오톡채널 관리자센터 `https://center-pf.kakao.com/` 의 **채널 > 비즈니스 심사 > [심사 신청하기]** 로 확인됨. 단 `sender_key`/`template_code`/`sender_no` 의 실제 발급·등록 화면은 **계약한 메시지 플랫폼(디케이테크인) 콘솔** 내부이며, 콘솔 직링크는 계약 계정/워크스페이스별이라 일반 공개 URL이 없음(계약 계정에서 확인).
- **레이트리밋/TPS 수치 미확인**: 공식 수치 없음(요금제·계약별 존재 가능). 대량 발송은 가드 처리됨.
- **카카오엔터프라이즈 v1.0 스펙과 디케이테크인 v2.0 스펙의 1:1 동일 여부 미확인**(v1 문서가 로그인 게이트라 대조 불가).
