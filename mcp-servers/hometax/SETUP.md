# 키 발급 가이드 — 홈택스 / 국세청 (hometax)

> 검증 출처: `_workspace/hometax/02_validation.md` (검증일 2026-06-26, verdict=spec_ready), `mcp-servers/hometax/README.md`
> 대상 데이터셋: 국세청_사업자등록정보 진위확인 및 상태조회 서비스 (data.go.kr publicDataPk=15081808)

## 연동 유형 / 인증 방식

- **연동 유형:** 공공데이터포털(data.go.kr) 오픈 API 데이터셋 활용신청 방식. OAuth/토큰 교환 없음.
- **인증 방식:** 발급받은 인증키(`serviceKey`)를 **URL 쿼리파라미터 `serviceKey`** 로 전달.
- **키 종류 주의:** 일반 인증키 2종(Encoding / Decoding) 중 **Decoding 키**를 사용한다. (SDK 소스 기준 URL 디코딩된 원문 키를 그대로 쿼리에 부착하는 것이 검증된 방식.) Encoding 키만 보유 시 디코딩 후 사용.
- **범위 안내:** 홈택스 본체 기능(세금 신고·납부조회·증명발급·연말정산·전자세금계산서·현금영수증)은 국세청 공식 공개 API가 없어 본 커넥터로 발급/연동 대상이 아니다. 발급 대상은 사업자등록 진위확인·상태조회 데이터셋 1종뿐.

## 발급 절차 (스텝)

1. [data.go.kr 데이터 상세 페이지](https://www.data.go.kr/data/15081808/openapi.do) 접속 후 **로그인** (개인 회원가입만으로 충분).
2. 데이터 15081808 **활용신청** 클릭 → 개발/운영 모두 **자동승인**(심사·제휴 불필요, 무료).
3. 마이페이지 → 해당 활용신청 항목 → **일반 인증키(Decoding)** 복사.
4. 복사한 **Decoding 키**를 환경변수 `NTS_SERVICE_KEY` 에 넣는다.

> 가장 흔한 초기 실패: Encoding 키를 그대로 넣으면 "등록되지 않은 인증키" 오류가 난다 → 반드시 Decoding 키 사용.

## 전제조건

- **사업자 심사 / 제휴 / 플랜:** 없음. 개인 회원가입 → 활용신청 → 즉시 자동승인(개발·운영 모두). 비용 무료.
- **발신번호 사전등록 등:** 해당 없음(이 API와 무관).

## 스코프 / 권한

- 스코프 개념 없음 — **데이터셋 단위 승인**. 활용신청이 승인되면 해당 데이터셋(15081808)의 status/validate 두 오퍼레이션을 호출할 수 있다.
- 권한 한도(레이트리밋): 1회 요청 최대 100건(초과 시 에러), 1일 100만 건(개발계정 한도). 출처: data.go.kr 상세 / 오픈공지.

## 환경변수 매핑

| 변수 | 값을 어디서 | 필수 |
|------|-------------|------|
| `NTS_SERVICE_KEY` | data.go.kr 마이페이지 → 활용신청 → **일반 인증키(Decoding)** | 예 |
| `NTS_BASE_URL` | 베이스 URL 오버라이드(기본값 `https://api.odcloud.kr/api/nts-businessman/v1`) | 아니오 |

## 출처 (공식 URL)

- 데이터셋 개요/활용신청/자동승인/무료: https://www.data.go.kr/data/15081808/openapi.do
- 데이터셋 상세(수정일 2026-05-13, 레이트리밋 100건/요청·1일 100만건): https://www.data.go.kr/tcs/dss/selectApiDataDetailView.do?publicDataPk=15081808
- 레이트리밋 오픈공지: https://www.data.go.kr/bbs/ntc/selectNotice.do?pageIndex=1&originId=NOTICE_0000000002110
- 홈택스 본체 공개 API 부재(국세청 안내): https://www.nts.go.kr/nts/cm/cntnts/cntntsView.do?mi=6690&cntntsId=8102
- 전자세금계산서 발급 방법(공식 발급 API 부재): https://www.nts.go.kr/nts/cm/cntnts/cntntsView.do?mi=2462&cntntsId=7788

> 베이스 URL·엔드포인트·인증키 디코딩 규약은 data.go.kr 본문에 비노출(Swagger에만 게시)이라, 유지보수 중 SDK 소스(PublicDataReader nts.py) 및 다수 커뮤니티 출처로 교차검증됨. 상세는 `_workspace/hometax/02_validation.md` 참조.

## 미확인 / 주의

- **serviceKey를 Authorization 헤더로 전달하는 방식**은 이 API에서 미검증 → 쿼리파라미터 방식만 사용.
- 공식 발급 절차 화면(활용신청 → 인증키 복사)의 표·엔드포인트 표는 data.go.kr 본문 비노출(Swagger UI 클라이언트 렌더링)이라 정적 캡처로 1:1 확보 불가. 값/필드명은 다수 출처 일치하나 "공식 코드표 전량"은 미대조.
- 홈택스 본체/전자세금계산서/현금영수증 자동화가 필요하면 사설 ASP(팝빌)·핀테크 스크래핑(CODEF)의 유료·공동인증서 위임 계약이 별도로 필요(본 커넥터 범위 외, 별도 검토).
