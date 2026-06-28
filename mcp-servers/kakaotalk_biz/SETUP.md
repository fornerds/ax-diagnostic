# 키 발급 가이드 — 카카오톡 (kakaotalk_biz)

> 근거: `_workspace/kakaotalk_biz/02_validation.md`(Kakao Developers 공식 문서 라이브 1:1 대조, 2026-06-26) + `mcp-servers/kakaotalk_biz/README.md`.
> 원칙: 추측 금지. 아래는 출처 있는 사실만. 불명확한 부분은 "미확인"으로 남김.
> 범위: 표면 A(카카오톡 메시지 API, OAuth2 + REST)만. 알림톡/친구톡(표면 B, 딜러사 제휴 필수)·개인 대화(표면 C, 공식 API 부재)는 이 커넥터 대상이 아니다.

## 연동 유형 / 인증 방식

- **연동 유형**: 로컬 빌드 MCP 서버(공식 원격 MCP 커넥터 없음). stdio 전송.
- **인증 방식**: OAuth 2.0 Authorization Code Grant — 사용자별 access/refresh token.
  - MCP 서버는 인터랙티브 OAuth 콜백을 받지 않는다. **사전 발급한 access token을 환경변수로 주입**하는 방식.
  - 호출 헤더: `Authorization: Bearer ${KAKAO_ACCESS_TOKEN}`.
  - 인증 서버: `https://kauth.kakao.com` · 자원 서버: `https://kapi.kakao.com`.

## 발급 절차 (스텝)

> **발급 콘솔 직링크**: 앱 관리 콘솔 → **https://developers.kakao.com/console/app** (로그인 후 [내 애플리케이션] 목록). 개별 앱의 설정 화면은 앱별 ID가 URL에 들어가 워크스페이스마다 달라 **앱-단위 직링크는 존재하지 않음** → 아래 콘솔 루트 진입 후 메뉴 경로를 따른다.

1. **앱 등록**: https://developers.kakao.com/console/app 접속(로그인) → 우측 상단 **[애플리케이션 추가하기]** 버튼 → 앱 이름·사업자명 입력 → [저장].
2. **REST API 키 확인**: 생성된 앱 선택 → **[앱] > [플랫폼 키] > [REST API 키]** (구 UI: [앱 설정] > [요약 정보] / [앱 키])에서 REST API 키 확인. → `KAKAO_REST_API_KEY`.
3. **카카오 로그인 활성화 + Redirect URI 등록**:
   - **[카카오 로그인] > [사용 설정]** → 상태를 **[ON]**.
   - Redirect URI: **[앱] > [플랫폼 키] > [REST API 키] > [리다이렉트 URI]**에 콜백 URL 등록(인가코드 받는 주소).
4. **동의항목 설정**: **[카카오 로그인] > [동의항목]** 화면에서
   - `talk_message`: **[접근권한] > [카카오톡 메시지 전송]** 설정(메시지 발송 도구 전부 필수).
   - `friends`(친구 발송까지 쓸 경우): **[개인정보] > [카카오 서비스 내 친구목록(프로필사진, 닉네임, 즐겨찾기 포함)]** 설정.
5. **client_secret 확인**: REST API 키는 **클라이언트 시크릿 기능이 기본 ON**으로 생성된다. **[앱] > [플랫폼 키] > [REST API 키] > [클라이언트 시크릿]**에서 값/상태 확인. ON이면 토큰 발급·갱신에 `client_secret` 필수.
6. **인가코드 요청**:
   `GET https://kauth.kakao.com/oauth/authorize?client_id={REST_API_KEY}&redirect_uri={URI}&response_type=code&scope=talk_message,friends`
   (scope는 쉼표 구분 다중 가능, response_type=code 고정)
7. **토큰 교환**:
   `POST https://kauth.kakao.com/oauth/token`
   form: `grant_type=authorization_code`, `client_id`, `redirect_uri`, `code`, `client_secret`
   → 응답: `access_token`, `refresh_token`, `expires_in`, `refresh_token_expires_in`, `token_type=bearer`.
8. **환경변수 주입**: 받은 `access_token`을 `KAKAO_ACCESS_TOKEN`에 넣는다. 자동 갱신을 쓰려면 `refresh_token`/REST API 키/client_secret도 함께 주입.
9. **(친구 발송 시) 추가 기능 신청**: 비즈 앱 전환 + 비즈니스 정보 심사 후 **[앱] > [추가 기능 신청] > [카카오톡 친구/메시지]**에서 사용 권한 신청·승인(아래 전제조건 참조).
10. **(선택) 토큰 갱신**:
   `POST https://kauth.kakao.com/oauth/token`
   form: `grant_type=refresh_token`, `client_id`, `refresh_token`, `client_secret`
   (응답의 refresh_token은 기존 리프레시 토큰 만료 1개월 미만일 때만 갱신·반환)

## 전제조건

- **"나에게 보내기"(memo 계열 3개 도구)**: 개인 앱에서 즉시 가능. 앱 등록 + 카카오 로그인 활성화 + `talk_message` 동의항목 설정만으로 호출 가능. **별도 심사 없음.**
- **"친구에게 보내기"(friends 계열 4개 도구)**: 다음 게이트 통과 필수 — 미승인 시 호출은 `-402`(insufficient scopes/403) 또는 `-3`(앱키 미허용/403)로 실패.
  1. **비즈 앱 전환** + **비즈니스 정보 심사** 통과.
  2. **[앱] > [추가 기능 신청] > [카카오톡 친구/메시지]**에서 사용 권한 신청·승인. (제출: 서비스 정보/웹·앱 URL, 메시지 전송 적용 화면 스크린샷, 신청 사유·활용 시나리오; 개인정보 마스킹 필수.) 친구목록 권한도 동일하게 **[앱] > [추가 기능 신청]**에서 신청.
  > 메뉴 경로 확정(2026-06 공식 문서): kakaotalk-message/common 및 kakaotalk-social/common 모두 신청 경로를 **[앱] > [추가 기능 신청]**(메시지/친구는 [카카오톡 친구/메시지] 항목)으로 명시. README 라벨과 일치, 02_validation의 "[카카오톡 소셜] 사용 권한 신청" 표기는 구 라벨로 대체.
- **스크랩 발송(memo/friends scrap)**: `request_url` 도메인을 [앱] > [제품 링크 관리] > [웹 도메인]에 등록해야 동작.
- **발신번호 사전등록**: 표면 A에는 해당 없음(SMS/알림톡 영역 아님).

## 스코프 / 권한

| 스코프 | 의미 | 동의 유형 | 출처 |
|--------|------|-----------|------|
| `talk_message` | 카카오톡 메시지 전송(memo·friends 발송 전 도구 필수) | 선택 동의 | https://developers.kakao.com/docs/ko/kakaotalk-message/rest-api |
| `friends` | 카카오 서비스 내 친구목록(닉네임/프로필사진/즐겨찾기 포함) | 선택 동의 | https://developers.kakao.com/docs/ko/kakaotalk-social/rest-api |

> 친구 발송 대상은 **같은 앱에 카카오 로그인 + `friends`/`talk_message` 둘 다 동의한 친구**의 `uuid`뿐. 임의의 전화번호/카카오톡 사용자에게는 발송 불가.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `KAKAO_ACCESS_TOKEN` | 토큰 교환(스텝 5) 응답의 `access_token`. 모든 호출 `Authorization: Bearer`에 사용 | **필수** |
| `KAKAO_REFRESH_TOKEN` | 토큰 교환 응답의 `refresh_token`. 자동 갱신용 | 선택(갱신 사용 시 필수) |
| `KAKAO_REST_API_KEY` | 콘솔 앱의 **REST API 키**. 갱신 시 `client_id`로 사용 | 선택(갱신 사용 시 필수) |
| `KAKAO_CLIENT_SECRET` | 콘솔의 클라이언트 시크릿(기본 ON). 토큰 교환·갱신의 `client_secret` | 선택(시크릿 ON이면 사실상 필수) |

> 시크릿 하드코딩 금지. 전부 `process.env`로만 읽는다. `.env`(커밋 금지) 또는 `.mcp.json`의 `${VAR}` 환경변수 확장 사용. `KAKAO_ACCESS_TOKEN` 부재 시 서버는 즉시 실패.

## 출처 (공식 URL)

- 카카오 로그인 REST API (OAuth authorize/token/refresh · client_secret 기본 ON): https://developers.kakao.com/docs/ko/kakaologin/rest-api
- 메시지 REST API (memo/friends 엔드포인트 · 스코프 talk_message): https://developers.kakao.com/docs/ko/kakaotalk-message/rest-api
- 친구목록 REST API (스코프 friends · uuid · 사용 권한): https://developers.kakao.com/docs/ko/kakaotalk-social/rest-api
- 공통 REST 규격 · 에러 envelope (-401/-402/-3/-10): https://developers.kakao.com/docs/ko/rest-api/reference
- 메시지 에러코드 (-502/-530/-532/-533/-536): https://developers.kakao.com/docs/ko/kakaotalk-message/trouble-shooting
- 쿼터 (일 30,000 / 발신자당 100 / 수신자당 100 / pair당 20 / 월 3,000,000): https://developers.kakao.com/docs/ko/getting-started/quota
- 카카오 로그인 설정하기 (콘솔 메뉴 경로: 사용 설정/동의항목/리다이렉트 URI/클라이언트 시크릿): https://developers.kakao.com/docs/ko/kakaologin/prerequisite
- 앱 설정 (앱 키/플랫폼 키 메뉴 경로): https://developers.kakao.com/docs/ko/app-setting/app
- 카카오톡 메시지 공통 (권한 신청 경로 [앱]>[추가 기능 신청]>[카카오톡 친구/메시지]): https://developers.kakao.com/docs/ko/kakaotalk-message/common
- 카카오톡 소셜 공통 (친구목록 권한 신청 경로 [앱]>[추가 기능 신청]): https://developers.kakao.com/docs/ko/kakaotalk-social/common
- Kakao Developers 앱 관리 콘솔(발급 직링크): https://developers.kakao.com/console/app

## 미확인 / 주의

- **(확정) 친구 발송 권한 신청 콘솔 메뉴 경로**: **[앱] > [추가 기능 신청] > [카카오톡 친구/메시지]** (2026-06 공식 문서 kakaotalk-message/common·kakaotalk-social/common 확인). 02_validation의 "[카카오톡 소셜] 사용 권한 신청"은 구 라벨로, 위 경로로 대체됨.
- **(미확인) `talk_message` 동의항목 "필수/선택" 설정 가능 여부 및 검수 영향**: 런타임 호출 형식에는 영향 없음(콘솔 설정/검수 영역).
- **쿼터 매우 빡빡**: 발신자당 100건/일, pair당 20건/일. 반복 자동 발송 시 빠르게 `-532/-533/-536` 도달. 다음날까지 회복 안 됨 → 재시도 금지.
- **기대-현실 갭(도입 전 필수 고지)**: 임의 지인/단톡방 자동화 불가, 알림톡/친구톡 대량 발송 불가(표면 B 딜러사 제휴 필요), 개인 카톡 대화 자동화 불가(표면 C 공식 API 부재).
- **광고성 친구 발송 규제**: 정보통신망법(수신동의·야간발송 제한) 대상. 정보성 용도 권장.
