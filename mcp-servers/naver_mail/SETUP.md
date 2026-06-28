# 키 발급 가이드 — 네이버 메일 (naver_mail)

> 대상: 소비자용 네이버 메일(`mail.naver.com` / 개인 `@naver.com` 계정).
> 네이버는 소비자 메일용 공개 REST/OAuth API와 공식 원격 MCP 커넥터를 제공하지 않는다. 따라서 "키 발급"은 표준 메일 클라이언트용 **애플리케이션(앱) 비밀번호** 발급을 의미한다.
> NAVER WORKS Mail(B2B OAuth)·NCP Cloud Outbound Mailer(발송전용 REST)와는 **다른 제품**이다. 이 가이드는 개인 `@naver.com` 계정 전용이다.

## 연동 유형 / 인증 방식

- **연동 유형**: REST/OAuth 아님. 표준 **IMAP(수신/검색/상태변경) + SMTP(발송)** 프로토콜 인증.
- **인증 방식**: 사용자명 = 전체 이메일 주소(`<id>@naver.com`), 비밀번호 = **애플리케이션(앱) 비밀번호**(일반 로그인 비밀번호 아님). 전송은 SSL/TLS.
- **토큰 흐름**: OAuth 없음 → 스코프 개념 없음. 사용자가 사전 발급한 앱 비밀번호를 서버가 환경변수로 읽어 각 IMAP/SMTP 세션에 그대로 사용한다(장기 유효 자격증명).

## 발급 절차 (스텝)

> **직링크 안내**: 네이버는 앱 비밀번호 콘솔/IMAP 설정 화면에 대한 **로그인 직후 바로 열리는 안정적 직링크(deep-link)를 공개하지 않는다**(해당 화면은 세션/인증으로 보호됨). 따라서 진입은 아래 **기본 URL + 메뉴 경로**로 한다. (복수 3자 출처에서도 직링크 없이 동일한 메뉴 경로만 안내됨.)

1. **2단계 인증 활성화** — 네이버 ID 로그인 페이지 `https://nid.naver.com/` 접속·로그인 → **보안설정** → **2단계 인증**을 켠다. (앱 비밀번호 생성의 선행 조건)
2. **애플리케이션 비밀번호 생성** — 같은 `https://nid.naver.com/` 의 **보안설정** → **2단계 인증 '관리'**(버튼) → **애플리케이션 비밀번호 관리** → 종류선택에서 이름 입력 → **'생성하기'**(버튼). 생성된 비밀번호 값을 즉시 복사해 둔다(`NAVER_MAIL_APP_PASSWORD` 로 사용). **한 번 닫으면 다시 확인 불가 — 분실 시 재생성**해야 한다.
3. **메일 환경설정에서 IMAP 사용함 켜기** — 네이버 메일 `https://mail.naver.com/` 접속 → 왼쪽 메뉴 하단 **환경설정** → 상단 **POP3/IMAP 설정** → **'IMAP 사용함'** 활성화 → **저장**. (이 설정이 꺼져 있으면 인증에 성공해도 IMAP 접속이 거부된다.)
4. 발급한 값을 환경변수로 주입한다(아래 "환경변수 매핑" 참조). 일반 로그인 비밀번호로는 **2025-11-18 유예 종료 이후 외부 클라이언트 접속이 불가**하다.

## 전제조건

- 유료 플랜 / 사업자 심사 / 제휴 승인 / 발신번호 사전등록: **없음** (개인 네이버 계정으로 가능).
- 단, **2단계 인증 활성화**가 앱 비밀번호 생성의 필수 선행 조건이다.
- 참고: 단체(조직) 아이디도 2025-10-15부터 앱 비밀번호 생성이 지원된다(본 서버는 개인 `@naver.com` 대상이라 영향은 제한적).

## 스코프 / 권한

- OAuth 스코프 개념 **없음**(프로토콜 인증). 권한 세분화·읽기전용 스코프 불가.
- 앱 비밀번호 1개 = **계정 전체 메일함 읽기 / 발송 / 삭제 권한**.
- 보안 주의: 권한 분리가 불가능하므로 반드시 환경변수/시크릿 스토어로만 보관하고 코드·로그·공유 파일에 노출하지 말 것. 파괴적 도구(`delete_email`)는 신중히 사용.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 | 기본값 |
|------|------------|------|--------|
| `NAVER_MAIL_USER` | 전체 이메일 주소(`<id>@naver.com`). IMAP/SMTP 사용자명. | 예 | — |
| `NAVER_MAIL_APP_PASSWORD` | 발급 절차 2번에서 생성한 **애플리케이션 비밀번호**(일반 로그인 비번 아님) | 예 | — |
| `NAVER_IMAP_HOST` | IMAP 호스트(override용) | 아니오 | `imap.naver.com` |
| `NAVER_IMAP_PORT` | IMAP 포트(implicit SSL/TLS) | 아니오 | `993` |
| `NAVER_SMTP_HOST` | SMTP 호스트(override용) | 아니오 | `smtp.naver.com` |
| `NAVER_SMTP_PORT` | SMTP 포트(465=implicit TLS, 587=STARTTLS) | 아니오 | `465` |

서버 엔드포인트(참고):

| 용도 | 호스트 | 포트 | 보안 |
|------|--------|------|------|
| IMAP (수신/검색/상태변경) | `imap.naver.com` | 993 | implicit SSL/TLS |
| SMTP (발송, 기본) | `smtp.naver.com` | 465 | implicit TLS |
| SMTP (발송, 대안) | `smtp.naver.com` | 587 | STARTTLS |

## 출처 (공식 URL)

- 정책 공지(외부 클라이언트 일반 비번 차단, 2025-06-24 시행 / 2025-11-18 유예 종료, 앱 비밀번호 필수) — 네이버 1차 공지 URL: `https://notice.naver.com/notices/mail/22462` (존재 확인. 단 notice/help/nid.naver.com 도메인은 검증 환경에서 직접 fetch 차단 → 원문 캡처는 3자 경유).
- 앱 비밀번호 필수·발급 경로(메뉴 경로·버튼명 3자 교차 확인. 직링크는 어느 출처에도 없고 `https://nid.naver.com/` + 메뉴 경로로만 안내됨): `https://www.enterapps.kr/notice/?bmode=view&idx=168294489` , `https://day2life.zendesk.com/hc/ko/articles/360035370933` , `https://servertrix.com/1879` , `https://imweb.me/faq?mode=view&category=29&category2=34&idx=72049`
- 메일 IMAP 사용 설정 메뉴 경로(`mail.naver.com` → 환경설정 → POP3/IMAP 설정 → 사용함 → 저장. 직링크 없음): `https://imweb.me/faq?mode=view&category=29&category2=34&idx=68667`
- IMAP/SMTP 서버·포트 값(복수 독립 출처 일치): `https://smtpedia.com/naver-com-email-settings-pop3-imap-smtp/` , `https://www.getmailbird.com/setup/access-naver-com-via-imap-smtp` , `https://www.serversettings.email/naver.com-email-server-settings-imap.php`
- 소비자 메일용 공개 Open API 부재(공식 Open API 목록): `https://naver.github.io/naver-openapi-guide/apilist.html`

## 미확인 / 주의

- **네이버 1차 문서 원문 미확보**: notice.naver.com / help.naver.com / nid.naver.com 모두 검증 환경에서 직접 fetch가 차단되었다. 서버/포트/정책/발급경로 값은 복수의 독립 3자 출처가 일치하여 신뢰도가 높으나, **네이버 고객센터에서 서버/포트·정책 최신 여부를 한 번 재확인**하는 것을 권장한다.
- **앱 비밀번호 콘솔·IMAP 설정 화면 직링크(deep-link)**: **존재하지 않음(확인)**. 두 화면 모두 세션/인증 보호로, 어느 출처도 직링크를 제공하지 않고 `https://nid.naver.com/`·`https://mail.naver.com/` 기본 URL + 메뉴 경로로만 안내한다 → 본 가이드도 "기본 URL + 메뉴 경로" 방식으로 기재.
- **help.naver.com POP3/IMAP 설정 *도움말 문서* 1차 URL**: 미확인(네이버 1차 도움말 문서 자체의 URL. 단 설정 메뉴 경로·버튼명과 서버/포트 값은 3자 복수 출처로 검증됨).
- **SMTP 발송 한도 공식 수치**: 미확인. 네이버 1차 공식 수치 없음 → 클라이언트 측 임의 한도 강제 금지, SMTP 응답코드(4xx 일시 / 5xx 영구)로 처리.
- **한글 폴더(보낸메일함/스팸 등)의 실제 IMAP 경로**: 미확인(환경 의존). 하드코딩 금지 — `list_folders` 또는 special-use(`\Sent`/`\Trash`/`\Junk`)로 런타임 해석.
- 일반 로그인 비밀번호로는 2025-11-18 유예 종료 이후 외부 클라이언트 접속 불가. 인증 실패 시 (1) 2단계 인증, (2) 앱 비밀번호, (3) IMAP 사용 설정을 점검할 것.
