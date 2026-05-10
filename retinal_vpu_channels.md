# Retinal VPU Channel Roadmap

망막 영감 VPU (Vision Preprocessing Unit) 개발을 위한 채널 설계 문서.
현재 구현 상태와 생물학적 망막의 알려진 채널을 매핑하고, 구현 우선순위를 정리.

---

## 1. 현재 구현된 VPU 채널 (10ch)

| # | 채널명 | 수식/방법 | 생물학적 대응 | 역할 |
|---|--------|----------|--------------|------|
| 0 | Luminance bandpass | `lum - avg_pool(lum, 15)` | Parasol/M cell center-surround | 중심-주변 대비, 공간 고주파 |
| 1 | R-G opponent | `R - G` | Midget/P cell (L vs M cone) | 적-녹 색 대립 |
| 2 | B-Y opponent | `B - 0.5(R+G)` | Bistratified/K cell (S vs L+M) | 청-황 색 대립 |
| 3 | Edge magnitude | `√(Sx² + Sy²)` | Local edge detector RGC | 에지 강도 |
| 4 | Edge orientation | `atan2(Sy, Sx)` | Orientation-selective RGC | 에지 방향 (⚠ 불연속) |
| 5 | Gabor 0° | 5×5 fixed | Orientation-selective RGC | 수평 에지 |
| 6 | Gabor 45° | 5×5 fixed | Orientation-selective RGC | 우상향 에지 |
| 7 | Gabor 90° | 5×5 fixed | Orientation-selective RGC | 수직 에지 |
| 8 | Gabor 135° | 5×5 fixed | Orientation-selective RGC | 좌상향 에지 |
| 9 | Raw luminance | `0.5R + 0.5G` | - | 원본 밝기 (baseline) |

**성격**: 전부 **공간(spatial) 채널**. 시간축 처리 0. 정적 이미지용.
**현재 성능**: CIFAR-10 기준 RGB 대비 ~98% 정확도 (거의 정보 손실 없음).

---

## 2. 포유류 망막의 알려진 채널 전체 지도

망막 신경절세포(RGC)는 포유류에서 **40~50종**이 구분되어 있음. 주요 기능 채널별로 분류.

### 2.1 공간 채널 (Spatial)

| 세포 타입 | 비율 | 기능 | 구현 상태 |
|----------|------|------|-----------|
| **Midget ON/OFF (Parvo, P)** | ~80% | 고해상도, 색(R-G), 중심와 집중, sustained | 🟡 부분 (R-G만, sustained X) |
| **Parasol ON/OFF (Magno, M)** | ~10% | 저해상도, 휘도, transient, 전체 분포 | 🟡 부분 (bandpass만, transient X) |
| **Bistratified (Konio, K)** | ~8% | B-Y 색 대립 | 🟡 부분 (공간만) |
| **Local edge detector (W3)** | 소수 | 미세 에지, 작은 수용야 | ✅ 있음 |
| **Orientation-selective (OS)** | 소수 | 특정 방향 에지 | ✅ 있음 (Gabor 4방향) |
| **Uniformity detector** | 소수 | 균일한 영역 검출 (대비 없음) | ❌ 없음 |

### 2.2 시간 채널 (Temporal) — **전부 없음**

| 세포 타입 | 기능 | 구현 상태 |
|----------|------|----------|
| **ON transient** | 밝아지는 순간만 반응 | ❌ 없음 |
| **OFF transient** | 어두워지는 순간만 반응 | ❌ 없음 |
| **ON sustained** | 밝은 상태 지속 신호 | ❌ 없음 |
| **OFF sustained** | 어두운 상태 지속 신호 | ❌ 없음 |
| **ipRGC (melanopsin)** | 매우 느린 밝기 평균 (초 단위) | ❌ 없음 |
| **Suppressed-by-contrast** | 대비 낮을 때만 반응 | ❌ 없음 |

### 2.3 움직임 채널 (Motion) — **전부 없음**

| 세포 타입 | 기능 | 구현 상태 |
|----------|------|----------|
| **ON-OFF direction-selective (DS)** | 4방향 움직임 감지 (starburst amacrine 기반) | ❌ 없음 |
| **ON direction-selective** | 3방향 느린 움직임 (광시운동반사) | ❌ 없음 |
| **Object motion sensitive (OMS)** | 배경 대비 물체 움직임 (자기운동 상쇄) | ❌ 없음 |
| **Approach-sensitive / Looming** | 빠른 확장 (위협 감지) | ❌ 없음 |

### 2.4 적응 & 게인 제어 (Adaptation) — **전부 없음**

| 메커니즘 | 담당 세포 | 기능 | 구현 상태 |
|---------|----------|------|-----------|
| **Luminance adaptation** | Horizontal cell | 전체 밝기 대비 정규화 | ❌ 없음 (지금은 고정 정규화) |
| **Contrast gain control** | Amacrine cell | 로컬 대비 자동 조절 | ❌ 없음 |
| **Scotopic/photopic switch** | Rod vs Cone | 저조도/고조도 경로 전환 | ❌ 없음 |
| **Mesopic mixing** | AII amacrine | 중간 조도 신호 통합 | ❌ 없음 |

### 2.5 색 (Chromatic) 세부

| 채널 | 세포 | 구현 상태 |
|------|------|----------|
| L-M (적-녹) | Midget RGC | ✅ R-G |
| S-(L+M) (청-황) | Small bistratified | ✅ B-Y |
| S-OFF | Giant bistratified | ❌ 없음 |
| Luminance (L+M) | Parasol | ✅ raw lum |

---

## 3. 구현 우선순위 (연구 임팩트 기준)

### Priority 1 — Temporal 기본 3채널 ⭐⭐⭐
망막의 정체성. 이게 있어야 "VPU가 RGB 대체" 주장 가능.

```
- dL/dt          (time derivative of luminance)
- ON transient   = ReLU(dL/dt)
- OFF transient  = ReLU(-dL/dt)
```

**FPGA 구현 비용**: 1프레임 지연 버퍼 + 뺄셈 = 거의 공짜

### Priority 2 — Sustained/Transient 분리 ⭐⭐⭐
Parvo vs Magno 구분 구현:

```
- Sustained: IIR low-pass filter (느린 시간상수)
- Transient: IIR high-pass filter (빠른 시간상수)
```

**FPGA 구현 비용**: 1-tap IIR 필터 2개 = 저렴

### Priority 3 — Direction-selective 4채널 ⭐⭐⭐
Reichardt detector (곤충 시각에서 유래, 망막 starburst도 유사):

```
DS_right = prev_frame(x-1, y) * current_frame(x, y)   # 오른쪽 이동
DS_left, DS_up, DS_down 동일 방식
```

**FPGA 구현 비용**: 곱셈 + 1픽셀 시프트 = 매우 저렴

### Priority 4 — Luminance adaptation ⭐⭐
Horizontal cell 역할. 전체 파이프라인 robustness 크게 올림:

```
local_mean = EMA over (space + time)
adapted = (lum - local_mean) / (local_std + ε)
```

**FPGA 구현 비용**: 이동평균 레지스터

### Priority 5 — Looming detector ⭐
Approach-sensitive RGC. 자율주행/로봇 응용에서 킬러 피처:

```
expansion = Σ(outward optical flow)
```

**FPGA 구현 비용**: 중간 (옵티컬 플로우 필요)

### Priority 6 — Log-polar foveation ⭐⭐
세포 타입은 아니지만 **공간 분포**: 중심와 밀집 / 주변 희박. 연산량 절감 핵심.

**FPGA 구현 비용**: 좌표 변환 LUT

### Priority 7 — Scotopic path ⭐
저조도용 rod 경로. 낮 영상에선 의미 없음. 저조도 응용 시에만.

### Priority 8 — ipRGC (초저주파 밝기) △
시간상수 초 단위. AI 추론용 신호론 쓸모 적음. 노출 자동조절용.

### 생략해도 되는 것
- **Uniformity detector** — CNN이 알아서 학습 가능
- **Suppressed-by-contrast** — 응용 특수
- **S-OFF** — S-cone 자체가 드물어 영향 미미

---

## 4. 최종 목표 채널 구성 (예상)

구현 우선순위 1~4까지 다 넣으면 **22채널 VPU**가 됨:

```
Spatial (10ch, 현재):   lum_bp, R-G, B-Y, edge_mag, edge_ori,
                         gabor0, gabor45, gabor90, gabor135, raw_lum
Temporal (4ch):          dL/dt, ON_trans, OFF_trans, sustained_lum
Motion (4ch):            DS_right, DS_left, DS_up, DS_down
Adaptation (2ch):        contrast_normalized_lum, local_std
Color extension (2ch):   S-OFF, konio_trans
───────────────────────────────────────────────
총 22 channels → Vision Model 입력
```

공간 분포에 **log-polar foveation** 적용하면 토큰 수는 오히려 RGB 대비 **줄어듦** (주변부 압축). 이게 "같은 연산에 더 높은 해상도" 논리의 완성.

---

## 5. 개선이 필요한 현재 채널

### 채널 4 (edge orientation) — `atan2` 불연속 문제
`[-π, π]` 경계에서 `π`와 `-π`가 같은 방향인데 수치상 정반대.
→ **대안**: `(gx/mag, gy/mag)` 두 채널로 쪼개기 (정규화된 gradient 벡터).

### FPGA 이식 시 피해야 할 연산
- `atan2`: CORDIC 필요, 비쌈
- `sqrt`: 근사 필요 → `|gx| + |gy|` (L1 norm) 대체 검토
- 부동소수점 나눗셈: 전역 제거 목표

---

## 6. 참고 문헌

- **Gollisch & Meister (2010)** "Eye smarter than scientists believed" — 망막 계산 종합 리뷰
- **Baden et al. (2016, Nature)** "The functional diversity of retinal ganglion cells" — 마우스 RGC 32종 분류
- **Masland (2012)** "The neuronal organization of the retina" — 세포 타입 교과서급 리뷰
- **Lindsey et al. (2019)** "A unified theory of early visual representations from retina to cortex" — ML 관점 해석
