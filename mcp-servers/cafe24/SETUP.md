# 키 발급 가이드 — 카페24 (cafe24)

> 출처: `_workspace/cafe24/02_validation.md`(doc-validator, spec_ready) + 카페24 개발자센터 1회 재확인(2026-06-27).
> 원칙: 출처 있는 사실만. 발급 디테일이 불명확한 항목은 "미확인"으로 명시.

## 연동 유형 / 인증 방식
- **연동 유형:** REST (Admin API), 전송 stdio MCP.
- **인증:** OAuth 2.0 Authorization Code Grant.
  - API 호출: `Authorization: Bearer {access_token}`.
  - 토큰 발급/갱신: `Authorization: Basic base64(client_id:client_secret)`.
- **호스트 분리(함정):** authorize 동의만 `https://{mall_id}.cafe24.com`, token 교환·모든 API 호출은 `https://{mall_id}.cafe24api.com`.
- MCP 서버는 브라우저 리다이렉트를 못 하므로 **OAuth 동의(authorize)는 운영자가 1회 수동**으로 수행해 토큰을 얻고 env로 주입한다. 이후 401 시 서버가 refresh_token으로 자동 갱신한다.
  - 출처: https://developer.cafe24.com/docs/guide/authentication_security_guide.html

## 발급 절차 (스텝)

### A. 앱 등록 → Client ID / Client Secret 발급 (운영자 1회)
1. 카페24 개발자센터 접속: https://developers.cafe24.com/
2. **개발사(개발자) 등록.** 앱 생성 전 개발사 등록이 선행되어야 한다. 자체 운영 쇼핑몰이면 쇼핑몰 운영자 정보를 입력하고 전문분야는 "쇼핑몰 운영"을 선택한다.
   - 출처: https://developers.cafe24.com/app/front/app/develop/oauth/clientid , https://developers.cafe24.com/
3. **앱(애플리케이션) 생성.** 개발사 등록 후 새 앱을 만든다.
4. **OAuth 설정에서 Redirect URI 등록 + 키 확인.** redirect_uri를 등록하고, 발급된 **Client ID(App Key)** 와 **Client Secret** 을 확인한다.
   - 출처: https://developers.cafe24.com/app/front/app/develop/oauth/clientid
5. **사용할 스코프 지정**(아래 "스코프 / 권한" 참조).

> 자체 운영(자기 몰) 비공개 연동은 앱스토어 '판매'가 아니므로 **앱스토어 심사(screening)는 불요**. 카페24 계정으로 앱 등록 후 자기 몰 URL에 권한을 부여하는 방식.
> 출처: https://developers.cafe24.com/app/front/app/develop/oauth/clientid

### B. 최초 토큰 발급 (authorize → token, 1회 수동·MCP 외부)
1. **authorize 동의** — 브라우저에서 접속(scope는 콤마 구분):
   ```
   https://{mall_id}.cafe24.com/api/v2/oauth/authorize?response_type=code&client_id={CLIENT_ID}&redirect_uri={REDIRECT_URI}&scope=mall.read_product,mall.write_product,mall.read_order,mall.write_order,mall.read_customer&state=xyz
   ```
   동의 후 `redirect_uri`로 `?code=...`가 전달된다.
2. **token 교환** — 받은 code로 토큰 발급:
   ```
   curl -X POST "https://{mall_id}.cafe24api.com/api/v2/oauth/token" \
     -H "Authorization: Basic $(printf '%s:%s' "{CLIENT_ID}" "{CLIENT_SECRET}" | base64)" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "grant_type=authorization_code&code={CODE}&redirect_uri={REDIRECT_URI}"
   ```
   응답의 `access_token`(~2시간) → `CAFE24_ACCESS_TOKEN`, `refresh_token`(~2주) → `CAFE24_REFRESH_TOKEN`.
   - 출처: https://developer.cafe24.com/docs/guide/authentication_security_guide.html , https://developers.cafe24.com/app/front/app/develop/oauth/token
3. **자동 갱신:** 호출이 401이면 서버가 `grant_type=refresh_token`으로 1회 재발급 후 재시도. refresh_token도 만료되면(약 2주) A·B를 다시 수행해 토큰을 재주입.

## 전제조건
- **플랜:** 별도 유료 플랜 요구사항 **미확인**(공식 출처에서 특정 플랜 강제 근거 미확보). 카페24 쇼핑몰 계정 보유가 기본 전제.
- **사업자 심사:** 자체 운영 몰의 비공개 연동은 **앱스토어 심사 불요**(앱스토어 '판매' 시에만 심사). 출처: https://developers.cafe24.com/app/front/app/develop/oauth/clientid
- **발신번호 사전등록 등:** 해당 없음(이 커넥터는 상품/주문/회원 조회·쓰기 범위, SMS 발송 아님).
- **개발사 등록:** 필요(앱 생성 선행 단계). 출처: https://developers.cafe24.com/app/front/app/develop/oauth/clientid

## 스코프 / 권한
authorize 요청 시 `scope`(콤마 구분)로 지정. 이 커넥터가 쓰는 스코프:

| 스코프 | 용도 |
|---|---|
| `mall.read_product` | 상품 목록/단건 조회 |
| `mall.write_product` | 상품 등록/수정 |
| `mall.read_order` | 주문 목록/단건 조회 |
| `mall.write_order` | 주문 처리상태 변경 |
| `mall.read_customer` | 회원(고객) 목록 조회 |

- 출처: https://developers.cafe24.com/docs/api/admin/

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|---|---|---|
| `CAFE24_MALL_ID` | 쇼핑몰 서브도메인. `{mall_id}.cafe24api.com`의 `{mall_id}` (예: `mystore.cafe24api.com` → `mystore`) | 예 |
| `CAFE24_CLIENT_ID` | 개발자센터 앱 OAuth 설정의 **Client ID(App Key)** | 예 |
| `CAFE24_CLIENT_SECRET` | 개발자센터 앱 OAuth 설정의 **Client Secret** | 예 |
| `CAFE24_ACCESS_TOKEN` | 절차 B(token 교환) 응답의 `access_token` (~2시간) | 예 |
| `CAFE24_REFRESH_TOKEN` | 절차 B(token 교환) 응답의 `refresh_token` (~2주) | 예 |
| `CAFE24_API_VERSION` | API 버전 헤더. 기본값 `2026-03-01` (날짜 버저닝) | 아니오(기본값 제공) |
| `CAFE24_ALLOW_DESTRUCTIVE` | `true`일 때만 delete_* 노출(현재 계약엔 delete 도구 없음) | 아니오(기본 false) |

> 시크릿은 코드/`.mcp.json`에 실값 금지 — 환경변수로 주입하고 `.mcp.json`은 `${VAR}` 확장 사용.

## 출처 (공식 URL)
- 앱 등록·Client ID/Secret 발급: https://developers.cafe24.com/app/front/app/develop/oauth/clientid
- 개발자센터 홈: https://developers.cafe24.com/
- 인증/보안 가이드(authorize·token·refresh·토큰 수명): https://developer.cafe24.com/docs/guide/authentication_security_guide.html
- token 발급 레퍼런스: https://developers.cafe24.com/app/front/app/develop/oauth/token
- Admin API 가이드(베이스 URL·버전 헤더·스코프·페이지네이션): https://developers.cafe24.com/docs/api/admin/
- 레이트리밋·에러·공통 규약: https://developers.cafe24.com/docs/api/

## 미확인 / 주의
- **요구 플랜:** 특정 유료 플랜 강제 여부 공식 근거 미확보 → **미확인**.
- **자체몰 특화 등록 절차 세부:** 개발자센터 앱 등록 페이지가 SPA라 단계별 화면 캡션까지는 추출 제한. 큰 흐름(개발사 등록 → 앱 생성 → OAuth 설정에서 redirect URI 등록·Client ID/Secret 확인)은 확인됨. 세부 화면 라벨은 페이지 직접 열람 권장.
- **토큰 수명:** access ~2시간, refresh ~2주 — "정책에 따라 변동 가능". 출처: 인증/보안 가이드.
- **버전 헤더 고정 필수:** 모든 요청에 `X-Cafe24-Api-Version: 2026-03-01`. 미지정 시 앱 설정 버전이 적용되어 동작이 달라질 수 있음.
- **호스트 분리 주의:** authorize=`{mall_id}.cafe24.com`, token·API=`{mall_id}.cafe24api.com`. 혼동 시 토큰 교환/호출 실패.
- **MCP는 토큰 자동 발급 불가:** authorize 동의는 운영자 1회 수동. MCP는 refresh 자동 갱신만 담당.
- **계약 제외 작업:** 회원 생성/수정(공식 근거 미확인), 모든 delete(파괴적·일부 path 미확인), 서브리소스(주문 items/cancellation, 고객 memos 등 경로 미확인).
