# Smart Safety System

---

# 1. 주제

**스마트 세이프티 시스템 (Smart Safety System)**
건설 현장 작업자 IoT 안전 모니터링 플랫폼
STM32 / Raspberry Pi 5 / Qt Dashboard / Bluetooth SPP / TCP / ML

---

# 2. 목표

현장에서 안전모 미착용·근무 중 음주·과중 작업 지시 등이 실제로 빈번하게 발생한다. 이를 방지하기 위해 스마트 세이프티 시스템을 개발했다.

| 감지 항목 | 방법 |
| --- | --- |
| 음주 여부 | MQ-3 알코올 센서 → gas_do 플래그 → Qt 경보 |
| 심박수 이상 | HW-827 맥박 → ADC → BPM 피크감지 알고리즘 |
| 헬멧 착용 여부 | IR Tracker → 착용 시각 기록 → 근로시간 자동 누적 |
| 행동 / 낙상 | MPU-6050 6축 IMU → ML 행동인식 + 낙상 감지 |
| 영상 증거 | GStreamer → 사고 전후 10초 MP4 자동 저장 |

기업 ↔ 작업자 양방향 대응력 강화 : Qt 대시보드로 관리자가 실시간 모니터링, STM32 LCD로 작업자에게 상태 역전송

---

# 3. 시스템 구성도
<p align="center">
  <img src="https://github.com/user-attachments/assets/19da634f-f89a-4c02-896a-38c978bcfcbf" width="700"/>
</p>


```markdown
[STM32 BAND]
  MPU-6050  가속도+자이로 6축
  BMP-280   기압 (층간 이동 감지)
  MQ-3      알코올
  HW-827    맥박 ADC
  LCD 1602  BPM / 착용 / 행동 표시
        |
        | Bluetooth SPP (HC-06 / rfcomm0 / 9600bps)
        |
[Helmet RPi5]  ← 클라이언트
  BH-1750   조도
  IR Tracker  헬멧 착용 감지
  웹캠      GStreamer JPEG 스트림
  ML 모델   imu_motion_model2.pkl
  SQLite DB helmet_sensor_data.db
        |
        | WiFi TCP  9090 (센서) / 9091 (영상)
        |
[Server RPi5]  ← 서버
  C++17 멀티스레드 서버
  DeviceState  헬멧별 독립 상태 관리
  GStreamer    MP4 저장
  JSONL 로그  헬멧별 파일 분리
        |
        | WiFi TCP  9092 (영상) / 9093 (센서) / 9094 (SWITCH 명령)
        |
[Qt Dashboard]  ← 관리자 UI
  실시간 센서 수신
  영상 스트리밍 표시
  헬멧 전환 (SWITCH:helmet-02)
  사고 / 낙상 / 음주 알림 카드
```

# 3-1 센서정리

## 🔧 센서 정리

<p align="center">
  <img src="https://github.com/user-attachments/assets/7011687d-19de-4f5d-8152-943ebb6f1224" width="600"/>
</p>
<p align="center">
  <img src="https://github.com/user-attachments/assets/78c32710-4c1c-4c6e-be13-65b6c8e64cdb" width="600"/>
</p>

# 3-2 하드웨어 사진
<p align="center">
  <img src="https://github.com/user-attachments/assets/e53dd218-75bb-4a08-ac5d-aa78a961e0f3" width="450"/>
</p>

---

# 4. 기술 정리

## 하드웨어

| 부품 | 역할 |
| --- | --- |
| STM32 | 센서 허브 (MPU-6050, BMP-280, MQ-3, HW-827, LCD) |
| MPU-6050 | 가속도+자이로 6축 → ML 행동인식 / accel_mag / accel_delta 계산 |
| BMP-280 | 기압 → 층간 이동 감지 보조 |
| MQ-3 | 알코올 디지털 출력 |
| HW-827 | 맥박 ADC 원시값 → BPM 피크감지 변환 |
| IR Tracker | 헬멧 착용 유무 판단 → 근로시간 누적 |
| BH-1750 | 조도(lux) 측정 → 환경 분류 (야간 / 실내 / 야외) |
| 웹캠 | GStreamer 영상 스트리밍 → 사고 영상 저장 |
| LCD 1602 | BPM / 착용상태 / 행동 역전송 표시 |
| Raspberry Pi 5 | 클라이언트(헬멧) 1대 + 서버 1대 |

## 소프트웨어

| 항목 | 내용 |
| --- | --- |
| 클라이언트 | Python 3 |
| 서버 | C++17 + json + GStreamer |
| 데이터베이스 | SQLite3 (WAL 모드 / 10패킷 배치 commit) |
| ML 모델 | scikit-learn Random Forest — 정확도 97.5% |
| 영상 스트리밍 | GStreamer + libcamerasrc → JPEG → TCP |
| ML 학습 방식 | UCI WISDM 공개 데이터 선학습 → 실측 데이터 재학습 |
| Qt Dashboard | 실시간 센서 / 영상 / 경보 표시 |

## 통신

| 구간 | 방식 | 세부 |
| --- | --- | --- |
| STM32 ↔ RPi5 | Bluetooth SPP | HC-06 / rfcomm0 / 9600bps |
| Helmet RPi5 → Server | WiFi TCP | 9090 센서 JSON / 9091 영상 |
| Server → Qt | WiFi TCP | 9092 영상 / 9093 센서 / 9094 SWITCH 명령 |

---

# 5. 주요 기능

## TCP 통신 구조

- 포트 9090 : 헬멧 → 서버, JSON 센서 데이터 1줄씩 전송 (\n 구분)
- 포트 9091 : 헬멧 → 서버, 8byte 길이 헤더 + JPEG 바이너리
- 포트 9092 : 서버 → Qt, 영상 브로드캐스트
- 포트 9093 : 서버 → Qt, 센서 JSON 브로드캐스트
- 포트 9094 : Qt → 서버, SWITCH:helmet-02\n 명령 (영상 전환)
- 연결 직후 DEVID:helmet-01\n 전송 → 다중 헬멧 device_id 매핑

## SPP (Bluetooth Serial Port Profile)

- STM32가 HC-06을 통해 UART로 센서 패킷 전송
- 패킷 포맷 : [DATA] ax,ay,az,gx,gy,gz,pressure,adc,gas_do
- RPi5에서 /dev/rfcomm0 로 수신 → parse_line() 정규식 파싱
- 역방향 : RPi5 → STM32 LCD로 [BPM] / [HLM] / [ACT] 문자열 전송

## 머신러닝 (행동인식)

- 모델 : Random Forest Classifier (scikit-learn)
- 입력 : IMU 6축 데이터 (ax, ay, az, gx, gy, gz) × 50샘플 윈도우
- 피처 : 각 축별 평균 / 표준편차 / 최대 / 최소 / 범위 + 가속도 합성크기 통계 (총 33개)
- 분류 결과 : 걷기 / 뛰기 / 앉음_정지 (확률 < 0.6 → 알수없음)
- 추론 구조 : ml_worker 별도 스레드 + Queue(maxsize=1) → 메인루프 논블로킹
- 낙상 감지 : 기압 감소 0.2hPa 이상 + 가속도 급변 5.0m/s² 이상 동시 발생

## 데이터 저장 처리

SQLite3 배치 commit + WAL 모드로 성능 최적화. 아래 섹션 참고.

---

# 6. 데이터 저장 — 상세

## DB 구조 (helmet_sensor_data.db)

```markdown
CREATE TABLE sensor_log (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp      DATETIME DEFAULT (datetime('now','localtime')),

    -- IMU (STM32 → BT → RPi)
    ax REAL, ay REAL, az REAL,    -- 가속도 m/s²  (원시값 / 10000)
    gx REAL, gy REAL, gz REAL,    -- 자이로 rad/s (원시값 / 10000)

    -- 환경
    pressure       REAL,          -- 기압 hPa (원시값 / 100 or / 10000 자동 판별)
    lux            REAL,          -- 조도 (BH-1750, 0.2초 캐시)

    -- 착용 / 근로
    tracker_state  INTEGER,       -- IR센서 원시값 (0/1)
    helmet_wearing INTEGER,       -- 착용 여부 (0/1)
    work_seconds   INTEGER,       -- 누적 착용 근로시간 (초)

    -- 맥박
    adc_avg        INTEGER,       -- HW-827 ADC 원시값
    bpm            INTEGER,       -- 피크감지 계산 결과

    -- 가속도 파생값
    accel_mag      REAL,          -- 합성크기 sqrt(ax²+ay²+az²)
    accel_delta    REAL,          -- 이전 프레임 대비 변화량

    -- 알코올
    gas_do         INTEGER,       -- MQ-3 출력 (0=정상 / 1=감지)

    -- ML 결과
    activity_ml    TEXT,          -- 걷기 / 뛰기 / 앉음_정지 / 알수없음

    -- 사고
    fall_detected  INTEGER        -- 낙상 감지 (0/1)
);
```

## 성능 최적화

| 항목 | 내용 |
| --- | --- |
| WAL 모드 | PRAGMA journal_mode=WAL → 읽기/쓰기 동시 접근 허용 |
| 동기화 수준 | PRAGMA synchronous=NORMAL → fsync 부하 감소 |
| 배치 commit | COMMIT_EVERY=10 → 10패킷마다 1회 commit |
| 조도 캐시 | BH-1750 0.2초 캐시 → I2C 블로킹 방지 |

## 서버 로그 (JSONL)

- 경로 : /home/pi/smart-band/sensor_logs/<device_id>.jsonl
- 헬멧별 파일 분리 (helmet-01.jsonl / helmet-02.jsonl)
- 각 줄 : { ts_server, client_addr, device_id, activity, environment, alert, payload }

## 사고 영상 저장

- 경로 : /home/pi/smart-band/accidents/accident_<device_id>_<timestamp>.mp4
- 조건 : 가속도 급변 > 15m/s² OR BPM 이상 OR 낙상 감지
- 내용 : 사고 직전 10초 버퍼 + 사고 후 10초 → GStreamer mp4mux 합성
- 저장 완료 후 Qt로 ACCIDENT_VIDEO_SAVED JSON 전송

---

# 7. 통신

## Bluetooth SPP (STM32 → RPi5)

패킷 형식

```markdown
[DATA] ax,ay,az,gx,gy,gz,pressure,adc,gas_do

파싱 정규식
\[DATA\]\s*(-?\d+),(-?\d+),(-?\d+),(-?\d+),(-?\d+),(-?\d+),(-?\d+),(-?\d+),(\d+)

변환
  ax = int(group(1)) / 10000.0
  pressure = int(group(7)) / 100.0  → 800~1100 hPa 범위 검증
  gas_do = int(group(9))            → 0 또는 1만 유효
```

역방향 (RPi5 → STM32 LCD)

```markdown
[BPM]075\r\n   심박수
[HLM]1\r\n     착용=1 / 미착용=0
[ACT]WALK\r\n  행동 (WALK / RUN / SIT / ------)

주기 : 1초마다 DB 최신값 조회 → 변경된 항목만 전송
```

## WiFi TCP — 센서 채널 (포트 9090)

```markdown
방향 : 헬멧 RPi → 서버
형식 : JSON 한 줄 + \n

{
  "seq": 1234,
  "device_id": "helmet-01",
  "data": {
    "ax":0.1, "ay":-0.9, "az":9.8,
    "gx":0.01, "gy":0.0, "gz":0.02,
    "pressure":1013.25, "adc_avg":2048,
    "bpm":72, "gas_do":0
  },
  "lux": 320.5,
  "tracker": 1,
  "helmet_on": true,
  "work_min": 45,
  "accel_mag": 9.85,
  "accel_delta": 0.12,
  "activity": "걷기",
  "fall_detected": false,
  "ts_client": "2025-03-08T14:32:01"
}

전송 실패 시 즉시 소켓 재연결 (무한 루프)
```

## WiFi TCP — 영상 채널 (포트 9091)

```markdown
방향 : 헬멧 RPi → 서버
연결 직후 : "DEVID:helmet-01\n" 전송 (device_id 식별)
프레임 형식 : [8byte 크기 헤더 little-endian uint64] + [JPEG 바이너리]
해상도 : 320x240 / FPS : 15 / JPEG quality : 60

GStreamer 파이프라인
  libcamerasrc
  → videoconvert
  → videoscale
  → video/x-raw,width=320,height=240,framerate=15/1
  → jpegenc quality=60
  → appsink
```

## Qt 채널 (포트 9092~9094)

```markdown
9092 : 서버 → Qt  영상 브로드캐스트 (8byte + JPEG)
9093 : 서버 → Qt  센서 JSON 브로드캐스트 (activity / environment / alert 포함)
9094 : Qt → 서버  명령 수신
       "SWITCH:helmet-02\n" → 서버 활성 영상 헬멧 전환
```

## 다중 헬멧 지원 (C++ DeviceState)

```markdown
g_dev_map : map<device_id, DeviceState*>

DeviceState 당 독립 보유
  FrameQ      프레임 큐 (최대 2개, 초과 시 drop)
  frame_buf   사고 전 10초 버퍼 (FPS × 10 프레임)
  acc_active  사고 진행 중 플래그 (atomic<bool>)
  prev_mtx    prev_pressure / prev_bpm race condition 방지

스레드 구조
  sensor_in_server()   헬멧별 handle_sensor_client() 스레드
  video_in_server()    헬멧별 handle_video_client() + frame_processor_thread()
  qt_video_server()    Qt 영상 브로드캐스터 등록
  qt_sensor_server()   Qt 센서 브로드캐스터 등록
  qt_cmd_server()      Qt SWITCH 명령 수신 (포트 9094)
```
