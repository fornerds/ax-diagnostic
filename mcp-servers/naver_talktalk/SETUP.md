# 키 발급 가이드 — 네이버 톡톡 (naver_talktalk)

> 본 가이드는 `_workspace/naver_talktalk/02_validation.md`(검증일 2026-06-26)와 서버 `README.md`의 **출처 있는 사실만**으로 작성됨. 추측 항목은 "미확인"으로 표기.

## 연동 유형 / 인증 방식

- **연동 유형:** 네이버 톡톡 챗봇 API **V1** (REST 발송). 공식 원격 MCP 커넥터는 없으며, 본 커넥터는 발송 전용 로컬 stdio MCP 서버다.
- **인증 방식:** 파트너센터에서 발급한 **자체 발급 토큰 1개**를 HTTP `Authorization` 헤더에 **원문 그대로** 전달한다.
  - **`Bearer ` 접두어를 붙이지 않는다.** (공식 예시: `Authorization: ct_wc8b1i_Pb1AXDQ0RZWuCccpzdNL`)
  - OAuth2를 사용하지 않으며, 스코프 개념이 없다(계정 단위 단일 토큰).
- 출처: https://raw.githubusercontent.com/navertalk/chatbot-api/master/README.md

## 발급 절차 (스텝)

**발급 콘솔(직링크):** 네이버 톡톡 파트너센터 — **https://partner.talk.naver.com/**
> 워크스페이스(계정)별 발급 페이지로 바로 가는 딥링크는 존재하지 않는다. 위 콘솔에 로그인 후 아래 메뉴 경로를 따라 발급한다.

1. **파트너센터 계정 생성 및 검수 통과**
   - https://partner.talk.naver.com/ 에 접속해 단체/사업자(또는 대표성 있는 개인) 계정을 생성한다.
   - 계정 상태가 **"검수중" → "사용중"**(SMS 통지)으로 전환되어야 한다. **즉시 발급 불가.**
2. **챗봇 API 신청 + 이용약관 동의**
   - 콘솔 로그인 후 **화면 상단 [내 계정관리] → 개발자도구 > 챗봇 API 설정**으로 이동한다.
   - **[챗봇 API 신청하기]** 버튼을 눌러 신청하고 이용약관에 동의한다.
3. **보내기 API 토큰 생성**
   - 약관 동의 후 설정 화면에서 **보내기 API 항목의 `Authorization` 필드 [생성]** 버튼을 클릭해 토큰을 발급한다.
   - 전체 경로: **로그인 → [내 계정관리] → 개발자도구 > 챗봇 API 설정 → 보내기 API > Authorization [생성]**.
   - 같은 화면에서 **파트너 ID**(`partner`)와 `Authorization` 값을 함께 확인할 수 있다.
4. **환경변수에 주입**
   - 발급된 토큰을 `NAVER_TALKTALK_AUTH_TOKEN` 환경변수로 주입한다.
   - 토큰 노출 시 파트너센터에서 reset(재발급) 후 env 값을 교체한다.

> 위 1~2단계가 끝나기 전에는 어떤 도구도 `resultCode=01`(Authorization 오류)로 실패한다.

출처: https://partner.talk.naver.com/ , https://github.com/navertalk/chatbot-api , https://guide.ncloud-docs.com/docs/chatbot-chatbot-5-3

## 전제조건

- **파트너센터 계정 검수 통과 필수** ("검수중" → "사용중" 전환). 즉시 키 발급 아님.
- **챗봇 API 별도 신청 + 이용약관 동의** 필요.
- **사업자 심사:** 파트너센터 계정 검수가 사실상 이에 해당(단체/사업자 또는 대표성 있는 개인 계정 요구). 별도의 추가 심사 절차 명시는 **미확인**.
- **발신번호 사전등록:** 해당 없음. 본 API는 전화번호 기반 푸시(알림톡/문자) 모델이 아니다. 발송 대상은 톡톡 내부 `user` 식별자이며, 이는 사용자가 먼저 말을 걸 때 **Webhook 이벤트로만** 획득된다(콜드 아웃바운드 불가).
- **플랜(요금제) 조건:** 공식 문서에 명시 없음 → **미확인**.

출처: https://github.com/navertalk/chatbot-api

## 스코프 / 권한

- **스코프 없음.** OAuth2 미사용, 계정 단위 단일 토큰 하나로 발송 권한을 가진다.
- 별도 권한 범위 선택/동의 화면 없음.
- 출처: https://github.com/navertalk/chatbot-api

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `NAVER_TALKTALK_AUTH_TOKEN` | https://partner.talk.naver.com/ 로그인 → [내 계정관리] → 개발자도구 > 챗봇 API 설정 > 보내기 API > Authorization [생성]로 발급된 토큰. `Authorization` 헤더에 **원문**(Bearer 없음) | 예 |
| `NAVER_TALKTALK_BASE_URL` | 베이스 URL 오버라이드. 미지정 시 기본 `https://gw.talk.naver.com` | 아니오 |
| `NAVER_TALKTALK_DEFAULT_USER` | 테스트/단일 대화용 기본 `user` 식별자(Webhook 수신으로 획득한 값). 미지정 시 도구 호출마다 `user` 인자 필수 | 아니오 |

## 출처 (공식 URL)

- 파트너센터 콘솔(발급 화면 진입): https://partner.talk.naver.com/
- 챗봇 API V1 저장소(README/인증/엔드포인트/에러코드): https://github.com/navertalk/chatbot-api
- README 원문(인증 헤더·발송 엔드포인트): https://raw.githubusercontent.com/navertalk/chatbot-api/master/README.md
- NCloud 가이드(콘솔 URL·토큰 생성 메뉴 경로 교차확인, 2026-06-25 갱신): https://guide.ncloud-docs.com/docs/chatbot-chatbot-5-3

## 미확인 / 주의

- **파트너센터 콘솔 직접 URL:** **확인됨** — https://partner.talk.naver.com/ (NCloud 공식 가이드 교차확인). 단, 워크스페이스(계정)별 발급 페이지로 직행하는 딥링크는 존재하지 않으며, 콘솔 로그인 후 "[내 계정관리] → 개발자도구 > 챗봇 API 설정 > 보내기 API > Authorization [생성]" 메뉴 경로로 발급한다.
- **플랜/요금 조건, 추가 사업자 심사 절차:** 공식 문서에 명시 없음 → **미확인**.
- **레이트리밋/TPS:** 공식 문서에 수치 없음 → **미확인**. 정확한 한도는 파트너센터/네이버에 문의.
- **API V2 / 공식 SDK:** 현재 없음으로 간주(폐기·이전 공지 없음, 공식 SDK 부재).
- **운영 주의:** 본 MCP는 발송 전용. Webhook 수신·대화 이력 저장은 범위 밖이며, `request_user_profile` 결과(프로필 값)도 Webhook으로만 도착하므로 본 MCP에서 회수 불가.
