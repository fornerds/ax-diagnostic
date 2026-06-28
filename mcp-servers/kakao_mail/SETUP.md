# 키 발급 가이드 — 카카오/다음 메일 (kakao_mail)

> 근거: `_workspace/kakao_mail/02_validation.md` (검증일 2026-06-26, route=build) · `README.md`
> 핵심: 카카오/다음 메일은 **공개 REST API·공식 MCP 커넥터가 없다.** 자동화는 표준 **IMAP(수신)·SMTP(발신)** 로만 가능하며, "키"는 OAuth 토큰이 아니라 **카카오계정 앱 비밀번호**다. 발급은 **전적으로 사용자 수동 작업**이며 서버가 대행할 수 없다.

## 연동 유형 / 인증 방식
- **연동 유형:** 표준 메일 프로토콜 (REST/GraphQL/Webhook/공식 SDK 없음). 로컬 stdio MCP 서버가 IMAP/SMTP 클라이언트를 내장.
- **인증 방식:** SASL **PLAIN/LOGIN** + 사용자명(카카오계정 이메일) + **앱 비밀번호**.
  - OAuth2 / XOAUTH2 토큰 방식 **아님** (XOAUTH2 지원 여부는 미확인 → 미지원 가정으로 스펙 확정).
  - 토큰 교환·갱신 단계 없음. 앱 비밀번호 = 장기 자격증명.
- **전송 보안:** IMAP 993 / SMTP 465, **암묵적 TLS(SSL)** — STARTTLS 아님.
- 출처: 02_validation 인증(확정) 섹션, https://mail-notice.kakao.com/mboard/en/notice/79 , https://hanmail-notice.daum.net/list/907?index=1

## 발급 절차 (스텝)
> 아래 3단계는 모두 사용자가 직접 수행해야 한다(자동화·위임 불가). 2025-01-02 의무화 이후 일반 로그인 비밀번호로는 IMAP/SMTP 접속이 불가하다.

1. **카카오계정 2단계 인증 ON**
   - 경로: 카카오톡 설정 > 카카오계정 > 2단계 인증 (또는 카카오계정 웹).
   - 출처: https://hanmail-notice.daum.net/list/907?index=1 , https://cs.daum.net/m/faq/faqlist/33671
2. **앱 비밀번호 생성**
   - 경로: 2단계 인증 화면 > 앱 비밀번호. 생성된 값을 `KAKAO_MAIL_APP_PASSWORD` 에 사용한다(일반 비밀번호 아님).
   - 출처: https://hanmail-notice.daum.net/list/907?index=1 , https://mail-notice.kakao.com/mboard/en/notice/79
3. **메일 설정에서 IMAP "사용함" ON** (기본 OFF일 수 있음)
   - 카카오메일: 설정 > IMAP/POP3 설정 → "사용함".
   - 다음메일: 설정 > IMAP/POP3 설정 → "사용함".
   - 출처: https://cs.kakao.com/helps?service=156&category=519&locale=ko , https://cs.daum.net/faq/43/9234.html

## 전제조건
- **플랜/요금제:** 없음 (개인 카카오계정으로 충분).
- **사업자 등록·제휴 심사:** 없음 (사업자/제휴 심사 불필요. 출처: https://mail-notice.kakao.com/mboard/en/notice/79).
- **발신번호 사전등록:** 해당 없음 (이메일 연동, SMS 아님).
- **선행 필수:** 위 발급 절차 1~3단계 (2단계 인증 → 앱 비밀번호 → IMAP ON). 셋 다 사용자 수동.
- **제약:** 사용자별 앱 비밀번호 구조라 **멀티테넌트/B2B 위임 연동 불가** — 1인 1계정 로컬 연동 전용.

## 스코프 / 권한
- 앱 비밀번호는 세분화된 스코프가 없다. **메일함 전체 접근권(읽기 + 발송)** 을 가진다.
- 즉 IMAP(목록/검색/읽기/플래그/이동/삭제)과 SMTP(발송) 전 범위가 단일 자격증명으로 가능 → 자격증명 유출 시 위험이 큼.
- 출처: 02_validation "누락·리스크" #4.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|---|---|---|
| `KAKAO_MAIL_USER` | 카카오계정 이메일 주소(예: `me@kakao.com`, `me@daum.net`). IMAP/SMTP 사용자명 겸 From 주소. 도메인으로 호스트 자동판별. | 필수 |
| `KAKAO_MAIL_APP_PASSWORD` | 발급 절차 2단계에서 생성한 **앱 비밀번호**(일반 비밀번호 아님). | 필수 |
| `KAKAO_MAIL_IMAP_HOST` | IMAP 호스트 수동 오버라이드. 미설정 시 도메인 기반 자동판별. 변형/통합계정용. | 선택 |
| `KAKAO_MAIL_SMTP_HOST` | SMTP 호스트 수동 오버라이드. 미설정 시 자동판별. | 선택 |
| `KAKAO_MAIL_IMAP_PORT` | IMAP 포트 오버라이드(기본 993). | 선택 |
| `KAKAO_MAIL_SMTP_PORT` | SMTP 포트 오버라이드(기본 465). | 선택 |

### 호스트 자동판별 (참고)
`KAKAO_MAIL_USER` 도메인으로 서버를 자동 결정한다(오버라이드 env가 우선).

| 계정 도메인 | IMAP(수신) | SMTP(발신) |
|---|---|---|
| `@kakao.com` | `imap.kakao.com` : 993 / SSL | `smtp.kakao.com` : 465 / SSL |
| `@daum.net` / `@hanmail.net` | `imap.daum.net` (별칭 `imap.hanmail.net`) : 993 / SSL | `smtp.daum.net` (별칭 `smtp.hanmail.net`) : 465 / SSL |

그 외 도메인(통합계정·스마트워크 등)은 자동판별 불가 → `KAKAO_MAIL_IMAP_HOST`/`KAKAO_MAIL_SMTP_HOST` 직접 지정.

## 출처 (공식 URL)
- 카카오 개발자센터 REST 레퍼런스(메일 API 부재 확인): https://developers.kakao.com/docs/latest/en/rest-api/reference
- 카카오메일 IMAP/POP3 서버 설정(고객센터): https://cs.kakao.com/helps_html/1073195244?locale=ko
- 카카오메일 IMAP/POP3 카테고리: https://cs.kakao.com/helps?service=156&category=519&locale=ko
- 2단계 인증 의무화 + 앱 비밀번호 공지(카카오, EN): https://mail-notice.kakao.com/mboard/en/notice/79
- 2단계 인증 의무화 공지(다음, 2025-01-02): https://hanmail-notice.daum.net/list/907?index=1
- 앱 비밀번호 설정 후 IMAP/POP3(다음): https://cs.daum.net/m/faq/faqlist/33671
- 다음메일 IMAP/POP3 도움말: https://cs.daum.net/faq/43/9234.html
- 다음메일 IMAP/SMTP 주소: https://cs.daum.net/m/faq/faqlist/17988
- 로그인 실패(앱 비밀번호 필요): https://cs.kakao.com/helps_html/1073200223?locale=ko
- 메일 발송 정책(한도): https://mail.daum.net/policy?category=bulk&tab=guideline
- Anthropic 커넥터 디렉터리 FAQ(공식 커넥터 부재): https://support.claude.com/en/articles/11596036-anthropic-connectors-directory-faq

## 미확인 / 주의
- **XOAUTH2(OAuth 토큰 로그인) 지원 여부 — 미확인.** 어떤 공식 문서도 명시하지 않음. 공식 안내가 "IMAP/POP3는 2단계 인증을 지원하지 않아 앱 비밀번호 사용"이라 명시하므로 스펙은 앱 비밀번호로 확정(XOAUTH2 미지원 가정).
- **앱 비밀번호 생성 화면의 정확한 메뉴 라벨/스크린 흐름**은 카카오/다음 UI 개편에 따라 달라질 수 있음. 위 경로는 공식 공지·도움말 기준이며, 변경 시 출처 URL에서 재확인 필요.
- **일반 비밀번호 사용이 로그인 실패의 가장 흔한 원인.** 인증 실패(AUTHENTICATIONFAILED / SMTP 535) 시 앱 비밀번호 사용·2단계 인증·IMAP "사용함" ON·호스트(kakao vs daum)를 점검.
- **보안:** 앱 비밀번호는 메일함 전체 접근권을 가지므로 환경변수(.env, git 미추적)로만 주입하고 로그/에러에 마스킹. 절대 커밋/공유 금지.
- **발송 한도(공식 확정):** 일 1만 통 / 1회 동시 수신자 150명 / 한 IP 동시 접속 100개 / 메시지 50MB. 스팸 추정 회원은 더 낮게 적용될 수 있음(발급과 무관하나 운영 시 유의).
