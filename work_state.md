# Retinal VPU on KV260 — Work State

망막 영감 VPU(Vision Preprocessing Unit) FPGA 구현 프로젝트. 현재까지 진척과 다음 작업 정리.

---

## 0. 큰 그림

```
[목표]
망막처럼 사전처리한 신호를 AI 모델에 보내는 비전 시스템.
RGB 그대로 보내지 않고, retina-style opponent/edge/motion 채널로 변환.

[전략]
PyTorch 검증 → FPGA 구현 → 자체 ASIC (장기)

[하드웨어]
KV260 Vision AI Starter Kit
  ├─ Zynq UltraScale+ ZU5EV (PS + PL 한 칩)
  ├─ AR1335 13MP 센서 + AP1302 ISP (IAS 카메라 모듈)
  └─ DDR4, DisplayPort, USB, Ethernet
```

---

## 1. PyTorch 단계 (이전 완료)

### 구현
- **RetinalVPU** 10채널: lum_bp, R-G, B-Y, edge_mag, edge_ori, gabor 4방향, raw_lum
- **Pipeline**: VPU(고정) → SmallViT(학습)
- 데이터셋: CIFAR-10 (해상도 한계로 다른 데이터셋 필요)

### 결과
- VPU + ViT가 RGB + ViT의 **약 98% 정확도 달성**
- 정확도 우위는 아니지만 **near-lossless preprocessing** 증명
- 의미: VPU 다음 단계에서 압축/foveation 가능 → 같은 연산에 더 높은 해상도

### 핵심 통찰
> 망막은 130M photoreceptors → 1M ganglion cells (130배 압축).  
> ML은 모든 픽셀 균일 처리 = 비효율.  
> 망막 방식이면 같은 compute로 더 높은 해상도 가능 (이게 진짜 가치).

---

## 2. KV260 OS 셋업

### Ubuntu + PYNQ
```
SD 카드 (512GB)
└─ Ubuntu 22.04 Classic Desktop for Kria
   └─ Kria-PYNQ install.sh로 PYNQ 추가
      └─ PYNQ 3.0.1 + JupyterLab :9090
```

### 막혔던 곳들
- **PPA 차단**: ubuntu-xilinx PPA가 Connection refused
  - 시도: DNS 변경, IPv4 강제, MTU 조정 — 다 실패
  - 결국 발견: **PPA 자체가 private** (Xilinx 비공개)
  - 일반 사용자는 ppa.launchpad.net에서 카메라 firmware 못 받음
- **MTU 문제**: 한국 ISP의 PMTUD black hole로 TCP 끊김
  - 해결: 별 효과 없었음. 결국 ping은 되는데 curl은 안 되는 상태로 남음

### 결과
- PYNQ 환경 정상 (Python 3.10, /dev/video는 없음)
- launchpad PPA의 `xlnx-firmware-kv260-smartcam`은 못 받음
- → **자체 비트스트림 만드는 길로 결정**

---

## 3. Vivado Block Design (자체 비트스트림)

### Phase 1~2: 프로젝트 + PS
- 새 프로젝트 + KV260 보드 선택
- `Zynq UltraScale+ MPSoC` 추가 + Run Block Automation (Apply Board Preset)
- PS 설정:
  - `M_AXI_HPM0_FPD` ✅, `M_AXI_HPM1_FPD` ❌, `S_AXI_HP0_FPD` ✅
  - `pl_clk0` = 100MHz, `pl_clk1` = 200MHz
  - PL→PS Interrupts `IRQ0[0:7]` ✅

### Phase 3: 클럭 + 리셋
- **Clocking Wizard**: pl_clk0(100MHz) → clk_out1(200MHz, D-PHY용)
  - PLL이 100→200 변환 (×N/÷M)
- **Processor System Reset 3개**:
  - reset_0: 100MHz 도메인 (제어)
  - reset_1: 200MHz video 도메인 (비디오 픽셀)
  - reset_2: 200MHz D-PHY 도메인 (MIPI)
- 각 reset = 비동기 PS reset → 클럭 동기화 reset 변환기

### Phase 4: AXI 인프라
- **AXI SmartConnect (제어용)**: Slave 1, Master 2, **Clocks 2개** (다중 도메인)
  - aclk = 100MHz, aclk1 = 200MHz
  - PS의 M_AXI_HPM0_FPD 받아서 여러 IP에 분배
  - **다중 클럭 = 100MHz↔200MHz 자동 CDC 처리**
- ~~AXI IIC~~ 추가했다가 나중에 삭제 (PS MIO가 카메라 I2C 처리하므로 PL에 불필요)

### Phase 5: 비디오 입력
- **MIPI CSI-2 Rx Subsystem**:
  - YUV422 8bit, 4 lanes, 800Mbps line rate
  - Pixels Per Clock 1
  - **Include Shared Logic in core** (MMCM/PLL 내장 모드)
  - 클럭 3개: lite_aclk(100MHz), video_aclk(200MHz), dphy_clk_200M(clk_wiz)
- **AXI4-Stream Subset Converter** (Register Slice 대신):
  - 2바이트 → 3바이트 폭 변환 (Frame Buf가 24bit 폭 요구)
  - Remap: `8'b00000000,tdata[15:0]` (zero padding)
  - 이게 미래에 **Retinal VPU IP 자리**

### Phase 6: 메모리 출력
- **Video Frame Buffer Write**:
  - 1920×1080, UYVY8 only, Samples per Clock 1, Address Width 64
  - 단일 클럭 IP (`ap_clk` 하나로 데이터+제어 둘 다)
  - AXI-Stream → AXI Memory Map (DDR)
- **AXI SmartConnect (데이터용)**: Frame Buf의 m_axi_mm_video → PS의 S_AXI_HP0_FPD
- **Inline Concat**: 인터럽트 묶기 (MIPI Rx + Frame Buf → PS pl_ps_irq0)

### Phase 7: 마무리
- `mipi_phy_if`, ~~`IIC`~~ 외부 핀 (Make External)
- Address Editor → Auto Assign
- **Validate Design 통과**

### Phase 8: Bitstream
- HDL Wrapper 자동 생성
- Synthesis → Implementation → Bitstream
- 시간: 약 30분
- Export Hardware → `.xsa` (ZIP) → `.bit` + `.hwh` 추출
- KV260로 SCP 전송

### 막혔던 곳들
- **TDATA 폭 mismatch (3 vs 2)**: Register Slice는 폭 변환 못함 → Subset Converter로 교체
- **IIC 핀 LOC 미설정**: Make External만 하고 board 매핑 안 함
  - DRC NSTD-1, UCIO-1 에러 → bitstream 생성 거부
  - 해결: AXI IIC IP 자체 삭제 (KV260은 PS MIO가 카메라 I2C 처리)
- **Tcl `set_property SEVERITY` 명령 무효**: 새 Implementation 프로세스에 적용 안 됨

---

## 4. PYNQ Overlay 로드 (현재 위치)

### 성공한 것
```python
from pynq import Overlay
overlay = Overlay("/home/ubuntu/design_1.bit", ignore_version=True)
print(overlay.ip_dict.keys())
# → mipi_csi2_rx_subsyst_0, v_frmbuf_wr_0, axis_subset_converter_0, ...
```

dmesg:
```
fpga_manager fpga0: writing design_1.bin to Xilinx ZynqMP FPGA Manager
bitstream ... locked, ref=1
```

→ **PL 회로가 우리 디자인으로 변경됨** ✅

### 안 된 것
- `/dev/video*` 없음 — Linux device tree overlay (.dtbo) 없어서 카메라 driver 자동 로드 안 됨
- IP version mismatch warning (PYNQ driver는 v5.1, IP는 v6.0) — 동작엔 영향 X

### 의미
- **인프라 단계 완료**: 비트스트림 만들기 + 로드까지 됨
- 다음은 카메라 실제 동작 (큰 작업)

---

## 5. 다음 단계 (옵션)

### A. PL 동작 검증 (가장 빠름)
```python
print(overlay.v_frmbuf_wr_0.register_map)
# 레지스터 정의 출력되면 PS↔PL AXI 통신 OK
```

### B. 망막 VPU IP 만들기 (연구 본분)
- HLS로 ChromaExtractor (또는 더 풍부한 retinal IP) 작성
- 현재 디자인의 `axis_subset_converter_0` 자리에 교체
- 비트스트림 다시 만들기

### C. 카메라 직접 제어 (긴 작업, 1주+)
- AP1302 firmware 다운로드 (별도 경로 필요)
- I2C로 AP1302 init (수백 개 레지스터)
- MIPI Rx 레지스터 활성화
- Frame Buf 시작 + DDR에서 프레임 읽기
- 이미지 표시

---

## 6. 공부해야 할 개념 (이번에 등장한 것 정리)

### Vivado / Block Design
- [ ] **AXI 프로토콜**: AXI4 (full memory), AXI4-Lite (제어), AXI4-Stream (비디오)
  - TDATA, TLAST, TUSER, TKEEP의 역할
  - Master/Slave 인터페이스 + AXI 인터커넥트
- [ ] **Block Automation vs Connection Automation**
  - Block Automation = IP 내부 설정 자동
  - Connection Automation = IP 간 연결 자동
- [ ] **Clock Domain Crossing (CDC)**
  - 다른 클럭 도메인 간 데이터 전송 시 sync 필요
  - SmartConnect의 다중 클럭 = 자동 CDC FIFO 삽입
- [ ] **Validate Design**의 의미
  - 핀 연결 + 클럭 도메인 + AXI 호환성 등 자동 체크
- [ ] **Synthesis vs Implementation vs Bitstream**
  - Synthesis: HDL → 게이트
  - Implementation: 게이트 → 실제 LUT/FF 배치 (Place & Route)
  - Bitstream: 최종 FPGA 설정 파일

### KV260 / Kria
- [ ] **PS와 PL 분리 구조**
  - PS = ARM Cortex-A53 ×4, DDR, MIO
  - PL = FPGA fabric, BRAM, DSP
  - 둘 사이 AXI 버스로 연결
- [ ] **MIO vs PL IO**
  - MIO = PS 직결 핀 (UART, Ethernet, I2C 등)
  - PL IO = PL fabric에 연결된 핀
- [ ] **Device Tree Overlay (.dtbo)**
  - Linux에게 "이 보드에 이런 하드웨어 있음" 알림
  - PL 비트스트림 + .dtbo가 한 쌍
  - PYNQ는 .hwh로 IP 정보 제공 (device tree 대체)

### MIPI CSI-2
- [ ] **D-PHY**: MIPI 물리 계층 (고속 직렬, 200~2500Mbps/lane)
- [ ] **Lane**: 데이터 차선 (보통 2 또는 4)
- [ ] **Pixel format vs Line rate**:
  - Pixel format = 픽셀 비트 폭 (YUV422 8bit = 16bit/px)
  - Line rate = 한 lane의 비트 전송 속도 (Mbps)

### YUV vs RGB
- [ ] **YUV422 vs YUV420 vs YUV444**:
  - 4:4:4 = 모든 픽셀에 Y, U, V 다 있음
  - 4:2:2 = U, V는 가로 1/2 (망막 opponent와 가장 유사)
  - 4:2:0 = U, V는 가로 1/2 + 세로 1/2 (압축 효율)
- [ ] **UYVY vs YUYV**: 패킹 순서 차이 (MIPI 표준 = UYVY)
- [ ] **AP1302 ISP 칩**:
  - 카메라 모듈 안에 박힌 ISP
  - Demosaic + AWB + 감마 등 자동
  - 출력 형식 I2C로 설정 (YUV422, RGB888 등)

### PYNQ
- [ ] **Overlay**: 비트스트림 + .hwh 묶음, Python에서 로드/제어
- [ ] **`overlay.ip_dict`**: PL의 IP 목록
- [ ] **`overlay.<ip_name>.register_map`**: IP 레지스터 직접 접근
- [ ] **xlnx-config / xmutil**: Kria 보드 firmware 관리 도구

### FPGA 설계 일반
- [ ] **Timing closure**: 클럭에 맞춰 데이터가 도착하는지
- [ ] **Methodology Violations vs Critical Warnings**: 보통 무시 가능
- [ ] **DRC (Design Rule Check)**: 핀 LOC, IO standard 등 강제 검사
- [ ] **HDL Wrapper**: Block Design을 Verilog 모듈로 감쌈

---

## 7. 다음에 다시 시작할 때 체크리스트

### Vivado 환경
- [ ] 프로젝트: `C:/Users/ejfhr/FPGA/project_2/project_2.xpr`
- [ ] Block Design: `design_1.bd` (활성)
- [ ] Bitstream: `project_2.runs/impl_1/design_1_wrapper.bit`
- [ ] Hardware Handoff: `project_2.gen/sources_1/bd/design_1/hw_handoff/design_1.hwh`

### KV260 상태
- [ ] IP: `192.168.0.11` (DHCP, 변할 수 있음)
- [ ] Ubuntu 22.04 + PYNQ 3.0.1
- [ ] DNS: 1.1.1.1, 8.8.8.8 (NetworkManager 설정됨)
- [ ] 비트스트림 위치: `/home/ubuntu/design_1.bit` + `design_1.hwh`

### Jupyter 접속
- URL: `http://192.168.0.11:9090/lab`
- 비번: `xilinx`
- Overlay 로드:
  ```python
  from pynq import Overlay
  overlay = Overlay("/home/ubuntu/design_1.bit", ignore_version=True)
  ```

---

## 8. 막혔던 곳 빠른 참조

| 문제 | 해결 |
|------|------|
| PPA Connection refused | private PPA였음 → 자체 비트스트림 |
| `/dev/video*` 없음 | 비트스트림에 device tree 없어서. PYNQ로 IP 직접 제어 |
| TDATA mismatch (3 vs 2) | Register Slice → Subset Converter (8'b00000000,tdata[15:0]) |
| IIC 핀 LOC 미설정 | axi_iic_0 IP 자체 삭제 |
| Implementation outdated | Tcl 명령은 새 run에 안 적용 → 디자인 자체 수정 |
| Bitstream 가운데 끊김 | Vivado 재실행 + Validate 다시 |

---

## 9. 참고 자원

- **AMD/Xilinx**:
  - KV260 product page: https://www.xilinx.com/products/som/kria/kv260-vision-starter-kit.html
  - Kria-PYNQ GitHub: https://github.com/Xilinx/Kria-PYNQ
  - kria-vitis-platforms: https://github.com/Xilinx/kria-vitis-platforms
- **로컬 ref design**: `C:/Users/ejfhr/FPGA/reference_designs/kria-vitis-platforms/kv260/platforms/kv260_ispMipiRx_vmixDP/`

---

_최종 업데이트: 비트스트림 KV260 로드 + IP 인식 확인 완료. /dev/video는 미생성._
