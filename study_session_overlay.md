# PYNQ Overlay & FPGA Deep Dive — Study Session

KV260에서 비트스트림 로드 성공한 후, 실제로 그 안에서 무엇이 일어나는지 깊이 들여다본 학습 세션 기록.

각 섹션은 **Q (질문)** + **A (답변)** 형태로 그대로 보존. 다른 환경에서 공부할 때 그대로 참조.

---

## 목차
1. [비트스트림 적용 메커니즘](#1-비트스트림-적용-메커니즘)
2. [Overlay 실행 시 CPU/팬 부하 이유](#2-overlay-실행-시-cpu팬-부하-이유)
3. [LUT, FF, 배선 세부 역할](#3-lut-ff-배선-세부-역할)
4. [PL 자원 사용량 — 256K 중 몇 개?](#4-pl-자원-사용량--256k-중-몇-개)
5. [안 쓰는 자원의 상태](#5-안-쓰는-자원의-상태)
6. [ASIC vs FPGA 트레이드오프 + DPU](#6-asic-vs-fpga-트레이드오프--dpu)
7. [PS와 PL 구분](#7-ps와-pl-구분)
8. [AXI 프로토콜 (Lite / Stream / Full)](#8-axi-프로토콜-lite--stream--full)
9. [MIPI CSI-2 + AP1302](#9-mipi-csi-2--ap1302)
10. [PYNQ가 파이썬 라이브러리인가?](#10-pynq가-파이썬-라이브러리인가)
11. [Overlay 정확한 동작 (6단계)](#11-overlay-정확한-동작-6단계)
12. [학습 진척 정리](#12-학습-진척-정리)

---

## 1. 비트스트림 적용 메커니즘

### Q
> 근데 궁금한거 또 하나는, 여기서 사실상 컴퓨터 하나에 비트스트림을 다 만들었는데 따로 적용했다고 보이진 않았거든, 그냥 파일만 옮겼을 뿐인데 왜 갑자기 카메라 인식이 되는거야?

### A

오 이거 FPGA의 핵심 개념 질문. 정말 중요함.

#### 일반 컴퓨터 vs FPGA의 본질적 차이

```
일반 CPU (i5, ARM 등):
  하드웨어 = 고정 (공장에서 만들어진 회로 그대로)
  소프트웨어 = 변경 (코드 실행)

FPGA:
  하드웨어 = 매번 변경 가능  ← ★
  비트스트림 = "어떤 회로로 만들지 청사진"
```

→ 즉 비트스트림 = **"PL fabric의 LUT, FF, BRAM, 배선을 어떻게 연결할지" 정의 파일**.

#### 비유

```
일반 컴퓨터:
  공장에서 납땜된 회로기판
  → 회로 못 바꿈, 코드만 실행

FPGA:
  레고 같은 거 + 자동 조립기
  → 비트스트림 = "이렇게 조립해" 설명서
  → 매번 다른 회로 만들 수 있음 (수 ms)
```

#### "갑자기 카메라 인식" 단계별 풀이

```
1. PC에서 Vivado가 .bit 파일 생성
   = "MIPI CSI Rx 회로는 LUT 1234~5000번에,
      Frame Buf는 LUT 5001~9000번에..."
      식으로 모든 LUT/FF 배치 정의

2. SCP로 KV260에 .bit 파일 전송
   = 그냥 파일 복사. 아직 PL은 안 변함.

3. PYNQ Overlay 로드 (overlay = Overlay("design_1.bit"))
   = 진짜 마법이 여기서 일어남:
     a. Linux의 fpga_manager driver가 .bit 읽음
     b. PS → PL의 PCAP 인터페이스로 비트스트림 전송
     c. PL fabric이 자기 LUT/FF/배선을 재프로그램
     d. 수 밀리초 만에 PL이 통째로 새 회로로 변경됨

4. 이제 PL = 우리 Vivado 디자인 그대로
   = MIPI CSI Rx, Frame Buf, SmartConnect 등이 실제로 회로로 존재
```

#### 그래서 PL 안에는 "물리적 회로"가 생긴 것

```
[전기적으로] PL 안에:
- LUT 수만 개 중 일부가 MIPI Rx 로직 흉내냄
- DSP, BRAM이 데이터 버퍼링
- 배선이 IP 간 신호 전달
- 외부 핀이 보드의 카메라 커넥터에 연결됨
```

→ 정말로 **회로가 만들어진 것**. 단지 LUT라는 재구성 가능한 부품으로.

#### dmesg에서 봤던 그 줄

```
fpga_manager fpga0: writing design_1.bin to Xilinx ZynqMP FPGA Manager
```

이게 바로 그 순간:
- "design_1.bin (비트스트림)을 FPGA에 굽고 있어"
- 약 100ms~1s 동안 PL fabric 재프로그램 진행
- 끝나면 PL = 우리 회로

#### 카메라 인식 보충

> "갑자기 카메라 인식이 되는 거야?"

사실 **`/dev/video*`는 아직 안 됨**. 우리가 만든 건:
- ✅ PL 회로 = MIPI CSI Rx, Frame Buf 같은 디지털 회로 부분
- ❌ Linux에게 그 회로 알려주는 device tree (.dtbo) = 안 만듦

비유:
```
✅ 차 만듦 (PL 회로)
❌ DMV에 등록 안 함 (device tree)
→ 차는 굴러가지만 합법적으로 도로 못 다님 (Linux가 모름)
```

그래서 PYNQ에서 IP 직접 제어는 가능하지만, Linux의 standard camera driver는 안 뜸.

#### 정리

```
.bit 파일 = "이런 회로 만들어줘" 청사진
Overlay 로드 = 청사진 → 실제 회로 변환 (PL fabric 재프로그램)
파일 옮김 = 단순 전송, 아직 PL 변경 X
실행 = 진짜 회로 생성
```

이게 FPGA가 일반 CPU와 본질적으로 다른 이유. **소프트웨어가 아니라 하드웨어를 다루는 것**.

---

## 2. Overlay 실행 시 CPU/팬 부하 이유

### Q
> 그리고 그럼 궁금한게 overlay 코드를 실행했을 때, 리소스 많이 먹는 듯이 CPU랑 팬 속도가 빨라지고 사용량이 높아졌거든, 그 이유는 뭐야?

### A

#### 그 한 줄 코드 안에서 실제로 일어나는 일

```python
overlay = Overlay("design_1.bit")
```

이게 사실 **5단계 작업의 압축**:

#### 단계별 부하

| 단계 | 부하 | 시간 |
|------|------|------|
| 1. `.bit` 파일 읽기 (~3MB) | 디스크 I/O | ms |
| 2. `.hwh` XML 파싱 (수만 줄) | **CPU 부하 ↑** | ~수백 ms |
| 3. PL fabric 재프로그램 (PCAP 전송) | **PS↔PL 대용량 전송** | ~1s |
| 4. PL의 모든 회로 power-on (transient current) | **전기적 부하 ↑↑** | ~ms |
| 5. PYNQ가 IP dict 생성 + Python 객체 (수백 개) | **CPU 부하 ↑↑** | 수 초 |

#### 팬 속도가 빨라지는 진짜 이유 = **단계 4**

```
PL fabric이 우리 디자인으로 변경되는 순간:
- 수만 개 LUT 동시 활성화
- 100MHz, 200MHz 클럭 회로 동시 동작
- 모든 FF가 매 사이클마다 전기 소모
- Frame Buf, SmartConnect 등 큰 회로 활성

→ PL의 power consumption이 갑자기 크게 증가
→ 칩 온도 상승
→ 보드의 thermal sensor 감지
→ BMC가 팬 속도 자동 증가
```

비유:
```
Overlay 로드 전: 빈 공장 (전기 거의 안 먹음)
Overlay 로드 후: 모든 기계 동시 가동 (전기 확 먹음)
```

#### CPU 부하는 단계 2 + 5

- `.hwh` 파싱 (XML 트리 분석)
- PYNQ가 모든 IP에 대해 Python 객체 생성, driver 검색, 메모리 매핑 등
- 전부 ARM Cortex-A53가 처리 → CPU 사용률 일시적 폭증

---

## 3. LUT, FF, 배선 세부 역할

### Q
> LUT FF 배선의 역할을 좀 더 세부적으로 설명해줘

### A

#### PL fabric 내부 구조 (KV260의 ZU5EV 기준)

```
┌──────────────────────────────────────────┐
│   PL Fabric (256K LUTs, 234K FFs)       │
│                                           │
│   ┌─CLB─┐ ┌─CLB─┐ ┌─CLB─┐ ┌─BRAM─┐      │
│   │     │ │     │ │     │ │ 36KB │      │
│   │ LUT │ │ LUT │ │ LUT │ │      │      │
│   │  FF │ │  FF │ │  FF │ └──────┘      │
│   └─────┘ └─────┘ └─────┘                │
│      ↕      ↕      ↕                     │
│   ═══════════════════ Routing ═══════    │
│      ↕      ↕      ↕                     │
│   ┌─DSP─┐ ┌─CLB─┐ ┌─CLB─┐                │
│   │곱셈 │ │     │ │     │                │
│   └─────┘ └─────┘ └─────┘                │
└──────────────────────────────────────────┘
```

#### 1) LUT (Look-Up Table) — "조합 회로"

##### 정체
**입력 → 출력의 진리표를 메모리로 저장한 작은 SRAM**.

```
6-input LUT = 64 bit 메모리
         ┌─────────────────────┐
입력 6bit → 어드레싱 → 64bit 메모리 → 1bit 출력
         └─────────────────────┘
```

##### 핵심: **무슨 boolean 함수든 LUT 하나로 구현**

예) AND 게이트:
```
입력 A B → 출력
0    0    0
0    1    0
1    0    0
1    1    1   ← 이 진리표를 LUT 메모리에 저장
```

→ "어떤 조합 회로든 LUT으로 변환 가능"이 FPGA의 핵심 발명.

##### 비트스트림이 LUT에서 하는 일

```
비트스트림 = "LUT_1234번에 이 64bit 메모리 값 써넣어"
           = "LUT_5678번에 이 32bit 메모리 값 써넣어"
           ...
```

진리표 자체가 비트스트림에 인코딩됨.

##### 예시: MIPI Rx 안의 어떤 부분
```
"video_aclk이 high이고 reset이 low면 카운터 +1"
= AND(video_aclk, NOT(reset)) → counter_enable
= 4-input LUT 1개로 구현
```

#### 2) FF (Flip-Flop) — "순차 회로 + 상태 저장"

##### 정체
**클럭 매 사이클마다 1비트 저장하는 메모리**.

```
       ┌──────┐
입력 D ─→│      │
         │  FF  │── 출력 Q (이전 사이클 D 값)
clk ───→│      │
         └──────┘
```

##### 핵심: **"이전 사이클 값을 기억"**

```
clock:    ▁▔▁▔▁▔▁▔
D 입력:   1  0  1  1
Q 출력:    1▔▁▔▔▁▔▔▁▔     ← 항상 한 사이클 늦음
```

##### 어디 쓰이나
- **카운터**: 매 사이클마다 +1 누적 (FF 32개로 32bit 카운터)
- **상태 머신**: 현재 상태 저장 (예: IDLE → REQ → DATA → DONE)
- **파이프라인**: 매 단계 결과 임시 저장
- **레지스터**: AXI 인터페이스의 모든 데이터 라인 중간 저장

##### 예시: 우리 디자인의 어떤 부분
```
Frame Buffer Write 안:
- 현재 픽셀 카운터 (1920×1080 셈) → FF ~21개
- 라인 카운터 → FF ~11개
- DDR 주소 → FF 64개
...
이런 식으로 수천 개 FF가 각자 1비트씩 상태 저장
```

#### 3) 배선 (Routing Fabric) — "프로그래머블 전선"

##### 정체
**LUT, FF, BRAM 등을 연결하는 와이어망 + 스위치**.

```
LUT_1 ────┬──── 스위치 ────┬──── FF_5
          │       │         │
          └───스위치────┐   │
                        │   │
LUT_2 ────────────────┘   │
                            │
LUT_3 ──────────────────────┘
```

각 교차점에 **switch matrix**가 있음. 비트스트림이 "이 스위치는 ON, 저 스위치는 OFF" 정의 → 어느 LUT 출력이 어느 FF 입력으로 갈지 결정.

##### 비트스트림에서 차지하는 비중

```
비트스트림 ~3MB 중:
- LUT 진리표:  ~30%
- FF 초기값:    ~5%
- 배선 스위치 설정: ~50%  ← 가장 큼
- 기타(BRAM, DSP 설정): ~15%
```

→ **사실 비트스트림의 절반은 "어떤 와이어를 연결할지"** 정의.

##### 왜 그렇게 큰가
ZU5EV는 LUT 사이 가능한 연결이 **수백만 가지**. 그 중 어떤 걸 활성화할지 다 비트로 표시해야 함.

#### 추가 부품 (보너스)

| 부품 | 역할 | 우리 디자인에서 |
|------|------|---------------|
| **BRAM** | 큰 메모리 (36Kb 한 블록) | MIPI Rx의 line buffer, FIFO |
| **DSP** | 곱셈/누산 전용 | 망막 VPU IP 만들면 사용 (곱셈) |
| **BUFG** | 클럭 글로벌 분배 | pl_clk0, pl_clk1 분배 |
| **MMCM/PLL** | 클럭 주파수 변환 | clk_wiz의 100→200MHz 변환 |
| **IO Block** | 외부 핀 ↔ PL 인터페이스 | mipi_phy_if 외부 IO |

#### 한 줄 정리

```
LUT  = "이 입력엔 이 출력" 진리표 (조합 회로 만들기)
FF   = 1비트 상태 저장 (순차 회로 만들기)
배선 = LUT/FF 사이 연결 + 스위치 (회로 토폴로지)

비트스트림 = LUT 진리표 + FF 초기값 + 배선 스위치 설정의 거대한 조합
```

우리가 만든 비트스트림 = 이 모든 게 정확히 "MIPI CSI Rx 받아서 Frame Buf로 보내는 회로" 가 되도록 LUT/FF/배선 다 설정한 ~3MB 파일.

---

## 4. PL 자원 사용량 — 256K 중 몇 개?

### Q
> 그러면 256 K 중에 우리는 몇만개만 쓴거임 현재?

### A

#### 먼저 정정: ZU5EV의 실제 자원

이전에 "256K LUTs"라고 했는데 약간 부정확. AMD datasheet:

| 자원 | 양 |
|------|---|
| **System Logic Cells** | 256K (LUT + 알파, 마케팅 숫자) |
| **CLB LUTs** | **117,120** ← 진짜 LUT |
| **CLB Flip-Flops** | 234,240 |
| **BRAM (36Kb)** | 144 blocks (5.1 Mb) |
| **UltraRAM** | 64 blocks |
| **DSP slices** | 1,248 |

→ 진짜 LUT는 **117K**.

#### 우리 디자인 사용량 추정

각 IP별 대략:

| IP | LUT 추정 |
|----|---------|
| Zynq UltraScale+ MPSoC (PS interface) | ~500 |
| MIPI CSI-2 Rx Subsystem | ~6,000 |
| Video Frame Buffer Write | ~3,000 |
| AXI SmartConnect (제어, 다중 클럭) | ~2,000 |
| AXI SmartConnect (데이터) | ~1,500 |
| AXI4-Stream Subset Converter | ~50 |
| Clocking Wizard | ~10 (PLL 자체는 LUT 안 씀) |
| Processor System Reset ×3 | ~150 |
| Inline Concat | ~0 |

**합계 약 13,000 LUT** = 117K의 **약 11%**.

#### 정확한 수치 보는 법

##### Vivado에서

1. 좌측 Flow Navigator → **IMPLEMENTATION → Open Implemented Design**
2. 메뉴 **Reports → Report Utilization**
3. 또는 직접 파일 열기:
```
C:/Users/ejfhr/FPGA/project_2/project_2.runs/impl_1/design_1_wrapper_utilization_placed.rpt
```

이 .rpt 파일 안에 표 형식으로:
```
+----------------------------+------+-------+-----------+-------+
|          Site Type         | Used | Fixed | Available | Util% |
+----------------------------+------+-------+-----------+-------+
| CLB LUTs                   | XXXX |     0 |    117120 | XX.XX |
| CLB Registers              | XXXX |     0 |    234240 | XX.XX |
| Block RAM Tile             |   XX |     0 |       144 | XX.XX |
| DSPs                       |    X |     0 |      1248 |  X.XX |
+----------------------------+------+-------+-----------+-------+
```

→ 정확한 사용량 다 보임.

#### 의미

만약 우리 디자인이 **~11% 쓴다면**:
- LUT 89%, FF 90%+, DSP 99%, BRAM 95% 가 **여유**
- 망막 VPU IP 추가할 자원 **충분**
- HLS로 만든 IP는 보통 1,000~10,000 LUT 정도
- 22채널 retinal VPU 다 넣어도 LUT 50K 미만일 듯 → **여유 충분**

#### 비교

```
[지금 디자인]    ~13K LUT (11%) — 인프라만
[VPU 추가 후]    ~30K LUT (25%) 예상
[VCU 풀 디자인]  ~80K LUT (70%) — Xilinx 공식 SmartCam 정도
```

→ 우리 디자인은 **아직 fabric 매우 여유 있음**. 자원 걱정 없이 IP 추가 가능.

---

## 5. 안 쓰는 자원의 상태

### Q
> 나머지는 활성화 안된거고?

### A

정확. **물리적으론 칩 안에 다 있지만 회로상 미사용** 상태.

#### 자원 상태 분류

```
KV260의 PL 칩 안 (XCZU5EV):
┌─────────────────────────────────────────┐
│  LUT 117,120개 + FF 234,240개 등        │
│                                          │
│  ├─ 우리가 쓴 ~13K LUT     (활성화)    │
│  │   → 비트스트림 진리표대로 동작        │
│  │   → 클럭 받음, switching함 → power씀  │
│  │                                       │
│  └─ 나머지 ~104K LUT       (미활성화)   │
│      → 진리표 0 또는 don't care          │
│      → 클럭 X, signal X → power 거의 0  │
│      → 그냥 잠자고 있음                  │
└─────────────────────────────────────────┘
```

#### 미활성화 = 어떻게 생겼나

비트스트림은 사실 **모든 LUT/FF/스위치에 설정값**을 줌. 안 쓰는 거는:

```
LUT_99999 진리표: 000000...0 (전부 0)
LUT_99999 입력: 어디 연결도 안 됨 (배선 스위치 OFF)
LUT_99999 출력: 어디 연결도 안 됨

→ 동작은 하지만 입력 0 → 출력 0 → 다른 데 연결도 안 됐으니 의미 없음
→ 사실상 잠자는 상태
```

#### 전력 측면

```
활성 LUT:       클럭 매 사이클 switching → 전력 소모
미활성 LUT:     입력 안 변함 → switching 없음 → 전력 거의 0 (leakage만)
```

→ **안 쓰는 자원은 전기 거의 안 씀**. 그래서 "큰 디자인" vs "작은 디자인"의 power 차이가 큼.

#### 다음 디자인에서 활용 가능

```
지금: ~104K LUT 여유

예상 자원:
- ChromaExtractor IP (HLS):  ~500 LUT
- Spatial VPU 8채널 (HLS):  ~5,000 LUT
- Temporal VPU 4채널 (HLS): ~3,000 LUT
- Motion VPU 4채널 (HLS):    ~10,000 LUT

→ 22채널 retinal VPU 다 넣어도 ~20K LUT 추가
→ 여전히 80K+ LUT 여유
```

#### 비유

```
PL fabric = 큰 공장
- 활성 회로 = 가동 중인 라인
- 미활성 회로 = 비어있는 공장 자리

새 IP 추가 = 빈 자리에 새 라인 설치
KV260 fabric에 우리 회로 + 망막 VPU 다 넣고도 여유 충분
```

---

**핵심 한 줄**: 안 쓴다고 사라지지 않음. 잠자고 있을 뿐. 다음 비트스트림에서 깨워서 쓸 수 있음.

---

## 6. ASIC vs FPGA 트레이드오프 + DPU

### Q1 — 비효율 추론
> 6-input LUT = 64 bit 메모리... 이렇게 되는데 만약 이게 FPGA가 아니라 그냥 전용 칩이었다면 그냥 1bit를 그대로 전송하면 됐던건가?
> 회로를 자유자재로 바꿔야하기때문에 6bit이 1bit으로 출력되는 비효율이 발생하는거임?

### A1

🎯 **정확히 짚었음**. 그게 **FPGA의 본질적 비용**임.

#### ASIC vs FPGA: AND 게이트 하나 비교

##### ASIC (전용 칩)
```
AND 게이트:
  트랜지스터 ~6개 (CMOS)
  면적 매우 작음
  지연 ~10ps
  전력 거의 0
```

##### FPGA (LUT으로 AND 구현)
```
6-input LUT으로 AND 흉내내기:
  ├─ 64bit SRAM (진리표 저장):  트랜지스터 ~384개
  ├─ 64:1 MUX (어드레싱):       트랜지스터 ~63개
  ├─ 입력 버퍼:                 트랜지스터 ~12개
  └─ 출력 버퍼 + 배선 스위치:    트랜지스터 ~50+개

  합계 ~500+ 트랜지스터
  지연 ~50ps (3배 느림)
  전력 ~10× 더 씀
```

#### 그 중 64비트 진리표는?

```
AND의 진리표:
000000 → 0
000001 → 0
000010 → 0
000011 → 0  ← 안 쓰는 입력 조합도 다 0 저장
...
111111 → 1  ← 마지막만 1

→ 64bit 중 의미있는 정보는 1 bit
→ "재구성 가능하게 하려고" 나머지 63bit이 낭비
```

#### 정리: 트레이드오프

| 항목 | ASIC | FPGA |
|------|------|------|
| **면적** | 1× | ~50~80× |
| **속도** | 1× | ~3× 느림 |
| **전력** | 1× | ~10~15× |
| **변경** | ❌ 한 번 만들면 끝 | ✅ 매번 다른 회로 |
| **개발 시간** | 1~2년 | 1~2주 |
| **NRE 비용** | 수십억원 | 0 |

#### 왜 그래도 FPGA?

```
FPGA 비용 = 재구성 가능성을 위한 대가

장점:
1. 알고리즘 바꾸면 즉시 새 회로 (수 분)
2. 처음 만드는 거에 위험 부담 없음
3. 소량 생산 가능 (1개도 OK)
4. 학습/연구에 적합
```

### Q2 — DPU
> 근데 이렇게 비효율적이라고 해도, 전용 회로로 설계되어서 효율적인거야? DPU가 FPGA로 된다는데 이게 FPGA로 하는건지 그냥 DPU로 따로 만든건지 궁금해서 ASIC으로

### A2

#### 질문 1: FPGA가 비효율적인데도 GPU/CPU보다 효율적?

**네**. "회로 자체를 만든다"는 개념 때문에 특정 작업에선 GPU보다 빠르고 전력 효율 좋음.

##### 왜 그런가 — 오버헤드 비교

```
CPU에서 픽셀 처리:
  매 픽셀마다:
  1. 명령어 fetch       (오버헤드)
  2. 명령어 decode      (오버헤드)
  3. 레지스터 read      (오버헤드)
  4. 실제 연산 ★
  5. 레지스터 write     (오버헤드)
  6. 다음 명령어 점프    (오버헤드)

  → 실제 연산은 전체 시간의 ~10%

GPU도 비슷:
  - 수천 thread 병렬이지만 thread scheduler, warp 관리 오버헤드
  - 메모리 latency 숨기느라 batch 필요

FPGA:
  데이터 흐름 따라 회로 자체가 만들어짐:

  [픽셀 in] ─▶ [Sobel 회로] ─▶ [임계값 회로] ─▶ [픽셀 out]

  → 매 사이클 픽셀 1개씩 흐름. 오버헤드 0.
  → 100 MHz면 1억 픽셀/초 (1080p@48fps)
```

##### 비유

```
CPU = 만능 식당 셰프 한 명
  주문 받음 → 메뉴 확인 → 재료 가져옴 → 요리
  매번 같은 작업 반복

GPU = 식당 1000개 (병렬)
  같은 메뉴 1000개 동시. 빠르지만 식당 관리 오버헤드.

FPGA = 통조림 공장 컨베이어
  재료가 컨베이어 따라 자동 가공
  → 사람 없이 매 초 수만 캔 생산
  → 단, 다른 메뉴 만들려면 컨베이어 재구성 (= 비트스트림 변경)
```

#### 질문 2: DPU = FPGA? ASIC?

##### 정답: KV260의 DPU = **FPGA에 합성된 IP** (ASIC 아님)

```
[KV260의 PL fabric]
   ↓
[Xilinx가 만든 DPU IP를 비트스트림에 포함]
   ↓
[PL이 그 회로로 재구성됨]
   ↓
[DPU = LUT + FF + DSP + BRAM 조합으로 만들어진 가속기]
```

→ **재구성 가능한 ASIC 같은 IP**. 칩 자체에 박힌 고정 회로 아님.

##### 다른 AI 가속기들과 비교

| 가속기 | 종류 | 위치 |
|--------|------|------|
| **NVIDIA Tensor Core** | ASIC | GPU chip에 박힘 |
| **Apple Neural Engine** | ASIC | M칩에 박힘 |
| **Google TPU** | ASIC | TPU chip 자체 |
| **Xilinx DPU (KV260)** | **FPGA IP** | PL에 합성 |
| **Versal AI Engine** | ASIC | Versal chip에 박힘 |

→ DPU는 ASIC AI 가속기들과 비교하면 효율 떨어지지만, **재구성 가능**해서 다른 알고리즘으로 바꿀 수 있음.

##### DPU 자원 비교

```
KV260의 DPU 옵션 (B = batch size):
- B512:  ~10K LUT
- B1024: ~20K LUT
- B2304: ~30K LUT
- B4096: ~50K LUT  (가장 큰 거, ZU5EV의 1/2 차지)

→ 비트스트림 바꿔서 DPU 크기 조절 가능
→ 우리 망막 VPU IP도 같이 넣을 수 있음
```

#### 정리

| | CPU | GPU | FPGA | ASIC |
|--|-----|-----|------|------|
| 유연성 | ★★★★★ | ★★★★ | ★★★ (재합성) | ★ (고정) |
| 효율 | ★ | ★★ | ★★★ | ★★★★★ |
| 개발 비용 | 0 | 0 | 1주~ | 수억원~ |
| 단가 (대량) | 중 | 중-비쌈 | 비쌈 | 매우 쌈 |

**KV260의 DPU = FPGA의 강점(재구성) + ASIC 같은 효율의 절충안**.
우리 프로젝트도 같은 길:
1. FPGA로 prototype + 알고리즘 확정 (지금)
2. 알고리즘 안정되면 ASIC tape-out (장기)

---

## 7. PS와 PL 구분

### Q
> PS와 PL 구분 정확히 안되

### A

OK 핵심 개념. 한 번에 정리.

#### 한 줄 정의

```
PS (Processing System)     = ARM CPU 시스템 (변경 불가, Linux 돌아감)
PL (Programmable Logic)    = FPGA fabric (비트스트림으로 회로 매번 변경)
```

→ **같은 칩 안에 두 영역이 있음**. KV260의 ZU5EV가 그렇게 생긴 칩.

#### 칩 내부 그림

```
┌─────────────────────────────────────────────────┐
│         Zynq UltraScale+ ZU5EV (단일 칩)        │
│                                                  │
│  ┌─────────────────────────────────────────┐   │
│  │ PS (Processing System)                   │   │
│  │ ─────────────────────                    │   │
│  │  • ARM Cortex-A53 ×4 (CPU)               │   │
│  │  • DDR Controller (외부 DDR 메모리 관리) │   │
│  │  • USB, Ethernet, SD, UART (MIO 핀)      │   │
│  │  • I2C, SPI (MIO 핀)                     │   │
│  │  • Linux Ubuntu 돌아감                    │   │
│  │                                          │   │
│  │  → "변경 못 하는 일반 CPU 부분"          │   │
│  └─────────────────────────────────────────┘   │
│              ▲                                   │
│              │   PS-PL 인터페이스                │
│              │   ───────────────                 │
│              │   AXI 버스 (제어용 M_AXI_HPM0)    │
│              │   AXI 버스 (데이터용 S_AXI_HP0)   │
│              │   클럭 (pl_clk0, pl_clk1)         │
│              │   리셋 (pl_resetn0)               │
│              │   인터럽트 (pl_ps_irq0)            │
│              ▼                                   │
│  ┌─────────────────────────────────────────┐   │
│  │ PL (Programmable Logic / FPGA Fabric)    │   │
│  │ ─────────────────────                    │   │
│  │  • LUT 117K + FF 234K + BRAM 144개       │   │
│  │  • DSP 1248개                            │   │
│  │  • 외부 핀과 직접 연결 (PL IO)            │   │
│  │                                          │   │
│  │  → "비트스트림으로 매번 회로 만드는 부분" │   │
│  │                                          │   │
│  │  [현재 비트스트림이 만든 회로]            │   │
│  │  - MIPI CSI Rx Subsystem                 │   │
│  │  - Frame Buffer Write                    │   │
│  │  - SmartConnect 2개                      │   │
│  │  - Clocking Wizard, Reset, ilconcat      │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

#### 우리 디자인에서 누가 PS이고 누가 PL인가

| 요소 | PS or PL | 역할 |
|------|---------|------|
| **Zynq UltraScale+ MPSoC** (블록) | 사실 PS의 인터페이스 | PS와 PL 연결 통로 |
| ARM이 Ubuntu Linux 돌림 | **PS** | OS 실행 |
| Python/Jupyter 실행 | **PS** | OS 위 Python 인터프리터 |
| DDR 메모리 (4GB) | **PS** | PS controller가 관리 |
| MIPI CSI Rx Subsystem | **PL** | 비트스트림으로 만든 회로 |
| Video Frame Buffer Write | **PL** | 비트스트림 회로 |
| AXI SmartConnect | **PL** | 비트스트림 회로 |
| Clocking Wizard | **PL** | 비트스트림이 PLL/MMCM 활성화 |
| Processor System Reset | **PL** | 리셋 동기화 회로 |

→ 우리가 Vivado로 그린 거는 **다 PL에 들어감**. PS는 "이미 있는 거" 활용.

#### KV260 보드의 외부 핀과 연결

```
┌─────────────────────┐
│   KV260 보드 (외부)  │
│                      │
│ [Ethernet] ──────────┼──▶ PS (MIO 핀)
│ [USB]      ──────────┼──▶ PS (MIO 핀)
│ [DisplayPort] ───────┼──▶ PS (DP controller)
│ [SD card]  ──────────┼──▶ PS (MIO 핀)
│ [UART]     ──────────┼──▶ PS (MIO 핀)
│                      │
│ [IAS 카메라 MIPI] ───┼──▶ PL (fabric IO)  ★
│ [IAS 카메라 I2C]  ───┼──▶ PS (MIO 핀)    ★
│ [PMOD]     ──────────┼──▶ PL (fabric IO)
└─────────────────────┘
```

**카메라 인터페이스 분리** ★:
- **MIPI 데이터 라인** → PL의 MIPI CSI Rx로 (고속 데이터)
- **I2C 제어 라인** → PS의 MIO로 (저속 설정)

→ 그래서 우리가 PL에 만들었던 axi_iic IP는 사실 미사용. PS의 MIO I2C가 카메라 init 처리.

#### 한 표로 정리

| | PS | PL |
|--|----|----|
| **풀 네임** | Processing System | Programmable Logic |
| **본질** | ARM CPU 시스템 (고정) | FPGA fabric (재구성) |
| **변경 가능?** | ❌ 칩 만들 때 fix | ✅ 비트스트림마다 변경 |
| **OS** | ✅ Linux 돌아감 | ❌ (회로일 뿐) |
| **메모리** | DDR (큰 외부 RAM) | BRAM (작은 내부) |
| **외부 핀 (KV260)** | MIO: Ethernet, USB, DP, SD | PL IO: MIPI, PMOD |
| **우리 디자인** | Python 실행, 카메라 init | MIPI Rx, Frame Buf |
| **속도** | 1.5GHz (CPU 클럭) | 100~500MHz (회로 클럭) |
| **유연성** | SW 변경 (Python, C) | HW 변경 (비트스트림) |

---

## 8. AXI 프로토콜 (Lite / Stream / Full)

### Q
> AXI 프로토콜 (Stream/Lite/Full 차이, TDATA/TLAST/TUSER)

### A

#### 정체

**AXI** = Advanced eXtensible Interface
ARM이 만든 **on-chip 통신 표준**. CPU↔메모리, IP↔IP 사이 데이터/제어 주고받는 약속.

→ 우리 디자인 안 모든 IP들이 **AXI로 대화**.

#### 3가지 타입

| 타입 | 용도 | 주소? | 단위 | 우리 디자인에서 |
|------|------|------|------|---------------|
| **AXI4 (Full)** | 큰 메모리 접근 | ✅ | Burst (여러 데이터) | Frame Buf → DDR |
| **AXI4-Lite** | IP 제어 레지스터 | ✅ | Single (한 번에 1개) | PS → IP들 (M_AXI_HPM0) |
| **AXI4-Stream** | 연속 데이터 흐름 | ❌ | Stream | MIPI Rx → Frame Buf |

#### 비유

```
AXI4 (Full):
  등기우편 + 트럭 한 대분 화물
  주소: "광주광역시 ○○구 1-1-1번지에 100개 박스 보내"
  burst 지원: 한 번에 여러 데이터

AXI4-Lite:
  편지 한 통
  주소: "○○ 레지스터에 값 0x1234 써"
  단순, 한 번에 한 건

AXI4-Stream:
  공장 컨베이어벨트
  주소 없음. "그냥 다음 픽셀 흘려보내"
  매 사이클 1개씩 흐름
```

#### 신호들 (T-prefix)

##### 공통 (모든 AXI4-Stream에 있음)
| 신호 | 역할 |
|------|------|
| **TDATA** | 실제 데이터 (8/16/32/64 bit 등) |
| **TVALID** | 보내는 쪽: "이 데이터 유효해" |
| **TREADY** | 받는 쪽: "받을 준비 됐어" |

##### 옵션 (필요시)
| 신호 | 역할 |
|------|------|
| **TLAST** | "이게 마지막 데이터" (프레임 끝, 패킷 끝) |
| **TUSER** | 사용자 정의 (보통 SOF = Start of Frame) |
| **TKEEP** | 바이트별 유효성 (TDATA의 어느 byte 유효한지) |
| **TSTRB** | TKEEP과 유사 (요즘은 TKEEP만 씀) |
| **TID** | 채널 ID (한 stream에 여러 흐름 구분) |
| **TDEST** | 목적지 ID |

#### 핵심 메커니즘: VALID/READY Handshake

데이터 전송 = **양쪽이 다 준비됐을 때만** 가능:

```
Clock:    ▁▔▁▔▁▔▁▔▁▔
TVALID:   ▁▁▔▔▔▔▁▁▁▁     보내는쪽 "준비됨"
TREADY:   ▁▁▁▔▔▁▁▁▁▁     받는쪽   "준비됨"
                ↑↑
              여기만 실제 전송
              (둘 다 high인 사이클)
```

→ 받는 쪽 안 받으면 보내는 쪽이 자동 대기. **back-pressure** 자동 처리.

비유:
```
보내는 사람: "줄까?"  (TVALID = 1)
받는 사람:   "응 줘" (TREADY = 1)
→ 둘 다 OK일 때만 데이터 이동

만약:
보내는 사람: "줄까?"  (TVALID = 1)
받는 사람:   "잠깐"   (TREADY = 0)
→ 보내는 사람 대기. 받는 사람 준비될 때까지.
```

이거 덕분에 **속도 다른 IP끼리 자동 동기화**됨. 우리가 별도 처리 안 해도 됨.

#### 우리 디자인 매핑

##### AXI4-Lite (제어 경로)

```
[PS의 ARM]
  ↓ M_AXI_HPM0_FPD (AXI4-Lite, 100MHz)
[SmartConnect_0]
  ├─ M00_AXI ──▶ MIPI CSI Rx의 csirxss_s_axi  (제어 레지스터)
  └─ M01_AXI ──▶ Frame Buf Write의 s_axi_CTRL (제어 레지스터)
```

PS가 Python으로:
```python
overlay.v_frmbuf_wr_0.register_map.AP_CTRL = 1  # IP 시작 명령
```
이게 AXI4-Lite로 가는 단순 명령.

##### AXI4-Stream (비디오 데이터 경로)

```
[MIPI CSI Rx] ─video_out (AXI4-Stream, 200MHz)
   TDATA: 16bit (UYVY 픽셀)
   TVALID/TREADY: handshake
   TLAST: 라인 끝 표시
   TUSER: SOF (프레임 시작)
   ↓
[Subset Converter]
   TDATA: 16bit → 24bit 변환 (zero padding)
   ↓
[Frame Buf Write] s_axis_video
```

매 사이클 픽셀 1개씩 흐름. 1080p@60 = 약 124M 픽셀/초 = 200MHz의 ~62%.

##### AXI4 Full (메모리 경로)

```
[Frame Buf Write] m_axi_mm_video (AXI4 Full, 200MHz)
   주소: DDR의 어느 위치에 쓸지 (예: 0x40000000)
   Burst: 16개 데이터 한 번에 (효율 ↑)
   ↓
[SmartConnect_1]
   ↓
[PS] S_AXI_HP0_FPD ──▶ DDR Controller ──▶ DDR4 메모리
```

Frame Buf가 매 1080×1920×2byte = 4MB 프레임을 DDR에 burst write.

#### 한 그림 정리

```
[Python on PS]
   ↓ AXI4-Lite (작은 명령)
[SmartConnect_0]
   ↓ ↓
[MIPI Rx] ────AXI4-Stream───▶ [Subset Conv] ────▶ [Frame Buf Write]
                                                          │
                                                          ↓ AXI4 Full (큰 데이터)
                                                  [SmartConnect_1]
                                                          ↓
                                                  [PS의 DDR]
```

**색깔 분류**:
- AXI4-Lite = 제어 (작고 가끔)
- AXI4-Stream = 데이터 (연속 흐름)
- AXI4 Full = 메모리 접근 (대용량 burst)

#### 핵심 인사이트

```
1. 모든 IP는 AXI로 대화한다 (표준이라 호환)
2. 인터페이스만 맞추면 IP 조합 자유
   - MIPI Rx의 video_out (AXI-Stream)
   - Subset Converter의 S_AXIS (AXI-Stream)
   → 그냥 + 끼리 연결 가능
3. VALID/READY 자동 동기화로 속도 차이 처리
4. SmartConnect는 AXI 라우팅 + 변환 자동
```

#### 일반 컴퓨터와 비교

```
일반 PC:
  CPU ↔ RAM = DDR4 protocol (전용)
  CPU ↔ GPU = PCIe (전용)
  CPU ↔ USB = USB protocol (전용)
  → 각자 다른 표준

Zynq (KV260):
  PS ↔ PL = AXI (단일 표준)
  PL 안 IP ↔ IP = AXI
  PL ↔ DDR (PS 통해) = AXI
  → 한 표준으로 통일 (간결)
```

이게 Zynq + AXI 조합의 강점. **한 칩 안에서 통일된 통신 표준** → 디자인 단순.

---

## 9. MIPI CSI-2 + AP1302

### Q
> MIPI ㄱㄱ

### A

#### 큰 그림

```
[빛] ──▶ [AR1335 센서] ──▶ [AP1302 ISP] ──MIPI CSI-2──▶ [KV260 PL]
         (13MP 베이어)    (외부 ISP 칩)   (고속 직렬)    (FPGA fabric)
```

**3단계로 분리됨**:
1. **센서** = 빛 → 베이어 RAW (전기 신호)
2. **ISP** = 베이어 RAW → YUV/RGB (이미 가공된 영상)
3. **MIPI CSI-2** = ISP 출력을 KV260로 전송하는 통신 표준

### Part 1: MIPI CSI-2 (통신 프로토콜)

#### 정체

**MIPI CSI-2** = Camera Serial Interface 2 (MIPI Alliance 표준)

스마트폰, 임베디드 비전 카메라의 **사실상 표준**. 모든 폰 카메라가 이거 씀.

#### 물리 구조 (D-PHY)

```
MIPI 4-lane 카메라 ↔ KV260 연결:

CLK_P / CLK_N  ━━━━━━━━━━━━━ (차분 클럭, 1쌍)
DATA0_P / DATA0_N ━━━━━━━━━━ (차분 데이터, 1쌍 = 1 lane)
DATA1_P / DATA1_N ━━━━━━━━━━
DATA2_P / DATA2_N ━━━━━━━━━━
DATA3_P / DATA3_N ━━━━━━━━━━

총 5쌍 = 10 핀 (D-PHY)
```

**핵심 개념**:

| 용어 | 의미 |
|------|------|
| **D-PHY** | 물리 계층 (전기 신호 표준, 차분 신호) |
| **Lane** | 데이터 차선 (보통 2 또는 4) |
| **Clock lane** | 1개 (모든 data lane이 따름) |
| **Lane rate** | 한 lane의 속도 (Mbps/lane) |

#### Lane Rate 계산

우리 디자인: **800 Mbps/lane × 4 lanes = 3.2 Gbps total**

필요 대역폭 (1080p@60, YUV422 8bit):
```
1920 × 1080 × 60 × 16bit = 1.99 Gbps
```

→ 4 lane × 800Mbps = 3.2 Gbps 중 ~62% 사용. **여유 있음** ✅.

만약 4K@30 가려면:
```
3840 × 2160 × 30 × 16bit = 3.98 Gbps → 4 lane × 1Gbps 필요 (Line rate up)
```

#### 비유

```
1-lane MIPI = 도로 1차선 (속도 한 대분만)
4-lane MIPI = 도로 4차선 (4배 빠름, 같은 시간에 4배 데이터)
Clock lane = 신호등 (모든 차선 동기화)
```

#### 패킷 구조 (간략)

```
MIPI CSI-2 데이터 흐름:

[Frame Start] ─▶ [Line 1 픽셀들] ─▶ [Line 2] ─▶ ... ─▶ [Frame End]

Short Packet (4 byte): 동기 신호 (FS, FE)
Long Packet: 실제 픽셀 데이터 (수십 KB per line)
```

→ MIPI Rx Subsystem이 알아서 처리. 우리는 **AXI-Stream으로 변환된 출력만** 받음:
```
TDATA: 픽셀 데이터 (16bit, UYVY)
TVALID/TREADY: 데이터 핸드쉐이크
TLAST: 라인 끝 표시
TUSER: SOF (프레임 시작)
```

### Part 2: AP1302 (외부 ISP 칩)

#### 정체

**AP1302** = ON Semiconductor의 **이미지 신호 처리 전용 칩**.

```
[AR1335 센서: 베이어 RAW]
   ↓ (sensor interface)
[AP1302 ISP 칩: 처리]
   ↓ (MIPI 출력)
[KV260로 가공된 영상]
```

**카메라 모듈 안에 박힌 작은 검은 칩**.

#### AP1302가 하는 일

```
입력: AR1335 센서의 베이어 RAW10 (가공 안 된 raw)
   ↓
1. Black Level Correction
2. Lens Shading Correction
3. Demosaic (Bayer → RGB)        ← 핵심
4. Auto White Balance (AWB)
5. Auto Exposure (AE)
6. Color Correction Matrix
7. Gamma correction
8. Noise reduction
9. Sharpening
10. (HDR 지원 시 톤매핑)
   ↓
출력: 가공된 영상 (YUV/RGB, 사용자 선택)
```

→ **DSLR 카메라 안의 ISP가 하는 일을 칩 하나로**.

#### 출력 형식 (I2C로 설정)

| 옵션 | 픽셀당 비트 | 용도 |
|------|----------|------|
| YUV422 8bit | 16bit | 비디오 표준 ⭐ 우리 선택 |
| YUV422 10bit | 20bit | HDR 영상 |
| YUV420 8bit | 12bit | 압축 효율 (영상 인코딩) |
| RGB888 | 24bit | 일반 RGB |
| RGB565 | 16bit | 압축 RGB |
| Mono | 8bit | 흑백 |
| RAW Bayer | 8/10bit | ISP bypass (실험적) |

#### 제어 인터페이스 = I2C

```
KV260의 PS Linux ──I2C──▶ AP1302
   "출력 형식: YUV422"
   "해상도: 1920×1080"
   "프레임레이트: 60fps"
   "노출: auto"
   ...
   (수십~수백 개 레지스터)
```

→ AP1302는 부팅 시 **firmware 파일 로드 필요** (수백 KB). PPA 막혀서 못 받은 그 firmware.

#### 제어 흐름

```
1. KV260 전원 ON
2. Linux 부팅
3. ap1302 driver (또는 사용자 코드) 가:
   a. AP1302의 firmware 다운로드 (I2C로 한 줄씩)
   b. AP1302 부팅 완료 대기
   c. 출력 모드 설정 (YUV422, 1080p, etc.)
4. AP1302가 AR1335 init (sensor에 명령 전달)
5. AR1335가 영상 캡쳐 시작 → AP1302가 처리 → MIPI로 출력
6. KV260의 PL이 MIPI 받음 → Frame Buf → DDR
7. PYNQ Python이 DDR 읽어서 표시
```

### Part 3: 왜 우리가 YUV422 8bit 골랐나

#### 6가지 이유

##### 1. AP1302 표준 출력 모드
- 가장 호환성 좋음, 기본값
- 다른 모드는 추가 설정 필요

##### 2. 망막 opponent와 거의 일치 ⭐
```
YUV422의 정의:
Y = 0.299R + 0.587G + 0.114B   ← luminance (망막 Parasol/Magno)
U = B - Y                         ← B-Y opponent (망막 Konio)
V = R - Y                         ← R-Y opponent (망막 Midget)
```

→ **YUV가 우연히 망막 opponent와 같음**. 1950년대 컬러 TV 만들 때 인간 시각 흉내냈으니 자연스러움.

→ AP1302가 이미 망막의 1차 처리(opponent) 끝낸 상태로 보내주는 셈.

##### 3. 대역폭 효율
```
YUV422 8bit:  16 bit/픽셀
RGB888:       24 bit/픽셀  → 50% 더 큼
YUV422 10bit: 20 bit/픽셀  → 25% 더 큼
```

MIPI 대역폭 절약 + Frame Buf 메모리 절약.

##### 4. 8bit 충분
- 사람 눈 동적 범위 약 ~8bit
- 망막 처리에 더 큰 정밀도 필요 X
- 10bit는 HDR 영상이나 정밀 컬러그레이딩 필요할 때만

##### 5. AXI-Stream TDATA 폭이 깔끔
- 16bit = 8bit × 2 (Y + U/V)
- AXI 디자인에서 처리 단순

##### 6. Frame Buffer Write IP가 안정적 지원
- Vitis HLS-generated IP
- UYVY8 모드 검증됨

#### 우리가 안 고른 형식

| 형식 | 안 고른 이유 |
|------|-------------|
| YUV420 | U/V 다운샘플 → opponent 정보 손실 |
| RGB888 | 24bit 너무 큼 + opponent로 다시 변환 필요 |
| RAW Bayer | AP1302 bypass 필요 (실험적, 비공식) |

### 핵심 인사이트

```
1. MIPI = "고속 카메라 케이블 표준" (4 lane × 수백 Mbps)
   → 우리는 800Mbps × 4 = 3.2Gbps. 1080p@60 충분.

2. AP1302 = "외부 ISP 칩"
   → 우리는 가공된 YUV422 받음. demosaic 등 다 끝남.

3. YUV422 = 망막 opponent와 자연스럽게 일치
   → AP1302가 이미 망막 1차 처리한 셈

4. PL의 MIPI CSI Rx Subsystem = MIPI → AXI-Stream 변환기
   → 우리는 AXI 영역에서만 작업
```

#### 망막 VPU 관점에서

```
[광수용기] = AR1335 센서
  ↓ 베이어 RAW
[수평/양극/아마크린] = AP1302 ISP
  ↓ YUV (이미 opponent)
[신경절세포] = 우리 망막 VPU IP (만들 거)
  ↓ 22채널 retinal output
[시상/시각피질] = AI 모델
```

→ 사실 **AP1302가 이미 망막의 1차 처리를 흉내낸 셈**. 우리 VPU는 그 위에 추가 처리 (edge, motion, temporal 등).

---

## 10. PYNQ가 파이썬 라이브러리인가?

### Q
> pYNQ가 파이썬 라이브러리임?

### A

반은 맞고 반은 아님. **PYNQ = Python 라이브러리 + Linux 이미지 + Jupyter 환경의 패키지**.

#### PYNQ의 정체

```
PYNQ = "Python productivity for Zynq"
   = AMD/Xilinx의 오픈소스 프로젝트
```

**Python 라이브러리 그 자체**는 아니고, 그 위에 더 큰 환경.

#### 구성 요소 4개

##### 1. **`pynq` Python 패키지** (라이브러리 본체)
```python
from pynq import Overlay
from pynq import allocate
from pynq import MMIO
```
`pip install pynq` 또는 PYNQ 이미지에 미리 설치됨. 이게 "PYNQ 라이브러리".

##### 2. **PYNQ Linux 이미지**
- Ubuntu 기반 (KV260: Ubuntu 22.04)
- 부팅용 SD 이미지
- 우리가 굽었던 `.img` 파일

##### 3. **Jupyter Lab 환경**
- 부팅 시 자동 실행 (`http://kria:9090/lab`)
- 미리 들어있는 예제 노트북들

##### 4. **Driver 라이브러리**
- 일반적인 IP들의 wrapper (AXI DMA, Video Frame Buffer 등)
- `pynq.lib`, `pynq.lib.video` 등 서브 모듈

#### 한 그림

```
┌─────────────────────────────────────┐
│ "PYNQ 환경"                         │
│                                      │
│  ┌──────────────────────────────┐   │
│  │ pynq Python 패키지            │   │  ← 코어 라이브러리
│  │  - Overlay, MMIO, allocate... │   │
│  │  - pynq.lib.video, .dma...    │   │
│  └──────────────────────────────┘   │
│           ▲                          │
│           │                          │
│  ┌────────┴───────────────────────┐ │
│  │ Jupyter Lab (Python 인터페이스) │ │  ← 사용자 접점
│  └────────────────────────────────┘ │
│           ▲                          │
│           │                          │
│  ┌────────┴───────────────────────┐ │
│  │ Ubuntu Linux (미리 셋업)         │ │  ← OS
│  └────────────────────────────────┘ │
│           ▲                          │
│           │                          │
│  ┌────────┴───────────────────────┐ │
│  │ KV260 (PS + PL)                  │ │  ← 하드웨어
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

#### 비교

```
NumPy = "Python 라이브러리" (pure)
PYNQ = "라이브러리 + OS 환경 + Jupyter 통합" (플랫폼)
```

#### 왜 그렇게 묶어서 배포?

PL 제어하려면:
- Linux 커널의 `/dev/mem`, `fpga_manager` driver 필요
- Python 라이브러리 + 커널 권한 + 부팅 환경 셋업이 까다로움

→ **이걸 한 번에 묶어서 SD 이미지로 배포** = "넣고 부팅하면 끝"

#### 우리가 한 일을 다시 보면

```
1. Ubuntu 22.04 for Kria 굽기
   → OS 베이스 (PYNQ 아님, AMD 일반 Ubuntu)

2. install.sh 실행
   → pynq 패키지 + Jupyter + 의존성 자동 설치
   → "Ubuntu" → "Ubuntu + PYNQ 환경" 으로 변환

3. Jupyter에서 Overlay() 사용
   → pynq Python 라이브러리 호출
```

#### pip로 설치 가능?

기술적으론 가능:
```bash
pip install pynq
```

근데:
- Linux 커널 driver, 부팅 설정, 메모리 권한 등 셋업이 추가로 필요
- 일반 PC에선 동작 안 함 (FPGA 없음)
- KV260 같은 보드 + 호환 OS 환경 필요

→ 그래서 보통은 **PYNQ 통합 이미지** 또는 **Ubuntu + install.sh** 방식.

#### 한 줄 정리

```
"PYNQ" = "Python으로 Zynq FPGA 제어하는 통합 환경"
       = pynq 라이브러리 + Linux + Jupyter + driver
```

좁은 의미로는 `pynq` Python 라이브러리. 넓은 의미로는 그 전체 환경.

---

## 11. Overlay 정확한 동작 (6단계)

### Q
> Overlay를 실행하면 펌웨어쪽에서 bit 파일을 인식해서 pl이 움직여서 ip를 대체한다는 건 이해했어, 근데 이것만 이해했고 구체적인 동작은 아직 몰라, 지금 실행 시의 정확한 동작에 대해서 한번 공부해보자

### A

#### `overlay = Overlay("design_1.bit")` 분해

##### 큰 그림 (단계별)

```
1. .bit 파일 읽기 (디스크 → 메모리)
2. .hwh 파일 자동 찾기 + XML 파싱
3. fpga_manager에 비트스트림 전달 → PL 재프로그램
4. PL의 IP 주소 맵을 PS 메모리 공간에 매핑 (mmap)
5. 각 IP에 대해 Python 객체 생성
6. Driver 매칭 (있으면 wrapper, 없으면 generic)
```

각각 풀어보자.

#### Step 1: .bit 파일 읽기

```python
overlay = Overlay("design_1.bit")
```

PYNQ가 내부적으로:
```python
with open("design_1.bit", "rb") as f:
    bitstream_bytes = f.read()  # ~3MB 바이너리
```

→ 그냥 디스크에서 메모리로 읽음. 이 단계는 빠름 (수 ms).

`.bit` 파일 내용:
- 헤더 (디자인 이름, 시간, FPGA 부품 정보)
- LUT 진리표
- FF 초기값
- 배선 스위치 설정
- 클럭/IO 설정
- ECC, CRC

#### Step 2: .hwh 파일 파싱

같은 이름의 `.hwh` 자동 찾기:
```
design_1.bit  → design_1.hwh
```

`.hwh`는 **XML 파일**. 디자인의 모든 IP 정보 들어있음:

```xml
<MODULE FULLNAME="design_1_i/v_frmbuf_wr_0"
        VLNV="xilinx.com:hls:Frmbuf_wr:2.5"
        BASEADDR="0x00A0000000">
  <REGISTERS>
    <REGISTER NAME="AP_CTRL"  OFFSET="0x000"/>
    <REGISTER NAME="GIE"      OFFSET="0x004"/>
    <REGISTER NAME="WIDTH"    OFFSET="0x010"/>
    <REGISTER NAME="HEIGHT"   OFFSET="0x018"/>
    <REGISTER NAME="STRIDE"   OFFSET="0x020"/>
    <REGISTER NAME="VIDEO_FORMAT" OFFSET="0x028"/>
    <REGISTER NAME="POINTER"  OFFSET="0x030"/>
    ...
  </REGISTERS>
</MODULE>

<MODULE FULLNAME="design_1_i/mipi_csi2_rx_subsyst_0"
        VLNV="xilinx.com:ip:mipi_csi2_rx_subsystem:6.0"
        BASEADDR="0x00A0010000">
  ...
</MODULE>
```

PYNQ가 이걸 파싱해서:
- 어떤 IP가 있는지
- 각 IP의 메모리 주소 (BASEADDR)
- 각 IP의 레지스터 offset
- IP 종류 (VLNV)

→ Python dictionary로 정리:
```python
ip_dict = {
    "v_frmbuf_wr_0": {
        "phys_addr": 0xA0000000,
        "addr_range": 0x10000,
        "type": "xilinx.com:hls:Frmbuf_wr:2.5",
        "registers": {...}
    },
    "mipi_csi2_rx_subsyst_0": {...},
    ...
}
```

이 단계 좀 무거움 (XML 파싱 = CPU 부하).

#### Step 3: PL 재프로그램 (★ 진짜 마법)

PYNQ가 Linux의 `fpga_manager`에 비트스트림 보냄:

```python
# 내부적으로
with open("/sys/class/fpga_manager/fpga0/state", "w") as f:
    f.write("write")

with open("/dev/fpga0", "wb") as f:
    f.write(bitstream_bytes)
```

`/sys/class/fpga_manager/fpga0/`는 Linux kernel의 FPGA Manager driver 인터페이스.

내부 동작:
```
1. fpga_manager driver가 비트스트림 받음
2. PS의 PCAP (Processor Configuration Access Port) 활성화
3. PCAP를 통해 PL의 configuration memory에 비트스트림 쓰기
   (DMA 사용해서 빠르게)
4. PL fabric의 LUT 진리표, FF 초기값, 배선 스위치 모두 재설정
5. 새 회로 활성화 신호 (configuration done)
```

dmesg에서 봤던 그 줄:
```
fpga_manager fpga0: writing design_1.bin to Xilinx ZynqMP FPGA Manager
[drm] bitstream c1c2f176-... locked, ref=1
```

이게 진행 중일 때 사용자가 본 **CPU/팬 부하 증가**의 원인:
- DMA 전송 부하
- PL 전체가 새 회로로 변경되며 transient power surge

이 단계가 **수백 ms ~ 1초** 걸림.

#### Step 4: Memory Map 설정 (mmap)

이제 PL은 새 회로지만, **PS에서 어떻게 접근?**

답: **AXI4-Lite를 PS의 주소 공간에 매핑**.

```
PS의 주소 공간 (64bit):
┌──────────────────────────────────┐
│ 0x0000000000 ~                   │
│   DDR 메모리 (4GB)                │
│ 0x00FFFFFFFF                     │
├──────────────────────────────────┤
│ 0xA000000000 ~                   │
│   PL의 v_frmbuf_wr_0 (64KB)      │  ← .hwh의 BASEADDR
├──────────────────────────────────┤
│ 0xA000010000 ~                   │
│   PL의 mipi_csi2_rx_subsyst_0    │
├──────────────────────────────────┤
│ 0xA000020000 ~                   │
│   PL의 axis_subset_converter_0   │
└──────────────────────────────────┘
```

PYNQ가 Linux의 mmap() 사용:
```python
import mmap
import os

# /dev/mem 열기 (커널 메모리 직접 접근, root 권한)
mem_fd = os.open("/dev/mem", os.O_RDWR | os.O_SYNC)

# v_frmbuf_wr_0의 물리 주소를 가상 주소로 매핑
v_frmbuf_mmap = mmap.mmap(
    mem_fd,
    length=0x10000,        # 64KB
    offset=0xA0000000      # 물리 주소
)

# 이제 v_frmbuf_mmap[0]에 쓰면 → 실제로 AXI 트랜잭션 발생
```

매핑 후:
- Python이 `v_frmbuf_mmap`의 어떤 byte에 쓰기 → MMU 변환 → AXI4-Lite write → SmartConnect → PL의 IP

→ **Python으로 PL 회로의 레지스터에 직접 접근 가능**.

#### Step 5: Python 객체 생성

각 IP에 대해 Python wrapper class 인스턴스 생성:

```python
overlay.v_frmbuf_wr_0       # → Python 객체
overlay.mipi_csi2_rx_subsyst_0  # → Python 객체
overlay.axis_subset_converter_0  # → Python 객체
```

이 객체들은:
- `.register_map`: 레지스터 직접 접근
- `.read(offset)`: 특정 offset 읽기
- `.write(offset, value)`: 특정 offset 쓰기
- `.mmio`: 저수준 메모리 매핑 객체

예:
```python
overlay.v_frmbuf_wr_0.read(0x000)   # AP_CTRL 레지스터 읽기
overlay.v_frmbuf_wr_0.write(0x010, 1920)  # WIDTH 레지스터에 1920 쓰기

# 또는 더 편하게:
overlay.v_frmbuf_wr_0.register_map.WIDTH = 1920
overlay.v_frmbuf_wr_0.register_map.HEIGHT = 1080
overlay.v_frmbuf_wr_0.register_map.AP_CTRL = 1   # 시작!
```

#### Step 6: Driver 매칭 (선택)

PYNQ는 **알려진 IP 종류에 대해 wrapper class** 가지고 있음:
```
xilinx.com:ip:axi_dma:7.1            → AxiDMA wrapper
xilinx.com:hls:Frmbuf_wr:2.x         → FrmbufWr wrapper
xilinx.com:ip:mipi_csi2_rx_subsystem:5.1 → MipiCsi2Rx wrapper (있다면)
```

매칭되면 편한 API 사용 가능:
```python
overlay.v_frmbuf_wr_0.write_frame(numpy_array)  # Frmbuf wrapper의 메서드
```

매칭 안 되면 (우리 케이스):
```
UserWarning: IP type 6.0, driver found for 5.1
```
→ 그냥 generic MMIO 객체로 fallback. **레지스터 직접 접근**은 가능.

`ignore_version=True` 옵션 = "버전 다르더라도 wrapper 강제 사용".

#### 실제 흐름 예시

```python
overlay = Overlay("design_1.bit", ignore_version=True)
overlay.v_frmbuf_wr_0.register_map.WIDTH = 1920
```

이 한 줄에서 일어나는 일:

```
[Python 인터프리터]
   ↓ register_map.WIDTH = 1920 설정자 호출
[PYNQ Python lib]
   ↓ "WIDTH의 offset = 0x010"
   ↓ "v_frmbuf_wr_0의 BASEADDR = 0xA0000000"
   ↓ "0xA0000010 위치에 1920(0x780) 쓰기"
[Linux mmap → /dev/mem]
   ↓ MMU 가상→물리 주소 변환
[PS의 ARM CPU]
   ↓ store instruction (AXI4-Lite write)
[AXI Interconnect (PS→PL 게이트웨이)]
   ↓ AXI 트랜잭션 (write addr=0xA0000010, data=0x780)
[smartconnect_0]
   ↓ M01_AXI 라우팅
[v_frmbuf_wr_0의 AXI4-Lite slave 인터페이스]
   ↓ decode address
[v_frmbuf_wr_0 내부 register file]
   ↓ WIDTH 레지스터 (FF로 만들어진)
   ↓ 0x780 값 저장됨
```

→ Python 한 줄이 **8개 계층의 하드웨어 통신**을 트리거. 모든 게 자동.

#### 비유로 정리

```
"Overlay 로드" =
  1. 청사진 읽기      (.bit 파싱)
  2. 부품 카탈로그 보기 (.hwh 파싱)
  3. 공장에 설계도 보내기 (PL 재프로그램)
  4. 공장 위치 등록     (mmap)
  5. 원격 조종 패널 만들기 (Python 객체)
  6. 매뉴얼 매칭         (driver)
```

```
"register_map.WIDTH = 1920" =
  원격 조종 패널의 다이얼 돌리기
  → 무선 신호 전송 (AXI)
  → 공장의 설정값 변경
  → 공장 동작 변경
```

#### 정리

```
.bit  = "PL이 어떤 회로 될지" 정의 (LUT/FF/배선 비트들)
.hwh  = "그 회로 안 IP 정보" 정의 (XML, 주소 맵)
Overlay() = 위 두 파일 + Linux + Python을 묶어서 추상화
```

PYNQ의 진짜 가치:
```
원래라면:
  - C로 mmap() 호출
  - 직접 주소 계산
  - register offset 외움
  - 드라이버 작성
  ...

PYNQ로:
  overlay.ip.register_map.X = Y  # 한 줄
```

→ **Python 한 줄로 PL 하드웨어 직접 제어**가 진짜 강점.

---

## 12. 학습 진척 정리

### Q
> 이제 workstate 참조해서 뭐 더 공부해야하는지 다시 확인해줘

### A

#### ✅ 이번 세션에 끝낸 것

```
[KV260 / Kria]
✅ PS와 PL 분리 구조
✅ MIO vs PL IO (카메라 MIPI=PL, I2C=PS 분리도 이해)

[Vivado / Block Design]
✅ AXI 프로토콜 (Lite/Stream/Full + TDATA/TLAST/TUSER + handshake)
✅ Clock Domain Crossing (CDC)

[MIPI CSI-2]
✅ D-PHY, Lane, Lane rate
✅ Pixel format vs Line rate

[YUV vs RGB]
✅ YUV422 vs YUV420 vs YUV444
✅ UYVY vs YUYV
✅ AP1302 ISP 칩

[PYNQ]
✅ Overlay 동작 메커니즘 (6단계)
✅ overlay.ip_dict, register_map
✅ mmap, AXI4-Lite, fpga_manager

[보너스로 배운 것]
✅ ASIC vs FPGA 트레이드오프
✅ LUT/FF/배선 세부 역할
✅ DPU on FPGA vs ASIC
```

→ work_state.md §6의 약 **70% 완료**.

#### ⬜ 남은 학습 항목 (4그룹)

##### 🅐 합성 흐름 (★ 가장 중요, 우리가 직접 거친 단계)
- [ ] **Synthesis vs Implementation vs Bitstream**
  - HDL → 게이트 → LUT 배치 → 비트
- [ ] **HDL Wrapper**
  - Block Design을 Verilog로 감싸기
  - 왜 wrapper가 top이어야 하나
- [ ] **Timing closure**
  - WNS, TNS 같은 보고서 숫자
  - 왜 clock 너무 높이면 합성 실패하나

##### 🅑 Device Tree Overlay (.dtbo) (★ 카메라 인식 안 되는 핵심 이유)
- [ ] Linux가 하드웨어 인식하는 방법
- [ ] .dtbo와 .bit의 관계
- [ ] PYNQ가 .hwh로 어떻게 dtbo 대체하는지
- [ ] 우리 디자인에 .dtbo 추가하면 /dev/video* 뜨게 만들 수 있음

##### 🅒 DRC + Methodology
- [ ] **DRC (Design Rule Check)**: NSTD-1, UCIO-1 같은 에러의 의미
- [ ] **Methodology Violations**: 우리가 본 TIMING-4, TIMING-27 경고
- [ ] **Critical Warning vs Error**의 차이

##### 🅓 Block Automation vs Connection Automation
- [ ] 이번에 자주 마주친 두 자동화 도구
- [ ] 언제 사용하나
- [ ] 잘못 잡힐 때 처리

##### 🅔 xlnx-config / xmutil (간단)
- [ ] Kria 보드 firmware 관리 도구
- [ ] xmutil listapps, loadapp의 의미
- [ ] 이게 PYNQ Overlay와 어떤 관계

#### 📌 추천 다음 순서

**중요도 + 우리 프로젝트 관련성 순**:

```
1. 🅐 합성 흐름 (Synth/Impl/Bit + Timing)
   → 우리가 30분 걸려서 만든 그 과정 정확히 이해
   → 다음 망막 VPU IP 만들 때 timing 디버깅 가능해짐

2. 🅑 Device Tree Overlay
   → "왜 /dev/video* 안 뜨나"의 근본 원인
   → 카메라 동작시키려면 결국 이걸 작성해야 함
   → 한 번 이해하면 카메라 동작 길 열림

3. 🅒 DRC + Methodology
   → 우리가 만났던 에러들 (IIC LOC, TIMING) 본질
   → 다음 디자인에서 미리 방지

4. 🅓 Block/Connection Automation (가벼움)
   → 빠르게 정리

5. 🅔 xlnx-config (가벼움)
   → 옵션, 시간 있으면
```

#### 다음 단계 옵션

##### A. 학습 계속 — 🅐 합성 흐름부터
Synthesis → Implementation → Bitstream의 정확한 동작 + Timing closure 같은 것까지.

##### B. 실전으로 — Next-A: PL 동작 검증
```python
print(overlay.v_frmbuf_wr_0.register_map)
overlay.v_frmbuf_wr_0.write(0x10, 1920)
```
→ 실제로 PS↔PL 통신 살아있는지 즉시 확인. 학습한 거 적용.

##### C. 본 연구 진입 — Next-B: 망막 VPU IP 작성 (HLS)
- ChromaExtractor부터 시작
- HLS C++ → IP block → Vivado에 추가
- 이게 사용자 진짜 연구

##### D. 휴식 / 정리

---

_세션 종료 시점: PYNQ Overlay 6단계 동작 + AXI/MIPI/PS-PL/CDC/AP1302 다 이해. 다음은 합성 흐름 (Synth/Impl/Bitstream) 또는 본 연구 진입._
