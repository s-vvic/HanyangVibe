# 1. 문서 개요

## 1.1 목적

본 문서는 보안 WebCam 시스템을 구성하는 6개 부체계 간 인터페이스를 정의함.  
각 부체계 간 통신 프로토콜, 데이터 형식, 메시지 구조, 오류 처리 방법을 기술함.

## 1.2 부체계 간 인터페이스 목록

| 인터페이스 ID | 송신 부체계               | 수신 부체계               | 통신 방식                 |
| ------------- | ------------------------- | ------------------------- | ------------------------- |
| ICD-01        | 안드로이드 앱/웹 대시보드 | 클라우드 서버             | REST API (HTTPS)          |
| ICD-02        | 웹캠 보드                 | 클라우드 서버             | REST API (HTTP, VPN 내부) |
| ICD-03        | 클라우드 서버             | 웹캠 보드                 | REST API (HTTP, VPN 내부) |
| ICD-04        | 웹캠 보드                 | 클라우드 서버 (mediamtx)  | RTSP over VPN             |
| ICD-05        | 클라우드 서버 (mediamtx)  | 안드로이드 앱/웹 대시보드 | RTSP over VPN             |
| ICD-06        | 모든 부체계               | VPN 게이트웨이            | WireGuard (UDP)           |
| ICD-07        | 클라우드 서버             | 안드로이드 앱/웹 대시보드 | Push 알림 (FCM)           |
| ICD-08        | 클라우드 서버             | DB                        | SQL (내부 연결)           |

## 1.3 공통 규격

| 항목           | 규격                                      |
| -------------- | ----------------------------------------- |
| 통신 암호화    | WireGuard VPN 내부 통신 / 외부 구간 HTTPS |
| 데이터 형식    | JSON (UTF-8)                              |
| 인증 방식      | JWT Bearer Token                          |
| 토큰 유효기간  | 1년 (525,600분)                           |
| 날짜/시간 형식 | ISO 8601 (YYYY-MM-DDTHH:mm:ssZ)           |
| 영상 프로토콜  | RTSP (TCP)                                |
| VPN 프로토콜   | WireGuard (UDP 443)                       |

# 2. ICD-01: 앱·웹 대시보드 ↔ 클라우드 서버

베이스 URL: [-](-)  
인증: Authorization: Bearer {JWT_TOKEN} 헤더 포함 (로그인 제외)

## 2.1 인증 API

### 2.1.1 로그인

| 항목          | 내용                                       |
| ------------- | ------------------------------------------ |
| 메서드 / 경로 | -                                          |
| 설명          | 이메일 비밀번호로 로그인하여 JWT 토큰 발급 |

▶ 요청 본문 (Request Body)

| 필드명     | 타입    | 필수 | 설명                             |
| ---------- | ------- | ---- | -------------------------------- |
| email      | String  | Y    | 사용자 이메일                    |
| password   | String  | Y    | 비밀번호                         |
| auto_login | Boolean | N    | 자동 로그인 여부 (기본값: false) |

▶ 응답 본문 (Response Body) 200 OK

| 필드명       | 타입   | 설명            |
| ------------ | ------ | --------------- |
| access_token | String | JWT 액세스 토큰 |
| token_type   | String | Bearer 고정     |
| user_id      | String | 사용자 고유 ID  |
| name         | String | 사용자 이름     |
| email        | String | 사용자 이메일   |

▶ 오류 코드

| HTTP | 코드 | 오류 코드           | 설명                        |
| ---- | ---- | ------------------- | --------------------------- |
| 400  |      | INVALID_CREDENTIALS | 이메일 또는 비밀번호 불일치 |
| 429  |      | TOO_MANY_REQUESTS   | 로그인 시도 횟수 초과       |

### 2.1.2 로그아웃

| 항목        | 내용                  |
| ----------- | --------------------- |
| 메서드/경로 | -                     |
| 설명        | 현재 세션 토큰 무효화 |

### 2.1.3 회원가입

| 항목          | 내용                  |
| ------------- | --------------------- |
| 메서드 / 경로 | -                     |
| 설명          | 신규 사용자 계정 생성 |

요청 본문

| 필드명   | 타입   | 필수 | 설명                      |
| -------- | ------ | ---- | ------------------------- |
| email    | String | Y    | 사용자 이메일 (중복 불가) |
| password | String | Y    | 비밀번호 (8자 이상)       |
| name     | String | Y    | 사용자 이름               |

## 2.2 보드 관리 API

### 2.2.1 보드 목록 조회

| 항목          | 내용                                     |
| ------------- | ---------------------------------------- |
| 메서드 / 경로 | -                                        |
| 설명          | 로그인 사용자의 등록 보드 전체 목록 조회 |

▶ 응답 본문 - 200 OK (배열)

| 필드명            | 타입   | 설명                             |
| ----------------- | ------ | -------------------------------- |
| device_id         | String | 보드 고유 ID (예: CAM-0001)      |
| name              | String | 보드 이름                        |
| location          | String | 설치 위치                        |
| serial_no         | String | 시리얼 번호                      |
| model             | String | 보드 모델명                      |
| firmware_version  | String | 펌웨어 버전                      |
| status            | String | online / offline                 |
| vpn_status        | String | connected / disconnected / error |
| vpn_ip            | String | 보드에 할당된 VPN IP             |
| last_seen         | String | 마지막 접속 시간 (ISO 8601)      |
| latest_event_time | String | 최근 이벤트 발생 시간 (ISO 8601) |
| thumbnail_url     | String | 최근 스냅샷 이미지 URL           |

### 2.2.2 보드 등록

| 항목        | 내용                                             |
| ----------- | ------------------------------------------------ |
| 메서드/경로 | -                                                |
| 설명        | QR 코드/시리얼 번호/등록 코드 방식으로 보드 등록 |

요청 본문

| 필드명        | 타입   | 필수   | 설명                                       |
| ------------- | ------ | ------ | ------------------------------------------ |
| register_type | String | Y      | qr / serial / code 중 하나                 |
| serial_no     | String | 조건부 | 시리얼 번호 (register_type=serial 시 필수) |
| register_code | String | 조건부 | 등록 코드 (register_type=code 시 필수)     |
| name          | String | N      | 보드 이름 (미입력 시 자동 부여)            |
| location      | String | N      | 설치 위치                                  |
| group_id      | String | N      | 그룹 ID                                    |

### 2.2.3 보드 상태 조회

| 항목          | 내용                                   |
| ------------- | -------------------------------------- |
| 메서드 / 경로 | -                                      |
| 설명          | 특정 보드의 현재 상태 및 VPN 정보 조회 |

▶ 응답 본문 - 200 OK

| 필드명           | 타입    | 설명                             |
| ---------------- | ------- | -------------------------------- |
| status           | String  | online / offline                 |
| vpn_status       | String  | connected / disconnected / error |
| vpn_ip           | String  | VPN IP 주소                      |
| last_handshake   | String  | 최근 VPN 핸드셰이크 시간         |
| uptime_seconds   | Integer | 업타임(초)                       |
| storage_total_gb | Float   | 전체 저장 용량 (GB)              |
| storage_used_gb  | Float   | 사용 저장 용량 (GB)              |
| rtsp_url         | String  | 고화질 스트림 URL                |
| rtsp_url_sub     | String  | 저화질 스트림 URL                |

### 2.2.4 보드 정보 수정

| 항목          | 내용                       |
| ------------- | -------------------------- |
| 메서드 / 경로 | -                          |
| 설명          | 보드 이름, 위치, 설명 수정 |

▶ 요청 본문

| 필드명      | 타입   | 필수 | 설명                  |
| ----------- | ------ | ---- | --------------------- |
| name        | String | N    | 보드 이름 (최대 50자) |
| location    | String | N    | 설치 위치             |
| description | String | N    | 설명 (최대 200자)     |

## 2.3 이벤트 API

### 2.3.1 이벤트 목록 조회

| 항목        | 내용                                             |
| ----------- | ------------------------------------------------ |
| 메서드/경로 | -                                                |
| 설명        | 조건에 맞는 이벤트 목록 조회 (페이지네이션 지원) |

▶ 쿼리 파라미터

| 파라미터명 | 타입    | 필수 | 설명                                 |
| ---------- | ------- | ---- | ------------------------------------ |
| page       | Integer | N    | 페이지 번호 (기본값: 1)              |
| size       | Integer | N    | 페이지 당 건수(기본값: 20)           |
| start_date | String  | N    | 조회 시작일 (YYYY-MM-DD)             |
| end_date   | String  | N    | 조회 종료일 (YYYY-MM-DD)             |
| device_id  | String  | N    | 특정 보드 필터                       |
| event_type | String  | N    | motion / sound / connection / system |
| status     | String  | N    | unconfirmed / confirmed              |

▶ 응답 본문 - 200 OK

| 필드명                 | 타입    | 설명                                    |
| ---------------------- | ------- | --------------------------------------- |
| total                  | Integer | 전체 이벤트 건수                        |
| page                   | Integer | 현재 페이지                             |
| items[].event_id       | String  | 이벤트 고유 ID                          |
| items[].device_id      | String  | 보드 ID                                 |
| items[].device_name    | String  | 보드 이름                               |
| items[].event_type     | String  | 이벤트 유형                             |
| items[].occurred_at    | String  | 발생 시간 (ISO 8601)                    |
| items[].status         | String  | unconfirmed / confirmed                 |
| items[].snapshot_url   | String  | 스냅샷 이미지 URL                       |
| items[].clip_url       | String  | 영상 클립 URL (있을 경우)               |
| items[].sound_level_db | Integer | 소리 크기 (dB, 소리 감지 이벤트에 한함) |
| items[].zone_id        | String  | 감지 영역 ID                            |

### 2.3.2 이벤트 상태 처리

| 항목        | 내용                                            |
| ----------- | ----------------------------------------------- |
| 메서드/경로 | -                                               |
| 설명        | 이벤트 확인 처리 / 중요 이벤트 지정 / 메모 저장 |

▶ 요청 본문

| 필드명       | 타입    | 필수 | 설명                  |
| ------------ | ------- | ---- | --------------------- |
| status       | String  | N    | confirmed (확인 처리) |
| is_important | Boolean | N    | 중요 이벤트 지정 여부 |
| memo         | String  | N    | 메모 (최대 500자)     |

## 2.4 저장 파일 API

### 2.4.1 저장 파일 목록 조회

| 항목        | 내용                              |
| ----------- | --------------------------------- |
| 메서드/경로 | -                                 |
| 설명        | 저장된 이미지/영상 파일 목록 조회 |

▶ 쿼리 파라미터

| 파라미터명 | 타입    | 필수 | 설명                     |
| ---------- | ------- | ---- | ------------------------ |
| device_id  | String  | N    | 특정 보드 필터           |
| start_date | String  | N    | 조회 시작일              |
| end_date   | String  | N    | 조회 종료일              |
| file_type  | String  | N    | image / video            |
| event_type | String  | N    | 이벤트 유형 필터         |
| sort       | String  | N    | created_at_desc (기본값) |
| page       | Integer | N    | 페이지 번호              |
| size       | Integer | N    | 페이지 당 건수           |

▶ 응답 본문 - 200 OK

| 필드명                  | 타입    | 설명                   |
| ----------------------- | ------- | ---------------------- |
| items[].file_id         | String  | 파일 고유 ID           |
| items[].file_name       | String  | 파일명                 |
| items[].file_type       | String  | image / video          |
| items[].device_id       | String  | 보드 ID                |
| items[].device_name     | String  | 보드 이름              |
| items[].created_at      | String  | 생성 시간              |
| items[].file_size_bytes | Integer | 파일 크기 (Bytes)      |
| items[].duration_sec    | Integer | 영상 길이 (초, 영상만) |
| items[].resolution      | String  | 해상도 (예: 1920x1080) |
| items[].event_type      | String  | 연결 이벤트 유형       |
| items[].thumbnail_url   | String  | 썸네일 URL             |
| items[].download_url    | String  | 다운로드 URL           |

### 2.4.2 파일 삭제

| 항목        | 내용                     |
| ----------- | ------------------------ |
| 메서드/경로 | -                        |
| 설명        | 파일 단건 또는 다중 삭제 |

▶ 요청 본문

| 필드명   | 타입          | 필수 | 설명                |
| -------- | ------------- | ---- | ------------------- |
| file_ids | Array<String> | Y    | 삭제할 파일 ID 목록 |

## 2.5 VPN 관리 API

### 2.5.1 VPN 연결 상태 목록 조회

| 항목        | 내용                                |
| ----------- | ----------------------------------- |
| 메서드/경로 | -                                   |
| 설명        | 등록 기기의 VPN 연결 상태 전체 조회 |

▶ 응답 본문 - 200 OK (배열)

| 필드명                      | 타입    | 설명                             |
| --------------------------- | ------- | -------------------------------- |
| device_id                   | String  | 보드 ID                          |
| device_name                 | String  | 보드 이름                        |
| serial_no                   | String  | 시리얼 번호                      |
| vpn_status                  | String  | connected / disconnected / error |
| vpn_ip                      | String  | VPN IP 주소                      |
| vpn_server                  | String  | VPN 서버 주소                    |
| connected_duration_sec      | Integer | 연결 지속 시간 (초)              |
| last_handshake              | String  | 최근 핸드셰이크 시간             |
| error_type                  | String  | 오류 유형 (없으면 null)          |
| auto_connect                | Boolean | 자동 연결 설정                   |
| reconnect_on_start          | Boolean | 시작 시 자동 연결                |
| reconnect_on_network_change | Boolean | 네트워크 변경 시 재연결          |

### 2.5.2 VPN 수동 연결/끊기

| 항목        | 내용                                |
| ----------- | ----------------------------------- |
| 메서드/경로 | -                                   |
| 설명        | 특정 기기의 VPN 수동 연결 또는 끊기 |

▶ 요청 본문

| 필드명 | 타입   | 필수 | 설명                 |
| ------ | ------ | ---- | -------------------- |
| action | String | Y    | connect / disconnect |

## 2.6 프라이버시 존 API

### 2.6.1 프라이버시 존 목록 조회

| 항목          | 내용                                |
| ------------- | ----------------------------------- |
| 메서드 / 경로 | -                                   |
| 설명          | 특정 보드의 프라이버시 존 목록 조회 |

▶ 응답 본문 - 200 OK (배열)

| 필드명         | 타입    | 설명                                   |
| -------------- | ------- | -------------------------------------- |
| zone_id        | String  | 프라이버시 존 ID (예: ZONE-01)         |
| zone_name      | String  | 존 이름 (최대 30자)                    |
| is_active      | Boolean | 활성화 여부                            |
| color          | String  | 영역 표시 색상 (Hex 코드, 예: #FF5252) |
| coordinates    | Object  | 영역 좌표 (화면 비율 기준)             |
| coordinates.x1 | Float   | 좌상단 X (0.0~1.0)                     |
| coordinates.y1 | Float   | 좌상단 Y (0.0~1.0)                     |
| coordinates.x2 | Float   | 우하단 X (0.0~1.0)                     |
| coordinates.y2 | Float   | 우하단 Y (0.0~1.0)                     |
| created_by     | String  | 생성자 ID                              |
| created_at     | String  | 생성 시간 (ISO 8601)                   |
| updated_at     | String  | 최근 수정 시간 (ISO 8601)              |

### 2.6.2 프라이버시 존 추가

| 항목          | 내용                                          |
| ------------- | --------------------------------------------- |
| 메서드 / 경로 | -                                             |
| 설명          | 프라이버시 존 신규 추가(보드 1개당 최대 20개) |

▶ 요청 본문

| 필드명         | 타입    | 필수 | 설명                            |
| -------------- | ------- | ---- | ------------------------------- |
| zone_name      | String  | Y    | 존 이름 (최대 30자)             |
| is_active      | Boolean | N    | 활성화 여부 (기본값: true)      |
| color          | String  | N    | 표시 색상 Hex (기본값: #FF5252) |
| coordinates.x1 | Float   | Y    | 좌상단 X 비율 (0.0~1.0)         |
| coordinates.y1 | Float   | Y    | 좌상단 Y 비율 (0.0~1.0)         |
| coordinates.x2 | Float   | Y    | 우하단 X 비율 (0.0~1.0)         |
| coordinates.y2 | Float   | Y    | 우하단 Y 비율 (0.0~1.0)         |

### 2.6.3 프라이버시 존 수정/삭제

| 항목             | 내용 |
| ---------------- | ---- |
| 수정 메서드/경로 | -    |
| 삭제 메서드/경로 | -    |
| 다중 삭제        | -    |

## 2.7 알림 설정 API

### 2.7.1 알림 설정 조회

| 항목        | 내용                       |
| ----------- | -------------------------- |
| 메서드/경로 | -                          |
| 설명        | 사용자 알림 설정 전체 조회 |

▶ 응답 본문 - 200 OK

| 필드명                    | 타입    | 설명                    |
| ------------------------- | ------- | ----------------------- |
| motion_alert              | Boolean | 움직임 감지 알림        |
| sound_alert               | Boolean | 소리 감지 알림          |
| offline_alert             | Boolean | 보드 오프라인 알림      |
| reconnect_alert           | Boolean | 보드 재연결 알림        |
| important_only            | Boolean | 중요 이벤트만 알림      |
| device_alerts[]           | Array   | 보드별 알림 활성화 목록 |
| device_alerts[].device_id | String  | 보드 ID                 |
| device_alerts[].enabled   | Boolean | 알림 활성화 여부        |

### 2.7.2 알림 설정 수정

| 항목        | 내용                |
| ----------- | ------------------- |
| 메서드/경로 | -                   |
| 설명        | 알림 설정 전체 저장 |

# 3. ICD-02/03: 웹캠 보드 ↔ 클라우드 서버

베이스 URL: [-](-) (VPN 내부망)
보드에서 서버로 전송하는 내부 API. VPN 터널 내부 HTTP 사용.

## 3.1 보드 프로비저닝

### 3.1.1 프로비저닝 요청

| 항목          | 내용                                         |
| ------------- | -------------------------------------------- |
| 메서드 / 경로 | -                                            |
| 설명          | 보드 최초 부팅 시 서버에 VPN 키·IP 발급 요청 |
| 호출 주체     | 웹캠 보드                                    |

▶ 요청 본문

| 필드명           | 타입   | 필수 | 설명                    |
| ---------------- | ------ | ---- | ----------------------- |
| serial_no        | String | Y    | 보드 시리얼 번호        |
| wg_pubkey        | String | Y    | 보드의 WireGuard 공개키 |
| firmware_version | String | Y    | 현재 펌웨어 버전        |
| model            | String | Y    | 보드 모델명             |

▶ 응답 본문 - 200 OK

| 필드명          | 타입   | 설명                    |
| --------------- | ------ | ----------------------- |
| device_id       | String | 서버에서 부여한 보드 ID |
| vpn_ip          | String | 할당된 VPN IP 주소      |
| server_pubkey   | String | 서버의 WireGuard 공개키 |
| server_endpoint | String | VPN 서버 주소: 포트     |
| provision_token | String | heartbeat 인증용 토큰   |
| rtsp_proxy_ip   | String | mediamtx RTSP 프록시 IP |

## 3.2 Heartbeat (상태 보고)

### 3.2.1 Heartbeat 전송

| 항목          | 내용                                     |
| ------------- | ---------------------------------------- |
| 메서드 / 경로 | -                                        |
| 설명          | 보드에서 30초 주기로 생존 및 상태 보고   |
| 호출 주체     | 웹캠 보드                                |
| 인증          | provision_token 필드로 인증 (JWT 불필요) |

▶ 요청 본문

| 필드명             | 타입    | 필수 | 설명                           |
| ------------------ | ------- | ---- | ------------------------------ |
| serial_no          | String  | Y    | 보드 시리얼 번호               |
| provision_token    | String  | Y    | 프로비저닝 시 발급된 인증 토큰 |
| status             | String  | Y    | online 고정                    |
| rtsp_ok            | Boolean | Y    | RTSP 스트리밍 정상 여부        |
| wg_ok              | Boolean | Y    | WireGuard 터널 정상 여부       |
| storage_used_bytes | Integer | N    | 사용 저장 용량 (Bytes)         |
| firmware_version   | String  | N    | 현재 펌웨어 버전               |
| latency_ms         | Integer | N    | 서버 응답 지연 시간 (ms)       |

▶ 응답

| HTTP 코드      | 설명                            |
| -------------- | ------------------------------- |
| 204 No Content | 정상 수신                       |
| 403 Forbidden  | 토큰 불일치 - 재프로비저닝 필요 |

## 3.3 이벤트 전송

### 3.3.1 이벤트 보고

| 항목          | 내용                                 |
| ------------- | ------------------------------------ |
| 메서드 / 경로 | -                                    |
| 설명          | 보드에서 감지한 이벤트를 서버에 전송 |
| 호출 주체     | 웹캠 보드                            |

▶ 요청 본문

| 필드명          | 타입    | 필수 | 설명                                        |
| --------------- | ------- | ---- | ------------------------------------------- |
| serial_no       | String  | Y    | 보드 시리얼 번호                            |
| provision_token | String  | Y    | 인증 토큰                                   |
| event_type      | String  | Y    | motion / sound / connection / system        |
| occurred_at     | String  | Y    | 이벤트 발생 시간 (ISO 8601)                 |
| zone_id         | String  | N    | 감지 영역 ID                                |
| sound_level_db  | Integer | N    | 소리 크기 (dB, 소리 이벤트 시)              |
| snapshot_data   | String  | N    | 스냅샷 이미지 Base64 (또는 별도 업로드 URL) |
| detail          | String  | N    | 추가 설명                                   |

## 3.4 PTZ 제어 명령

### 3.4.1 PTZ 명령 수신

| 항목          | 내용                                                   |
| ------------- | ------------------------------------------------------ |
| 메서드 / 경로 | -                                                      |
| 설명          | 서버를 통해 보드로 PTZ 제어 명령 전달 (서버→보드 중계) |
| 호출 주체     | 클라우드 서버 (앱/웹 요청 중계)                        |

▶ 요청 본문

| 필드명     | 타입    | 필수 | 설명                                                        |
| ---------- | ------- | ---- | ----------------------------------------------------------- |
| action     | String  | Y    | up / down / left / right / home / zoom_in / zoom_out / stop |
| speed      | Integer | N    | 이동 속도 1~10 (기본값: 5)                                  |
| zoom_scale | Float   | N    | 줌 배율 (1.0~10.0, zoom_in/out 시)                          |

# 4. ICD-04/05 : RTSP 영상 스트리밍

## 4.1 스트리밍 구성

▶ 보드 → mediamtx (ICD-04): 보드에서 직접 스트림 전송
mediamtx → 앱/웹 (ICD-05): 클라이언트에 스트림 제공

| 항목                   | 규격                            |
| ---------------------- | ------------------------------- |
| 프로토콜               | RTSP over TCP                   |
| 포트                   | -                               |
| 코덱                   | H.265 (기본), H.264 (호환 모드) |
| 오디오                 | G.711                           |
| 고화질 스트림 URL 형식 | -                               |
| 저화질 스트림 URL 형식 | -                               |
| 고화질 해상도          | 1920×1080 @ 30fps               |
| 저화질 해상도          | 704x576 @ 15fps                 |
| 전송 방식              | VPN 터널 내부 RTSP              |

## 4.2 스트림 경로 예시

보드 시리얼: -, VPN IP: -
고화질: -
저화질:-

# 5. ICD-06: WireGuard VPN 인터페이스

## 5.1 VPN 구성 규격

| 항목                | 규격              |
| ------------------- | ----------------- |
| VPN 프로토콜        | WireGuard         |
| 전송 프로토콜       | UDP               |
| VPN 서버 포트       | -                 |
| VPN 서버 주소       | -                 |
| IP 대역             | -                 |
| 서버 VPN IP         | -                 |
| 보드 IP 풀          | - (자동 할당)     |
| 폰/클라이언트 IP 풀 | - (자동 할당)     |
| 허용 IP 범위        | -                 |
| Keepalive 주기      | 25초              |
| MTU                 | 1420              |
| 암호화 알고리즘     | -(WireGuard 기본) |

## 5.2 보드 WireGuard 설정 파일 형식

▶ 보드 /etc/wireguard/wg0.conf 형식

```ini
[Interface]
PrivateKey = {보드 개인키}
Address = {할당 VPN IP}/24

[Peer]
PublicKey = {서버 공개키}
Endpoint =-
AllowedIPs = -
PersistentKeepalive = 25

```

## 5.3 VPN 연결 상태 코드

| 상태 코드               | 설명                  | 조치              |
| ----------------------- | --------------------- | ----------------- |
| connected               | VPN 터널 정상 연결    |                   |
| disconnected            | VPN 미연결 상태       | 자동 재연결 시도  |
| error_handshake_timeout | 핸드셰이크 타임아웃   | 재연결 시도       |
| error_auth_failed       | 인증 실패 (키 불일치) | 재프로비저닝 필요 |
| error_no_route          | 라우팅 경로 없음      | 네트워크 확인     |

# 6. ICD-07: Push 알림 (서버 → 앱/웹)

## 6.1 알림 전송 규격

| 항목           | 규격                           |
| -------------- | ------------------------------ |
| 전송 방식      | FCM (Firebase Cloud Messaging) |
| 최대 지연 시간 | 5초 이내                       |
| 알림 우선순위  | HIGH (이벤트 알림)             |

## 6.2 알림 메시지 형식

▶ 이벤트 알림 Payload

| 필드명             | 타입   | 설명                                   |
| ------------------ | ------ | -------------------------------------- |
| notification.title | String | 알림 제목 (예: "CAM-0001 움직임 감지") |
| notification.body  | String | 알림 내용                              |
| data.event_id      | String | 이벤트 ID                              |
| data.device_id     | String | 보드 ID                                |
| data.event_type    | String | motion / sound / connection / system   |
| data.occurred_at   | String | 발생 시간 (ISO 8601)                   |
| data.snapshot_url  | String | 스냅샷 URL                             |
| data.action        | String | open_event (앱 내 이동 액션)           |

## 6.3 알림 유형별 메시지

| 알림 유형     | 제목 형식            | 내용 형식                                       |
| ------------- | -------------------- | ----------------------------------------------- |
| 움직임 감지   | {보드명} 움직임 감지 | {설치 위치}에서 움직임이 감지되었습니다.        |
| 소리 탐지     | {보드명} 소리 탐지   | {설치 위치}에서 소리가 감지되었습니다. ({dB}dB) |
| 보드 오프라인 | {보드명} 연결 끊김   | {보드명}이 오프라인 상태가 되었습니다.          |
| 보드 재연결   | {보드명} 재연결      | {보드명}이 다시 연결되었습니다.                 |
| VPN 오류      | {보드명} VPN 오류    | VPN 연결 오류가 발생했습니다. ({오류 유형})     |

# 7. ICD-08: 클라우드 서버 ↔ DB

## 7.1 주요 테이블 정의

### 7.1.1 users (사용자)

| 컬럼명        | 타입         | 제약             | 설명                       |
| ------------- | ------------ | ---------------- | -------------------------- |
| user_id       | UUID         | PK               | 사용자 고유 ID             |
| email         | VARCHAR(255) | UNIQUE, NOT NULL | 이메일                     |
| password_hash | VARCHAR(255) | NOT NULL         | bcrypt 해시 비밀번호       |
| name          | VARCHAR(100) | NOT NULL         | 사용자 이름                |
| plan          | ENUM         | NOT NULL         | basic / standard / premium |
| created_at    | TIMESTAMP    | NOT NULL         | 가입 시간                  |
| updated_at    | TIMESTAMP    | NOT NULL         | 최근 수정 시간             |

### 7.1.2 devices (보드)

| 컬럼명           | 타입         | 제약             | 설명                   |
| ---------------- | ------------ | ---------------- | ---------------------- |
| device_id        | VARCHAR(50)  | PK               | 보드 ID (예: CAM-0001) |
| user_id          | UUID         | FK → users       | 소유자 ID              |
| serial_no        | VARCHAR(100) | UNIQUE, NOT NULL | 시리얼 번호            |
| name             | VARCHAR(100) | NOT NULL         | 보드 이름              |
| location         | VARCHAR(200) | NOT NULL         | 설치 위치              |
| model            | VARCHAR(100) |                  | 모델명                 |
| firmware_version | VARCHAR(50)  |                  | 펌웨어 버전            |
| wg_pubkey        | VARCHAR(255) |                  | WireGuard 공개키       |
| vpn_ip           | VARCHAR(20)  |                  | 할당된 VPN IP          |
| provision_token  | VARCHAR(255) |                  | heartbeat 인증 토큰    |
| is_active        | BOOLEAN      | DEFAULT true     | 활성화 여부            |
| registered_at    | TIMESTAMP    |                  | 등록 완료 시간         |
| created_at       | TIMESTAMP    | NOT NULL         | 생성 시간              |

### 7.1.3 events (이벤트)

| 컬럼명         | 타입         | 제약                | 설명                                 |
| -------------- | ------------ | ------------------- | ------------------------------------ |
| event_id       | UUID         | PK                  | 이벤트 고유 ID                       |
| device_id      | VARCHAR(50)  | FK → devices        | 보드 ID                              |
| event_type     | ENUM         | NOT NULL            | motion / sound / connection / system |
| occurred_at    | TIMESTAMP    | NOT NULL            | 이벤트 발생 시간                     |
| status         | ENUM         | DEFAULT unconfirmed | unconfirmed / confirmed              |
| is_important   | BOOLEAN      | DEFAULT false       | 중요 이벤트 여부                     |
| zone_id        | VARCHAR(50)  |                     | 감지 영역 ID                         |
| sound_level_db | INTEGER      |                     | 소리 크기 (dB)                       |
| snapshot_url   | VARCHAR(500) |                     | 스냅샷 URL                           |
| clip_url       | VARCHAR(500) |                     | 영상 클립 URL                        |
| memo           | VARCHAR(500) |                     | 메모                                 |
| created_at     | TIMESTAMP    | NOT NULL            | 레코드 생성 시간                     |

### 7.1.4 phones (폰/클라이언트)

| 컬럼명        | 타입         | 제약         | 설명               |
| ------------- | ------------ | ------------ | ------------------ |
| phone_id      | UUID         | PK           | 클라이언트 고유 ID |
| user_id       | UUID         | FK → users   | 사용자 ID          |
| device_name   | VARCHAR(100) | NOT NULL     | 기기 이름 (모델명) |
| wg_pubkey     | VARCHAR(255) | UNIQUE       | WireGuard 공개키   |
| vpn_ip        | VARCHAR(20)  |              | 할당된 VPN IP      |
| is_active     | BOOLEAN      | DEFAULT true | 활성화 여부        |
| registered_at | TIMESTAMP    |              | 등록 시간          |

### 7.1.5 privacy_zones (프라이버시 존)

| 컬럼명     | 타입        | 제약         | 설명                |
| ---------- | ----------- | ------------ | ------------------- |
| zone_id    | VARCHAR(50) | PK           | 존 ID (예: ZONE-01) |
| device_id  | VARCHAR(50) | FK → devices | 보드 ID             |
| zone_name  | VARCHAR(30) | NOT NULL     | 존 이름             |
| is_active  | BOOLEAN     | DEFAULT true | 활성화 여부         |
| color      | VARCHAR(10) | NOT NULL     | 표시 색상 (Hex)     |
| coord_x1   | FLOAT       | NOT NULL     | 좌상단 X 비율       |
| coord_y1   | FLOAT       | NOT NULL     | 좌상단 Y 비율       |
| coord_x2   | FLOAT       | NOT NULL     | 우하단 X 비율       |
| coord_y2   | FLOAT       | NOT NULL     | 우하단 Y 비율       |
| created_by | UUID        | FK → users   | 생성자 ID           |
| created_at | TIMESTAMP   | NOT NULL     | 생성 시간           |
| updated_at | TIMESTAMP   | NOT NULL     | 수정 시간           |

# 8. 공통 오류 코드

| HTTP 코드 | 오류 코드             | 설명                                 | 조치                  |
| --------- | --------------------- | ------------------------------------ | --------------------- |
| 400       | BAD_REQUEST           | 요청 파라미터 오류                   | 요청 형식 확인        |
| 401       | UNAUTHORIZED          | 인증 토큰 없음 또는 만료             | 재로그인 후 토큰 갱신 |
| 403       | FORBIDDEN             | 접근 권한 없음                       | 계정 권한 확인        |
| 404       | NOT_FOUND             | 리소스를 찾을 수 없음                | ID 값 확인            |
| 409       | CONFLICT              | 중복 데이터 (이메일, 시리얼 번호 등) | 데이터 중복 확인      |
| 429       | TOO_MANY_REQUESTS     | 요청 횟수 초과                       | 잠시 후 재시도        |
| 500       | INTERNAL_SERVER_ERROR | 서버 내부 오류                       | 서버 로그 확인        |
| 503       | SERVICE_UNAVAILABLE   | 서버 점검 중                         | 공지사항 확인         |

# 9. 변경 이력

| 버전 | 변경일     | 변경 내용 | 작성자     |
| ---- | ---------- | --------- | ---------- |
| v1.0 | 2026-08-03 | 최초 작성 | (주)스마클 |
