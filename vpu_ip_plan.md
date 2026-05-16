# 망막 VPU IP 구현 계획

KV260 비트스트림 인프라 완성 후, **본 연구 — Retinal VPU IP** 작성 계획.

---

## 현재 상태와 목표

### Block Design 현재 자리

```
[MIPI Rx] ──▶ [axis_subset_converter_0] ──▶ [Frame Buf Write] ──▶ DDR
                      ↑
              placeholder (16→24bit padding만 함)
              여기에 진짜 VPU IP 들어갈 자리
```

### 목표

```
[MIPI Rx] ──▶ [Retinal VPU IP (HLS)] ──▶ [Frame Buf Write] ──▶ DDR
                      ↑
              우리가 직접 만들 IP
              - 망막 22채널 처리
              - 매 사이클 픽셀 1개
              - HLS C++로 작성
```

---

## 전체 단계 (Phase A~E)

| Phase | 내용 | 산출물 | 예상 시간 |
|-------|------|--------|----------|
| **A** | HLS 환경 + 첫 IP (pass-through) | `vpu_chroma_extractor.zip` | 1주 |
| **B** | Vivado 통합 + 비트스트림 + KV260 검증 | 새 .bit + PYNQ 동작 확인 | 며칠 |
| **C** | 진짜 10채널 VPU (Spatial) | 10채널 망막 VPU IP | 2~3주 |
| **D** | 시간/모션 채널 추가 (12채널) | 22채널 풀 VPU | 2주 |
| **E** | 영상 검증 (시뮬 또는 카메라) | PyTorch와 cross-validation | 1~2주 |

**총 예상 7~10주**.

---

## Phase A: HLS 환경 + 첫 IP

### 목표

> HLS로 IP 만드는 흐름 익히기. 가장 단순한 pass-through IP 동작.

### 단계

#### A-1. Vitis HLS 셋업
1. Vitis HLS 2025.2 실행
2. Workspace 설정: `C:/Users/ejfhr/FPGA/hls_workspace`
3. 새 HLS Component:
   - **Name**: `vpu_chroma_extractor`
   - **Top function**: `chroma_extractor`
   - **Part**: `xck26-sfvc784-2LV-c` (KV260)

#### A-2. 소스 파일 3개 작성

##### `chroma_extractor.h`
```cpp
#ifndef CHROMA_EXTRACTOR_H
#define CHROMA_EXTRACTOR_H

#include <ap_int.h>
#include <hls_stream.h>
#include <ap_axi_sdata.h>

// AXI-Stream 인터페이스 정의
//   16 bits TDATA + 1 bit TUSER + 1 bit TLAST
typedef ap_axiu<16, 1, 1, 1> pixel_t;

void chroma_extractor(
    hls::stream<pixel_t>& in_stream,
    hls::stream<pixel_t>& out_stream
);

#endif
```

##### `chroma_extractor.cpp`
```cpp
#include "chroma_extractor.h"

void chroma_extractor(
    hls::stream<pixel_t>& in_stream,
    hls::stream<pixel_t>& out_stream
) {
    #pragma HLS INTERFACE axis port=in_stream
    #pragma HLS INTERFACE axis port=out_stream
    #pragma HLS INTERFACE s_axilite port=return

    while (true) {
        #pragma HLS PIPELINE II=1

        pixel_t in = in_stream.read();

        // 일단 그대로 통과 (pass-through)
        pixel_t out;
        out.data = in.data;
        out.keep = in.keep;
        out.last = in.last;
        out.user = in.user;
        out_stream.write(out);

        if (in.last) break;
    }
}
```

##### `testbench.cpp`
```cpp
#include "chroma_extractor.h"
#include <iostream>

int main() {
    hls::stream<pixel_t> in_stream;
    hls::stream<pixel_t> out_stream;

    // 테스트 입력: 10픽셀
    for (int i = 0; i < 10; i++) {
        pixel_t p;
        p.data = i * 100;  // 데이터: 0, 100, 200, ..., 900
        p.keep = -1;
        p.user = (i == 0) ? 1 : 0;     // SOF는 첫 픽셀에만
        p.last = (i == 9) ? 1 : 0;     // 마지막 픽셀 표시
        in_stream.write(p);
    }

    // IP 호출
    chroma_extractor(in_stream, out_stream);

    // 출력 확인
    int idx = 0;
    while (!out_stream.empty()) {
        pixel_t p = out_stream.read();
        std::cout << "Out[" << idx++ << "]: data=" << p.data
                  << " user=" << p.user
                  << " last=" << p.last << std::endl;
    }

    return 0;
}
```

#### A-3. C Simulation
- Vitis HLS에서 **Run C Simulation** 클릭
- 기대 출력:
  ```
  Out[0]: data=0   user=1 last=0
  Out[1]: data=100 user=0 last=0
  ...
  Out[9]: data=900 user=0 last=1
  ```
- 입력 = 출력이면 ✅ (pass-through 정상 동작)

#### A-4. C Synthesis
- **Run C Synthesis** 클릭
- 자원 보고서 확인:
  - LUT: 약 100~200개
  - FF: 약 100개
  - 매우 작은 IP
- Latency: ~1 cycle per pixel

#### A-5. IP Export
- **Export RTL** → **Vivado IP** 형식
- 출력: `vpu_chroma_extractor.zip` (Vivado에서 IP Catalog에 추가 가능)

### Phase A 산출물

```
C:/Users/ejfhr/FPGA/hls_workspace/vpu_chroma_extractor/
├─ chroma_extractor.cpp
├─ chroma_extractor.h
├─ testbench.cpp
├─ solution1/
│  ├─ syn/report/...   ← 자원 보고서
│  └─ impl/ip/         ← Export된 IP
```

---

## Phase B: Vivado 통합 + KV260 검증

### 목표

> 만든 IP를 우리 Block Design에 끼워넣고 KV260에서 동작 확인.

### 단계

#### B-1. Vivado IP Catalog에 추가
1. Vivado 프로젝트 (`project_2`) 열기
2. **IP Catalog** 우클릭 → **Add Repository**
3. 경로: `C:/Users/ejfhr/FPGA/hls_workspace/vpu_chroma_extractor/solution1/impl/ip`

#### B-2. Block Design 수정
1. `axis_subset_converter_0` 클릭 → **Delete**
2. **Add IP** → `chroma_extractor` 검색 → 더블클릭
3. 연결 4개:
   - `in_stream` ← MIPI Rx의 `video_out`
   - `out_stream` → Frame Buf Write의 `s_axis_video`
   - `ap_clk` ← `pl_clk1` (200MHz)
   - `ap_rst_n` ← `proc_sys_reset_1.peripheral_aresetn`
4. AXI-Lite 제어 인터페이스 → SmartConnect에 연결
5. **Validate Design** (F6)

#### B-3. 비트스트림 재생성
- Generate Bitstream (30분~1시간)
- Export Hardware (`.xsa`)

#### B-4. KV260에 로드
```bash
# PC에서 SCP
scp design_1.bit ubuntu@192.168.0.11:~/
scp design_1.hwh ubuntu@192.168.0.11:~/
```

#### B-5. PYNQ 동작 검증
```python
from pynq import Overlay
overlay = Overlay("/home/ubuntu/design_1.bit", ignore_version=True)

# IP 목록에 chroma_extractor 있는지
print(overlay.ip_dict.keys())

# 레지스터 직접 접근
print(overlay.chroma_extractor_0.register_map)

# IP 시작 명령
overlay.chroma_extractor_0.register_map.CTRL.AP_START = 1
```

### Phase B 산출물

- 새 비트스트림 (VPU IP 포함)
- PYNQ에서 IP 인식 + 제어 가능

---

## Phase C: 진짜 10채널 VPU (Spatial)

### 목표

> PyTorch에서 검증된 10채널 VPU를 HLS로 포팅. 픽셀 단위 처리.

### 구현 채널 (PyTorch 결과 기준 ~98% 정확도)

| # | 채널 | HLS 구현 |
|---|------|---------|
| 0 | Luminance bandpass | `Y - avg_pool(Y, 5×5)` |
| 1 | R-G opponent | YUV의 `V` 사용 (또는 변형) |
| 2 | B-Y opponent | YUV의 `U` 사용 (또는 변형) |
| 3 | Edge magnitude | `|gx| + |gy|` (Sobel, L1 norm) |
| 4 | Edge orientation X | `gx / mag` (정규화) |
| 5 | Edge orientation Y | `gy / mag` |
| 6~9 | Gabor 0°/45°/90°/135° | 5×5 fixed weights |

> 💡 edge_orientation은 atan2 대신 (gx/mag, gy/mag) 두 채널로 분리 (FPGA 친화적).

### 구현 전략

#### C-1. 라인 버퍼 (5줄)
```cpp
// 5×5 커널 처리를 위해 최근 5줄 BRAM에 유지
ap_uint<8> line_buffer[5][1920];
#pragma HLS ARRAY_PARTITION variable=line_buffer dim=1 complete
```

#### C-2. 슬라이딩 윈도우
```cpp
// 5×5 윈도우 (line buffer + 가로 4픽셀)
ap_uint<8> window[5][5];

// 매 사이클 1픽셀씩 시프트
for (int i = 0; i < 5; i++) {
    for (int j = 0; j < 4; j++) {
        #pragma HLS UNROLL
        window[i][j] = window[i][j+1];
    }
    window[i][4] = line_buffer[i][x];
}
```

#### C-3. 채널별 연산
```cpp
// Channel 3: Sobel magnitude
int gx = (window[0][2] - window[0][0])
       + 2*(window[1][2] - window[1][0])
       + (window[2][2] - window[2][0]);
int gy = (window[2][0] - window[0][0])
       + 2*(window[2][1] - window[0][1])
       + (window[2][2] - window[0][2]);
ap_uint<10> edge_mag = ABS(gx) + ABS(gy);

// Channel 6: Gabor 0° (fixed weights)
const int gabor0[5][5] = {...};
int gabor0_out = 0;
for (int i = 0; i < 5; i++)
    for (int j = 0; j < 5; j++)
        #pragma HLS UNROLL
        gabor0_out += window[i][j] * gabor0[i][j];
```

#### C-4. 10채널 출력 패킹
AXI-Stream으로 10채널 한꺼번에 송신:
- TDATA = 80bit (8bit × 10ch)
- 또는 시분할 송신 (다른 IP와 협의)

#### C-5. PyTorch와 Cross-validation
```python
# PyTorch 결과
pytorch_out = retinal_vpu(input_image)

# HLS C Simulation 결과
hls_out = run_hls_testbench(input_image)

# 차이 측정
np.allclose(pytorch_out, hls_out, atol=1)  # 비트 정확 또는 ε 이내
```

### Phase C 산출물

- `vpu_retinal_spatial.zip` (10채널 spatial VPU IP)
- 자원: 약 10~30K LUT 추정
- PyTorch 결과와 99%+ 일치

---

## Phase D: 시간/모션 채널 (12채널 추가)

### 목표

> 정적 처리 → 비디오 스트림 처리 (망막의 진짜 강점).

### 구현 채널

#### Temporal (4ch)
| 채널 | 수식 | HLS 구현 |
|------|------|---------|
| dL/dt | `Y(t) - Y(t-1)` | 1프레임 지연 버퍼 (DDR or BRAM) |
| ON transient | `ReLU(dL/dt)` | comparator |
| OFF transient | `ReLU(-dL/dt)` | comparator |
| Sustained | IIR low-pass | `y = α·x + (1-α)·y_prev` |

#### Motion (4ch, Reichardt detector)
| 채널 | 수식 | 의미 |
|------|------|------|
| DS_right | `prev(x-1, y) × curr(x, y)` | 오른쪽 이동 감지 |
| DS_left | `prev(x+1, y) × curr(x, y)` | 왼쪽 이동 감지 |
| DS_up | `prev(x, y+1) × curr(x, y)` | 위쪽 이동 |
| DS_down | `prev(x, y-1) × curr(x, y)` | 아래쪽 이동 |

#### Adaptation (2ch)
- Local luminance normalization
- EMA over space+time

#### Color extension (2ch)
- S-OFF (구현 시)
- konio_trans

### 메모리 요구사항

```
1프레임 지연 버퍼:
1920 × 1080 × 8bit (Y만) = 약 2MB

→ BRAM에 안 들어감 (5.1Mb 한계)
→ DDR 사용 필요 (AXI4 Full 인터페이스 추가)
```

### Phase D 산출물

- 22채널 풀 VPU IP
- DDR 활용 시간 처리
- 동영상에서 OFF/ON transient, motion 검출

---

## Phase E: 영상 검증

### 옵션 1: 시뮬 영상 (추천)

```
[PC]                  [KV260]
이미지 파일       ─▶  DDR에 직접 write (PYNQ)
                          ↓
                  AXI DMA가 VPU 입력에 흘려보냄
                          ↓
                  VPU 22채널 출력 → DDR write
                          ↓
PyTorch 결과 ◀── DDR에서 numpy로 회수
와 비교
```

**장점**:
- 카메라 변수 X (조명, 노이즈)
- 같은 입력 반복 가능
- PyTorch와 정확히 비교

**단, AXI DMA IP 추가 + 비트스트림 재생성 필요**.

### 옵션 2: 실제 카메라 (시연용)

- AP1302 firmware 받기 (GitHub 우회 or AMD 다운로드)
- I2C로 init
- MIPI Rx 활성화
- 실시간 영상 → VPU → DDR
- PYNQ에서 프레임 회수 + 표시

**장점**: 진짜 실시간 망막 카메라 데모.  
**단점**: AP1302 init 까다로움 (수백 레지스터).

### Phase E 산출물

- VPU IP의 실제 영상 처리 검증
- PyTorch와 cross-validation 결과 (정확도, latency, throughput)

---

## 시작점: Phase A-1

### 지금 즉시 할 수 있는 것

```
1. Vitis HLS 2025.2 실행
2. Workspace: C:/Users/ejfhr/FPGA/hls_workspace
3. 새 component: vpu_chroma_extractor
4. 3개 파일 작성 (위 코드 그대로)
5. C Simulation → Pass-through 동작 확인
```

### 진행 후 다음 작업

```
Phase A 끝나면 → Phase B (Vivado 통합)
Phase B 끝나면 → Phase C (진짜 10채널 VPU)
```

---

## 진척 추적용 체크리스트

### Phase A
- [ ] A-1: Vitis HLS 환경 셋업
- [ ] A-2: 소스 3개 작성
- [ ] A-3: C Simulation 통과
- [ ] A-4: C Synthesis 성공
- [ ] A-5: IP Export

### Phase B
- [ ] B-1: Vivado IP Catalog 추가
- [ ] B-2: Block Design에 끼워넣기
- [ ] B-3: 비트스트림 재생성
- [ ] B-4: KV260에 로드
- [ ] B-5: PYNQ에서 IP 인식 + 제어

### Phase C
- [ ] C-1: 라인 버퍼 구현
- [ ] C-2: 슬라이딩 윈도우
- [ ] C-3: 10채널 연산 다 구현
- [ ] C-4: 10채널 패킹/출력
- [ ] C-5: PyTorch와 cross-validation

### Phase D
- [ ] D-1: Temporal 4채널
- [ ] D-2: Motion 4채널 (Reichardt)
- [ ] D-3: Adaptation 2채널
- [ ] D-4: Color extension 2채널

### Phase E
- [ ] E-1: AXI DMA 추가
- [ ] E-2: 시뮬 영상 검증
- [ ] E-3: (옵션) 카메라 동작

---

## 참고

- **work_state.md**: 프로젝트 전체 상태
- **study_session_overlay.md**: PYNQ Overlay + 핵심 개념 학습 기록
- **retinal_vpu_channels.md**: 22채널 망막 VPU 로드맵 (생물학적 매핑 + FPGA 비용)
