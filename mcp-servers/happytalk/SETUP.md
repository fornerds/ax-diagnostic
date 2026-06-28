# 키 발급 가이드 — 해피톡 (happytalk)

> 근거: `_workspace/happytalk/02_validation.md`(검증일 2026-06-26, 공식 개발자센터 sitemap 전수 대조) + 공식 개발자센터 재확인(2026-06-27).
> 핵심: **인증키는 셀프발급 불가 — 해피톡 고객센터 협의 후 발급**. 시크릿이 세 종류로 나뉘며 도구 그룹별로 필요한 것만 있으면 됩니다.

---

## 연동 유형 / 인증 방식

공식 MCP 커넥터 없음 → 공개 REST API 기반 **로컬 빌드 커넥터**. 토큰 흐름은 전부 **사전 발급 정적 시크릿**(OAuth 토큰 교환 없음). 시크릿 체계가 3종 공존하며, 호출하는 도구 그룹에 따라 필요한 시크릿이 다릅니다.

| 방식 | 적용 표면(호스트) | 인증 형태 |
|------|------------------|-----------|
| A — HT-Client 키쌍 | 카카오 웹훅(`kakao-api`), Biz API(`biz-api`) | 헤더 `HT-Client-Id: <key>` + `HT-Client-Secret: <token>` (+ `Content-Type: application/json`) |
| B — token | customer(`customer.happytalk.io`), bnd(`bnd.happytalk.io`) | 요청 **바디**에 `token: <고객사 토큰>` (헤더 아님) |
| C — JWT 발급(보조) | `happytalk.io` | 헤더 `Authorization: <고유키>` → `POST /api_v1/out_side_auth/auth/format/json` (서버-투-서버). v1 도구 미제공 |

> 방식 A/B 시크릿은 LLM 입력으로 받지 않고 **서버가 환경변수로 주입**합니다.

---

## 발급 절차 (스텝)

해피톡은 디벨로퍼 센터 API용 인증 정보를 **자동 발급 콘솔로 제공하지 않습니다.** 공식 문구:

> "디벨로퍼 센터에서 제공하는 API 이용을 위해서는 인증키 등의 정보가 필요하며, 해당 인증 정보는 **사용 목적 확인 및 조건 협의 후 제공**됩니다. 자세한 내용은 고객센터로 문의해주세요."
> (출처: https://developer-center.happytalk.io/Biz-API/ , https://developer-center.happytalk.io/KakaoWebhook/ — 2026-06-27 재확인)

### 방식 A — HT-Client 키쌍 (카카오 / Biz)
> **셀프발급 콘솔 없음(확인).** 디벨로퍼 센터에 자동 발급 콘솔/버튼이 없으며, 고객센터 협의 채널로만 발급됩니다. 발급 페이지 **직링크 없음** → 아래 고객센터 채널로 진행.

1) 해피톡 고객센터로 API 이용(카카오 웹훅 또는 Biz API) 목적·조건을 문의·협의한다.
   - 문의 페이지(직링크): https://happytalk.io/contact
   - 전화: 1666-5263 · 이메일: happytalk@happytalk.io
2) 협의 후 해피톡이 `HT-Client-Id`(key)와 `HT-Client-Secret`(token)을 발급한다.
3) 발급받은 키쌍을 환경변수에 설정한다(아래 매핑 표).
> 발급 화면 직링크(셀프 콘솔)는 존재하지 않음 — 발급은 위 고객센터 채널 협의를 통해서만 진행됩니다.

### 방식 B — customer token (상담내역 조회·전화번호·상담원 도구)
1) 해피톡 관리자(어드민) 로그인.
2) **설정 > 회사정보관리 > 시스템정보 > TOKEN** 에서 고객사 토큰 값을 확인한다.
   (출처: https://developer-center.happytalk.io/Happytalk/customer/counsel_detail_search/counsel_room_list/ )
3) 그 값을 환경변수에 설정한다.

### 방식 C — JWT 고유키 (선택, v1 도구 미제공)
1) 해피톡 어드민의 **인증관리 > 고객정보요청 연동**에서 `Authorization` 고유키를 별도 발급.
   (출처: https://developer-center.happytalk.io/Happytalk/advanced/authentication_management/customer_info_request/JWT_issuance/ )
2) JWT 발급은 서버-투-서버 보조 기능으로, 현재 커넥터 v1에는 도구로 노출되지 않음.

---

## 전제조건

- **계약·키 발급 선결.** 빌드/기동은 키 없이도 되지만, 실호출에는 해피톡 계약 및 고객센터 협의를 통한 키/토큰 발급이 선결입니다.
- **카카오 표면(`send_kakao_message`, `get_kakao_sender_profile`)은 추가 심사 선행:**
  - 카카오 비즈채널 전환(심사 영업일 약 2~3일).
  - 발신프로필(senderKey) 등록.
  - 동일 상담톡이 타 솔루션에 이미 연동돼 있으면 **중복연동 오류** 가능.
  - (참고 출처: 카카오 비즈채널 가이드 https://blog.bizgo.io/howto/create-kakaotalk-business-channel-guide-2026/ , NHN Cloud https://docs.nhncloud.com/ko/Notification/KakaoTalk%20Bizmessage/ko/sender-overview/ )
- **customer/Biz 도구**(상담내역 조회·발송 등) 자체에는 별도 플랜/사업자 심사 요건이 문서에 명시돼 있지 않음 — 위 키 발급 협의에 포함됨. 단독 추가 전제조건: **없음(문서 기준)**.

---

## 스코프 / 권한

- 별도의 OAuth scope 개념 없음. **시크릿 종류 = 접근 가능 표면**이 곧 권한 경계입니다.
  - HT-Client 키쌍 → 카카오 웹훅 2종 + Biz API 8종.
  - customer token → customer 4종 + bnd 읽기/배정 도구.
  - JWT 고유키 → JWT 발급(보조).
- **파괴적 작업**: `assign_counselor`, `auto_assign_counselor`(상담원 배정, PUT)는 token 보유 시 가능하나, 커넥터에서 `HAPPYTALK_ENABLE_DESTRUCTIVE=true` 게이트로 기본 비활성.
- 세분화된 역할/권한 등급(어드민 vs 상담원 등)이 customer token 발급 단위로 갈리는지는 문서상 **미확인**.

---

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|------------|------|
| `HAPPYTALK_HT_CLIENT_ID` | 고객센터 협의 후 발급되는 HT-Client **key** | 카카오·Biz 도구 사용 시 |
| `HAPPYTALK_HT_CLIENT_SECRET` | 고객센터 협의 후 발급되는 HT-Client **token** | 카카오·Biz 도구 사용 시 |
| `HAPPYTALK_CUSTOMER_TOKEN` | 어드민 **설정 > 회사정보관리 > 시스템정보 > TOKEN** | 상담내역 조회·전화번호·상담원 도구 사용 시 |
| `HAPPYTALK_JWT_AUTH_KEY` | 어드민 **인증관리 > 고객정보요청 연동**에서 발급한 `Authorization` 고유키 | 선택(v1 도구 미제공) |
| `HAPPYTALK_ENV` | `production`(기본) 또는 `patch` — 호스트 5종 일괄 토글 | 선택 |
| `HAPPYTALK_ENABLE_DESTRUCTIVE` | 직접 설정값 `true` — 파괴적 배정 2종 등록 | 선택(기본 비활성) |

> 시크릿 그룹이 없으면 해당 그룹 도구는 등록되지 않습니다(서버는 정상 기동). 시크릿 실값을 `.mcp.json`에 직접 쓰지 말고 `${VAR}` 확장을 사용하세요.

---

## 출처 (공식 URL)

- Biz API 개요·키 발급 문구: https://developer-center.happytalk.io/Biz-API/
- 카카오 웹훅 개요·키 발급 문구: https://developer-center.happytalk.io/KakaoWebhook/
- 키 발급 문의 채널(고객센터/도입문의, 전화 1666-5263·이메일 happytalk@happytalk.io): https://happytalk.io/contact , https://happytalk.io/support
- customer token 위치(설정>회사정보관리>시스템정보>TOKEN): https://developer-center.happytalk.io/Happytalk/customer/counsel_detail_search/counsel_room_list/
- JWT 고유키 발급(인증관리>고객정보요청 연동): https://developer-center.happytalk.io/Happytalk/advanced/authentication_management/customer_info_request/JWT_issuance/
- 카카오 비즈채널/발신프로필(참고): https://blog.bizgo.io/howto/create-kakaotalk-business-channel-guide-2026/ , https://docs.nhncloud.com/ko/Notification/KakaoTalk%20Bizmessage/ko/sender-overview/

---

## 미확인 / 주의

- **셀프발급 콘솔 URL 없음(확인).** HT-Client 키쌍은 디벨로퍼 센터에 자동 발급 콘솔/버튼이 없고 고객센터 협의 채널로만 발급됩니다. 발급 페이지 직링크는 존재하지 않음. 단, 고객센터/도입문의 채널은 확인됨 — 문의 페이지 https://happytalk.io/contact , 전화 1666-5263, 이메일 happytalk@happytalk.io.
- **운영사 정확한 법인명/사업자등록 미확인**(브리프 "㈜엠비아이솔루션" 추정, 공식 푸터 미확정).
- **customer token 권한 세분화 여부 미확인**(어드민/상담원 등 역할별 token 차등 발급 근거 없음).
- **레이트리밋 수치 미확인**(전 표면 문서에 RPS/쿼터 명시 없음 → 런타임 가드).
- **호스트가 4~5개로 갈림**: customer/bnd/biz-api/kakao-api/(happytalk.io). 단일 BASE_URL로 묶으면 깨짐 — 코드 상수 + `HAPPYTALK_ENV`로만 prod/patch 토글.
- 카카오 표면은 키 발급 후에도 비즈채널 심사·발신프로필 등록이 끝나야 실동작.
