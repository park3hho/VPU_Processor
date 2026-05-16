# Retinal VPU — Full Roadmap (6 Stages)

KV260 망막 VPU 프로젝트의 전체 흐름. PyTorch 검증부터 로봇 RL까지.

---

## 큰 그림

```
[현재 위치 ★]
  ├─ Phase A: HLS 첫 IP 완료 (pass-through)
  └─ Phase B: KV260에 자체 IP 통합 + 동작 검증 완료

  ↓ 본 연구 진입 ↓

Stage 1: 22채널 VPU 완성 (Phase C+D)
Stage 2: 카메라 실시간 연동
Stage 3: 22채널 출력 검증 (디버깅)
Stage 4: 스트림 처리 검증
Stage 5: 칩 (FPGA 그대로 or ASIC)
Stage 6: 로봇 탑재 + RL
```

---

## Stage 1: 22채널 VPU 완성

### 목표
PyTorch 검증된 망막 알고리즘을 HLS로 포팅. KV260에 합성.

### 작업 (Phase C + D)

#### Phase C — Spatial 10ch
```
Channel 0: Luminance bandpass (Y - avg_pool(Y, 5×5))
Channel 1: R-G opponent     (V from YUV422)
Channel 2: B-Y opponent     (U from YUV422)
Channel 3: Edge magnitude   (|gx| + |gy|, Sobel L1)
Channel 4: Edge orientation X (gx/mag)
Channel 5: Edge orientation Y (gy/mag)
Channel 6: Gabor 0°
Channel 7: Gabor 45°
Channel 8: Gabor 90°
Channel 9: Gabor 135°
```

#### Phase D — Temporal + Motion 12ch
```
Channel 10: dL/dt (시간 미분)
Channel 11: ON transient (ReLU(dL/dt))
Channel 12: OFF transient (ReLU(-dL/dt))
Channel 13: Sustained low-pass

Channel 14: DS_right (prev[x-1] × curr[x])
Channel 15: DS_left
Channel 16: DS_up
Channel 17: DS_down

Channel 18: Contrast normalized lum
Channel 19: Local std

Channel 20: S-OFF (color extension)
Channel 21: Konio transient
```

### 검증
- HLS C Simulation에서 PyTorch 결과와 cross-validation
- 정확도 99%+ 일치 (zero or ε difference)

### 산출물
- `vpu_retinal_full.zip` (Vivado IP)
- 자원: ~20~30K LUT, ~수십 DSP, ~수십 BRAM

---

## Stage 2: 카메라 실시간 연동

### 목표
KV260 IAS 카메라 (AR1335 + AP1302) 실제 동작.

### 도전 과제
이전에 PPA private 문제로 막힌 부분:
- Linux device tree overlay (.dtbo) 없음
- ap1302 driver 자동 로드 X
- `/dev/video*` 안 생김

### 해결 방법 (옵션)

#### 방법 A: PYNQ에서 PS의 I2C 직접 제어 (추천)
```python
# /dev/i2c-1 통해 AP1302 직접 init
# Linux driver 우회
# AP1302 firmware는 GitHub 또는 AMD에서 별도 다운로드
```

#### 방법 B: Device Tree Overlay 작성
```
.dts 파일 작성 (수백 줄)
dtc로 .dtbo 컴파일
sudo xmutil loadapp ...
```

#### 방법 C: USB 카메라 (임시 우회)
```python
# OpenCV로 USB cam capture
# PYNQ로 DDR에 numpy 직접 주입
# VPU 통과 → DDR에서 결과 읽기
# KV260 IAS는 못 쓰지만 알고리즘 검증 가능
```

### 산출물
- 카메라 → PL의 MIPI Rx → VPU → DDR 풀 파이프라인 동작

---

## Stage 3: 22채널 출력 검증 (★ 디버깅 핵심)

### 목표
실제 영상에서 각 채널이 의도대로 동작하는지 시각화 + 분석.

### 검증 코드
```python
frame_rgb = camera.capture()
output_22ch = vpu_hardware(frame_rgb)  # KV260의 PL

# 22채널 시각화
fig, axes = plt.subplots(4, 6, figsize=(20, 14))
channel_names = [
    'Y bandpass', 'R-G', 'B-Y', 'edge_mag',
    'edge_x', 'edge_y', 'gabor_0', 'gabor_45',
    'gabor_90', 'gabor_135', 'dL/dt', 'ON',
    'OFF', 'sustained', 'DS_right', 'DS_left',
    'DS_up', 'DS_down', 'contrast', 'local_std',
    'S-OFF', 'konio_trans'
]
for i in range(22):
    axes[i//6, i%6].imshow(output_22ch[i])
    axes[i//6, i%6].set_title(channel_names[i])
plt.savefig('vpu_output_visualization.png')
```

### 검증 포인트

| 채널 | 기대 동작 |
|------|----------|
| Y bandpass | 윤곽선 강조, 평탄한 영역 어두움 |
| R-G | 빨간 물체 = 양수, 녹색 = 음수 |
| edge_mag | 명확한 경계만 밝게 |
| Gabor 0° | 수평 edge만 강조 |
| dL/dt | 정지 영상은 검정, 움직이면 활성 |
| DS_right | 오른쪽 이동 물체만 강조 |

### PyTorch와 cross-validation
```python
pytorch_output = retinal_vpu_pytorch(frame_rgb)
hardware_output = vpu_kv260(frame_rgb)

diff = np.abs(pytorch_output - hardware_output)
assert diff.max() < 1.0, "VPU 출력 불일치"
```

### 산출물
- 22채널 시각화 결과
- PyTorch와 hardware 정확도 비교 표
- 채널별 동작 분석 보고서

---

## Stage 4: 스트림 처리 검증

### 목표
연속 비디오에서 성능 측정.

### 지표

| 항목 | 목표 |
|------|------|
| Throughput | 1080p @ 60fps |
| Latency | 카메라 → VPU 출력 < 16ms (1프레임) |
| Power | KV260 전체 < 15W |
| Stability | 1시간 연속 동작 |

### 측정 방법
```python
import time

# Throughput 측정
start = time.time()
for _ in range(600):  # 10초 분량 @ 60fps
    rgb = camera.capture()
    out = vpu(rgb)
elapsed = time.time() - start
fps = 600 / elapsed

# Latency 측정
t0 = time.time()
rgb = camera.capture()
out = vpu(rgb)
latency_ms = (time.time() - t0) * 1000
```

### 시간 채널 검증 특히 중요
- 연속 프레임 100개 입력
- DS_right가 실제 움직임 방향과 일치하는지
- ON/OFF transient가 밝기 변화와 매칭되는지

### 산출물
- 성능 측정 보고서
- 비디오 데모 영상 (입력 + 22채널 출력 동시 표시)

---

## Stage 5: 칩 (FPGA 그대로 or ASIC)

### 옵션 A: KV260 그대로 (현실적, 학기 프로젝트)

```
장점:
- 비용 0 (이미 보유)
- 시간 추가 0
- 로봇에 KV260 + 5V 보조전원 탑재
- 기능 검증 충분

단점:
- ASIC 대비 비효율 (~50배 면적, ~10배 전력)
- 로봇 위 무게/크기 부담
```

### 옵션 B: ASIC tape-out (장기, 박사 분량)

```
저비용 ASIC 옵션:
- TinyTapeout: ~$100, 매우 작음 (망막 VPU는 안 들어갈 수도)
- eFabless OpenLane: 무료 (Open MPW), ~수개월 대기
- Skywater 130nm: 검증된 PDK

장점:
- 면적 1/50, 전력 1/10
- 진짜 "망막 칩"
- 학회/특허 임팩트 큼

단점:
- 시간 6개월~1년
- 디버깅 어려움 (틀리면 다시 tape-out)
- 비용 (정식 ASIC은 수억)
```

### 추천 진행
```
[학기 프로젝트]: 옵션 A (KV260 그대로)
[후속 연구]:    옵션 B (ASIC, 결과 좋으면)
```

---

## Stage 6: 로봇 탑재 + RL

### 목표
망막 VPU의 시각 정보로 로봇 행동 학습.

### 로봇 옵션

| 로봇 | 특징 | 추천 작업 |
|------|------|----------|
| **TurtleBot / JetBot** | 저렴, 실내 네비게이션 | 충돌 회피, 목표 추종 |
| **Unitree Go** | 사족 로봇, 야외 | 지형 인식, 객체 추적 |
| **Crazyflie** | 드론, 가볍게 KV260 비탑재 (외부 처리) | 시각 기반 비행 |
| **시뮬만** | CARLA, Isaac Gym | 빠른 RL 학습 |

### RL 환경

```python
class RetinalRL_Agent:
    def __init__(self):
        self.vpu = KV260_VPU_Hardware()      # FPGA 가속
        self.policy = PolicyNetwork(in_ch=22)  # PyTorch 학습 대상
        self.value = ValueNetwork(in_ch=22)
    
    def step(self, observation):
        # 환경에서 RGB observation 받음
        rgb = observation  # (H, W, 3)
        
        # VPU 통과 (하드웨어 가속)
        retinal_features = self.vpu(rgb)  # (22, H, W)
        
        # Policy 예측
        action = self.policy(retinal_features)
        log_prob = self.policy.log_prob(action)
        value = self.value(retinal_features)
        
        return action, log_prob, value
    
    def learn(self, batch):
        # PPO, A3C, SAC 등 표준 RL 알고리즘
        # VPU는 freeze, policy/value만 학습
        ...
```

### 비교 실험 (논문용)

```
실험: 충돌 회피 / 목표 추종 / 지형 적응 등

대조군 1: RGB → Policy Network
대조군 2: VPU 22ch → Policy Network
대조군 3: VPU 22ch → Two-Stream Policy

가설:
- VPU의 시간 채널 (motion, transient)이 동적 환경에 강함
- 같은 학습 에피소드로 더 빠르게 수렴
- 또는 같은 학습 후 generalization 더 좋음
```

### RL 알고리즘
- **PPO**: 표준, 안정적
- **SAC**: 연속 행동
- **DQN**: 이산 행동 (간단한 경우)
- **A3C**: 비동기 학습

### 산출물
- 로봇이 망막 VPU로 행동하는 데모
- 학습 곡선 (RGB vs VPU)
- 시각화 (어떤 채널이 결정에 기여)

---

## 시간 추정 (총 8~12개월)

| Stage | 작업 | 예상 시간 |
|-------|------|----------|
| 1 | 22ch VPU HLS | 4~6주 |
| 2 | 카메라 연동 | 2~3주 |
| 3 | 22ch 출력 검증 | 1~2주 |
| 4 | 스트림 검증 | 1주 |
| 5 | (옵션 A) 그대로 | 0주 |
|   | (옵션 B) ASIC | 6~12개월 |
| 6 | 로봇 + RL | 4~8주 |

**학기 프로젝트** = Stage 1~4 (3~4개월) 분량.  
**졸업 연구** = Stage 1~6 + 옵션 B = 약 1~2년.

---

## 참고

- **work_state.md**: 프로젝트 전체 상태 + 학습 기록
- **vpu_ip_plan.md**: Phase A~E (HLS IP 작성 세부)
- **study_session_overlay.md**: PYNQ Overlay + 핵심 개념
- **lessons_learned.md**: 디버깅 + 막혔던 곳들 정리
- **retinal_vpu_channels.md**: 22채널 생물학적 매핑
