# 디버깅 + 헷갈렸던 것들 정리

오늘 (2026-05-16) 진행 중 막히거나 헷갈렸던 부분들 + 해결 + 핵심 인사이트.

---

## 목차
1. [Vivado: 여러 Block Design 중 어느 게 합성되는가](#1-vivado-여러-block-design-중-어느-게-합성되는가)
2. [.bit과 .hwh가 매칭 안 되는 문제](#2-bit과-hwh가-매칭-안-되는-문제)
3. [Export Hardware가 잘못된 BD 가져감](#3-export-hardware가-잘못된-bd-가져감)
4. [PYNQ에서 우리 IP가 안 보임 (해결)](#4-pynq에서-우리-ip가-안-보임-해결)
5. [22채널 데이터 어떻게 모으나? (큰 오해)](#5-22채널-데이터-어떻게-모으나-큰-오해)
6. [AI 모델이 22채널을 어떻게 이해하나?](#6-ai-모델이-22채널을-어떻게-이해하나)
7. [정적 vs 동적 벤치마크](#7-정적-vs-동적-벤치마크)
8. [RL은 어떻게 진행되나](#8-rl은-어떻게-진행되나)
9. [TDATA 폭 매칭 (3바이트 vs 2바이트)](#9-tdata-폭-매칭)

---

## 1. Vivado: 여러 Block Design 중 어느 게 합성되는가

### 헷갈렸던 점
프로젝트 안에 BD가 여러 개:
```
Design Sources:
├─ design_1_wrapper  ← Top? 합성됨?
├─ FirstProject       ← ?
└─ Result1_wrapper    ← ?
```

### 핵심
**Top으로 설정된 wrapper 1개만 합성됨**. 나머지는 무시.

### 확인 방법

#### GUI
```
Sources 탭 → 굵게 표시된 wrapper = Top
```

#### Tcl Console
```tcl
get_property top [current_fileset]
```

#### Set as Top 변경
```tcl
set_property top Result1_wrapper [current_fileset]
update_compile_order -fileset sources_1
```

또는 GUI: wrapper 우클릭 → Set as Top.

### 핵심 인사이트
> 같은 프로젝트에 여러 BD를 만들면 헷갈림.  
> **1프로젝트 1BD가 깔끔**. 여러 버전 필요하면 Project 자체를 복사.

---

## 2. .bit과 .hwh가 매칭 안 되는 문제

### 헷갈렸던 점
xsa 압축 풀면 안에:
```
vpu_ver2.bit      ← 비트스트림
design_1.hwh      ← 하드웨어 정의 (이름 다름!)
```

PYNQ는 `.bit`과 `.hwh`가 **같은 이름**이어야 자동 매칭.

### 해결
```bash
# rename
mv design_1.hwh vpu_ver2.hwh
```

또는 PowerShell:
```powershell
Rename-Item design_1.hwh vpu_ver2.hwh
```

### 핵심 인사이트
```
PYNQ Overlay 동작:
  Overlay("/path/foo.bit") 호출
  → 같은 경로의 foo.hwh 자동 검색
  → 없으면 에러 또는 IP 인식 X
```

→ **두 파일 같은 이름 + 같은 폴더**에 두기.

---

## 3. Export Hardware가 잘못된 BD 가져감

### 가장 헷갈렸던 문제

Top을 Result1_wrapper로 설정했는데, Export Hardware (xsa)는 design_1의 .hwh를 가져감.

### 진단

#### 합성은 정확히 됐는지 확인
```tcl
# Implementation의 결과 .bit 위치
glob -nocomplain [get_property DIRECTORY [get_runs impl_1]]/*.bit
```
결과: `.../impl_1/Result1_wrapper.bit` ✅

→ **합성은 Result1로 잘 됨**.

#### .hwh 확인
```powershell
Get-Content vpu_ver3.hwh -TotalCount 5
```
결과:
```xml
<SYSTEMINFO ... NAME="design_1" .../>
```

→ **.hwh는 design_1 기준** ❌

### 해결: xsa 우회. 파일 직접 SCP

xsa 만들고 압축 풀고 rename 하는 거 다 건너뛰고, Vivado 결과 폴더에서 직접 가져옴:

```powershell
# .bit (impl_1 폴더)
scp project_2.runs/impl_1/Result1_wrapper.bit ubuntu@KV260:~/result1.bit

# .hwh (BD 자체의 hw_handoff)
scp project_2.gen/sources_1/bd/Result1/hw_handoff/Result1.hwh ubuntu@KV260:~/result1.hwh
```

→ **이름 통일 (result1.*)** + KV260로 직접.

### 핵심 인사이트
```
Vivado의 Export Hardware는 가끔 잘못된 BD 선택함
(특히 프로젝트에 여러 BD 있을 때)

대안: project_2.runs/impl_1/와 
      project_2.gen/sources_1/bd/<BD>/hw_handoff/ 직접 사용
```

---

## 4. PYNQ에서 우리 IP가 안 보임 (해결)

### 증상
```python
overlay = Overlay("/home/ubuntu/vpu_ver2.bit", ignore_version=True)
print(overlay.ip_dict.keys())
# → mipi_csi2_rx_subsyst_0, v_frmbuf_wr_0, zynq_ultra_ps_e_0
# chroma_extractor_0 안 보임! ❌
```

### 원인 진단 순서

#### Step 1: .hwh 안 chroma 키워드 있나
```bash
grep -c chroma /home/ubuntu/vpu_ver2.hwh
# → 0 = 없음
```
→ .hwh가 chroma 없는 BD 기반.

#### Step 2: .hwh의 BD 이름 확인
```bash
head -5 /home/ubuntu/vpu_ver2.hwh | grep NAME
# → NAME="design_1"
```
→ design_1 BD 기반. Result1 아님.

#### Step 3: Vivado에서 Top 확인
```tcl
get_property top [current_fileset]
# → Result1_wrapper
```
→ Top은 맞는데 export만 잘못됨.

### 해결
위 §3과 동일. 파일 직접 SCP.

### 최종 결과
```python
overlay = Overlay("/home/ubuntu/result1.bit", ignore_version=True)
print(overlay.ip_dict.keys())
# → chroma_extractor_0, mipi_csi2_rx_subsyst_0, v_frmbuf_wr_0, zynq_ultra_ps_e_0 ✅
```

### 검증
```python
print(overlay.chroma_extractor_0.register_map)
# AP_START=0, AP_IDLE=1 (idle)

overlay.chroma_extractor_0.register_map.CTRL.AP_START = 1

print(overlay.chroma_extractor_0.register_map)
# AP_START=1, AP_IDLE=0 (running!) ✅
```

→ **PS↔PL AXI 통신 양방향 동작 확인**.

---

## 5. 22채널 데이터 어떻게 모으나? (큰 오해)

### 헷갈렸던 점

> "보통 학습 시킬 때 데이터는 RGB 픽셀인데, 우리는 22채널이라 데이터 다 따로 만들어야 하나?"

### 정답

**아니. 별도 22채널 데이터 수집 X**.

### 이유

VPU는 **학습 안 함**. Deterministic transformation (고정 함수).

```
[ImageNet RGB 150만장]  ← 기존 데이터셋
       ↓
[VPU: 고정 가중치]      ← numpy 함수 같은 거. 학습 X
       ↓
[22ch features 자동]    ← 별도 라벨링 X
       ↓
[ViT 학습]              ← 이것만 학습 대상
```

### 코드 패턴

```python
class Pipeline(nn.Module):
    def __init__(self):
        self.vpu = RetinalVPU()        # 고정 가중치 (학습 X)
        self.vit = SmallViT(in_ch=22)  # 학습 대상

    def forward(self, x):   # x = RGB
        features = self.vpu(x)       # 매 batch 자동 변환
        return self.vit(features)
```

→ **학습 흐름 중 매 batch에 VPU 통과**. 마치 data augmentation 처럼.

### 비유

```
일반 학습:
  RGB → resize, crop, normalize → CNN
  
우리 학습:
  RGB → resize, crop, normalize → VPU → CNN
                                   ↑
                            추가된 전처리 1단계 (학습 X)
```

### 핵심 인사이트
```
VPU = "RGB → 22ch 변환 함수" (고정)
   ≠ "학습하는 모델"

기존 모든 RGB 데이터셋 그대로 사용 가능
22ch annotation 필요 없음
```

---

## 6. AI 모델이 22채널을 어떻게 이해하나?

### 헷갈렸던 점

> "VPU를 만들어도 AI가 22채널의 의미 (Y=밝기, motion=시간) 를 알아야 활용하지 않나?"

### 정답

**모델 아키텍처에 inductive bias 줘야 진짜 의미**. 단순 채널 수 늘리기는 부족.

### 4가지 접근 (난이도 + 효과 순)

#### 🅐 단순 채널 확장 (Baseline, 이미 한 것)
```python
patch_embed = nn.Conv2d(22, 384, ...)  # 3→22로 변경만
```
- 효과: 미미 (CIFAR에서 RGB 98% vs VPU 98%)
- 결론: VPU의 의미를 모델이 무시함

#### 🅑 Two-Stream Network (영상 액션 인식 클래식)
망막의 두 경로 (Parvo + Magno) 흉내:
```python
spatial_stream = ViT(in_ch=10)   # Y, edges, gabor, color
temporal_stream = ViT(in_ch=12)  # motion, transient
output = fusion(spatial, temporal)
```
- 효과: 비디오에서 큰 향상 기대
- Reference: Simonyan & Zisserman 2014

#### 🅒 Multi-Stream + Channel Grouping
각 망막 채널 그룹에 다른 처리:
```python
lum_path   = SpatialEncoder(1)   # Y
color_path = SpatialEncoder(2)   # R-G, B-Y
edge_path  = SpatialEncoder(5)   # edge_mag, gabor*4
temp_path  = TemporalEncoder(4)  # dL/dt, ON/OFF, ...
motion_path = MotionEncoder(4)   # DS_*
```
- 효과: VPU 의미 최대 활용
- 단점: 설계 복잡

#### 🅓 망막 → V1 → V2 → IT 모사 (이상적)
```
VPU → V1(orientation) → V2(form) → V4(color) → IT(object)
```
- Reference: HMAX, CORnet
- 박사 논문 분량

### 현실적 진행

```
Step 1 (이미 완료): 🅐 Baseline
   - CIFAR에서 RGB vs VPU 비교
   - 98% 비슷한 정확도 → "정적 이미지에선 큰 차이 없음"

Step 2 (다음): 🅑 Two-Stream + 동영상 데이터
   - Kinetics-mini 또는 UCF-101
   - 시간 채널 효과 입증

Step 3 (장기): 🅒 Channel-aware 아키텍처
```

### 핵심 인사이트
```
정적 이미지: VPU 효과 미미 (이미 확인된 사실)
   ↓
"VPU의 진짜 가치" = 동적 영상에서만
   ↓
모델 아키텍처도 망막 구조 반영해야 (Two-Stream)
   ↓
논문 한 줄 결과:
"비디오 RGB 대비 VPU 22ch + Two-Stream이 X% 정확도 향상"
```

---

## 7. 정적 vs 동적 벤치마크

### 헷갈렸던 점

> "ImageNet은 정적 이미지. 동영상은 어떤 벤치마크 씀?"

### 답

#### 정적 이미지 (이미 사용)
- **ImageNet**: 표준. 150만 이미지, 1000 클래스
- **CIFAR-10**: 작음. 60K, 10 클래스
- **Imagenette**: 작은 ImageNet 서브셋

#### 동영상 액션 인식 (★ 우리한테 적합)
- **Kinetics-400/600/700**: 사실상 표준
  - 30만 비디오, 10초씩, 400~700 액션 클래스
  - I3D, SlowFast, TimeSformer 다 여기서 평가
- **Something-Something v2**: 시간 추론 강조 ("위→아래" vs "아래→위")
- **UCF-101**: 작고 다운로드 쉬움 (~10GB)

#### 광학 흐름 (Motion 직접 검증)
- **MPI Sintel**: 합성, GT optical flow
- **KITTI Flow**: 자율주행 실데이터

#### Neuromorphic (망막 영감 커뮤니티)
- **DVS128 Gesture**: 이벤트 카메라 제스처
- **N-Caltech101, N-MNIST**: 이벤트 변환 버전

### 우리한테 가장 적합

```
1순위: Kinetics-400 (영향력)
2순위: Something-Something v2 (시간 추론 강조)
3순위: DVS128 Gesture (Neuromorphic 어필)
4순위: MPI Sintel (motion 자체 평가)
```

### 핵심 인사이트
```
ImageNet = 사진 1장 → 한 라벨
Kinetics = 영상 1개 (10초, 300프레임) → 한 액션 라벨

우리 VPU의 시간 채널 12개는 Kinetics 같은 동영상에서만 의미
정적 이미지엔 시간 미분 = 0
```

---

## 8. RL은 어떻게 진행되나

### 사용자 의도 정리

> "칩셋 만들고 작동 확인 → 로봇 RL 학습"

### 단계별 흐름

```
Stage 1-4: KV260에서 22ch VPU 동작 검증
   ↓
Stage 5: KV260 그대로 로봇에 탑재 (또는 ASIC)
   ↓
Stage 6: RL
```

### RL Agent 구조

```python
class RetinalRobotAgent:
    def __init__(self):
        self.vpu = KV260_VPU_Hardware()       # FPGA 가속
        self.policy = PolicyNetwork(in_ch=22) # PyTorch 학습 대상
        self.value = ValueNetwork(in_ch=22)
    
    def step(self, observation):
        rgb = observation  # 환경에서 받는 RGB (카메라)
        retinal_features = self.vpu(rgb)  # 하드웨어 가속
        action = self.policy(retinal_features)
        value = self.value(retinal_features)
        return action, value
    
    def learn(self, batch):
        # VPU는 freeze, policy/value만 학습
        # PPO, SAC 등 표준 RL
        ...
```

### 환경 옵션

| 환경 | 종류 | 추천 task |
|------|------|----------|
| **CARLA** | 자율주행 시뮬 | 운전, 충돌 회피 |
| **Habitat** | 실내 시뮬 | 네비게이션 |
| **Isaac Gym** | 다목적 시뮬 | 객체 조작 |
| **TurtleBot 실제** | 저렴 실로봇 | 추종, 회피 |
| **자체 시뮬** | 단순 | VPU 효과 데모 |

### 비교 실험 (논문)

```
Task: 동적 환경 충돌 회피

대조 1: RGB → Policy Network
대조 2: VPU 22ch → Policy Network  
대조 3: VPU 22ch → Two-Stream Policy

가설:
- VPU의 motion/transient 채널이 동적 장애물 회피에 유리
- 같은 학습 시간에 더 높은 성공률
- 또는 같은 성공률 도달까지 더 적은 에피소드
```

### 핵심 인사이트
```
RL = "환경 RGB → VPU(고정) → Policy(학습) → 행동"
환경은 RGB를 줌 (Atari, CARLA, 실제 카메라 등)
VPU가 RGB → 22ch 변환 (학습 X)
Policy만 PPO/SAC 등으로 학습
```

→ **RL 데이터도 별도 수집 안 함**. 환경에서 RGB 받아서 VPU 통과.

---

## 9. TDATA 폭 매칭

### 헷갈렸던 점

```
ERROR: Bus Interface property TDATA_NUM_BYTES does not match
  chroma_extractor.in_stream(3) vs mipi_csi2_rx.video_out(2)
```

### 원인
- 우리 IP를 24bit (3바이트) 입력으로 만들었음
- MIPI Rx 출력은 16bit (2바이트)
- 폭 불일치

### 해결

```cpp
// chroma_extractor.h
typedef ap_axiu<16, 1, 1, 1> pixel_in_t;   // 16bit 입력
typedef ap_axiu<24, 1, 1, 1> pixel_out_t;  // 24bit 출력 (Frame Buf용)

void chroma_extractor(
    hls::stream<pixel_in_t>& in_stream,
    hls::stream<pixel_out_t>& out_stream
) {
    while (true) {
        pixel_in_t in = in_stream.read();
        pixel_out_t out;
        out.data = (ap_uint<24>)in.data;  // 16→24 zero padding
        ...
    }
}
```

### 핵심 인사이트
```
AXI-Stream IP 통합 시 TDATA 폭 매칭 필수
- MIPI Rx 출력: 16bit (YUV422 1 픽셀)
- Frame Buf Write 입력: 24bit (RGB888 호환)
- 변환 IP가 그 사이에 필요

이전엔 axis_subset_converter가 했던 일
지금은 우리 IP가 그 역할 흡수
```

---

## 메타 교훈

### 1. 디버깅은 진단부터
```
"왜 안 돼" → "실제로 무엇이 어떻게 됐나"부터 확인
- Top 설정?
- 합성 결과?
- Export 결과?
- 단계마다 별도 확인
```

### 2. Vivado UI 신뢰 X, Tcl 신뢰 O
```
GUI의 "Set as Top" → 실제 적용 안 되는 경우 있음
→ Tcl Console에서 직접 확인 + 명령
```

### 3. xsa 우회
```
Export Hardware가 자주 잘못된 BD 가져감
→ project_2.runs/impl_1/와 hw_handoff/ 직접 SCP
```

### 4. PYNQ는 이름 매칭 엄격
```
.bit과 .hwh = 같은 이름 + 같은 폴더
```

### 5. VPU는 학습 X
```
망막 알고리즘은 deterministic
RGB 데이터셋 그대로 사용
별도 22ch 데이터 수집 불필요
```

### 6. 정적/동적 분리
```
ImageNet (정적) = VPU 효과 미미
Kinetics (동적) = VPU 시간 채널 가치 입증
논문 평가 시 둘 다 필요
```

---

## 다음 작업 체크리스트

- [ ] Phase C: HLS 22채널 VPU 작성
- [ ] PyTorch 비디오 데이터셋 준비 (UCF-101 또는 Kinetics-mini)
- [ ] Two-Stream Network 구조 설계
- [ ] HLS C Simulation에서 PyTorch와 cross-validation
- [ ] KV260에서 22ch 시각화
- [ ] 로봇 / 시뮬 환경 결정
- [ ] RL 알고리즘 (PPO) 적용

---

_다음에 또 헷갈리는 거 만나면 이 파일에 추가._
