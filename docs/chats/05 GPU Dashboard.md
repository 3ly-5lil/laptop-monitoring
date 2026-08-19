# GPU Dashboard — Conversation Summary

## 1. Context

The goal was to build a dedicated Grafana **NVIDIA GPU dashboard** for a laptop using **DCGM Exporter** and Prometheus.

The intended dashboard should make it easy to correlate:

- GPU utilization
- VRAM usage
- GPU temperature
- GPU power
- GPU clocks
- Memory activity
- GPU video engine activity
- GPU processes, if process-level metrics are available

The GPU is an **NVIDIA GeForce GTX 1650 Ti** with approximately **4 GiB VRAM**.

---

## 2. Original Dashboard Concept

The initial proposed dashboard had:

1. Top-row Stat panels:
   - GPU Utilization
   - VRAM Used
   - Temperature
   - Power

2. Time-series rows:
   - GPU Utilization
   - GPU Memory Utilization
   - GPU Temperature
   - GPU Power
   - GPU Clock
   - Memory Clock

3. A final GPU Processes table showing conceptually:

```text
Process          GPU      VRAM
--------------------------------
python           82%      2.3 GB
chrome            4%      120 MB
Xorg              1%       80 MB
```

The dashboard was designed around Grafana's **24-column grid**.

---

# 3. Problem: GPU Process Metrics Were Not Available

The available DCGM metrics were inspected directly.

There are **no process-level metrics** such as `PROC`, `PROCESS`, or equivalent.

Therefore, DCGM Exporter cannot currently provide a table such as:

python    82%    2.3 GB

chrome     4%    120 MB

from the metrics currently exposed.

### Decision

Do **not** invent or approximate GPU process metrics.

The GPU dashboard should remain focused on **device-level DCGM metrics**.

If process → GPU utilization → VRAM attribution is required later, a separate GPU process collector/exporter or NVIDIA-specific process metric source will be needed.

---

# 4. Available DCGM Metrics

The exporter currently exposes these relevant metrics:

### GPU utilization

DCGM_FI_DEV_GPU_UTIL

GPU utilization in percent.

Current example:

24%

### VRAM

DCGM_FI_DEV_FB_USED

DCGM_FI_DEV_FB_FREE

DCGM_FI_DEV_FB_RESERVED

All are in MiB.

Current values:

USED      = 32 MiB

FREE      = 3683 MiB

RESERVED  = 380 MiB

### GPU temperature

DCGM_FI_DEV_GPU_TEMP

Current example:

45 °C

### GPU power

DCGM_FI_DEV_POWER_USAGE

Current example:

4.4 W

### GPU clock

DCGM_FI_DEV_SM_CLOCK

Current example:

300 MHz

### Memory clock

DCGM_FI_DEV_MEM_CLOCK

Current example:

405 MHz

### Memory controller utilization

DCGM_FI_DEV_MEM_COPY_UTIL

Current example:

10%

Important distinction:

- `FB_USED` = how much VRAM is occupied
- `MEM_COPY_UTIL` = how actively the GPU memory subsystem is being used

They are not the same metric.

### Encoder utilization

DCGM_FI_DEV_ENC_UTIL

### Decoder utilization

DCGM_FI_DEV_DEC_UTIL

### Other available metrics

DCGM_FI_DEV_PCIE_REPLAY_COUNTER

DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION

DCGM_FI_DEV_VGPU_LICENSE_STATUS

DCGM_FI_DEV_MEMORY_TEMP

`DCGM_FI_DEV_MEMORY_TEMP` currently reports `0`, so it should not be displayed as a meaningful temperature.

---

# 5. Important VRAM Calculation Correction

Initially, VRAM total was considered as:

DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE

However, the actual metrics showed:

32 + 3683 = 3715 MiB

while the GPU has approximately 4096 MiB.

The missing ~380 MiB corresponds to:

DCGM_FI_DEV_FB_RESERVED = 380 MiB

Therefore:

DCGM_FI_DEV_FB_USED

+

DCGM_FI_DEV_FB_FREE

+

DCGM_FI_DEV_FB_RESERVED

returns approximately:

4095 MiB

which is effectively the 4 GiB framebuffer capacity.

## Correct total VRAM query

DCGM_FI_DEV_FB_USED

+

DCGM_FI_DEV_FB_FREE

+

DCGM_FI_DEV_FB_RESERVED

---

# 6. VRAM Gauge Solution

For the Grafana VRAM Gauge:

### Value query

DCGM_FI_DEV_FB_USED

### Unit

MiB

### Minimum

0

### Maximum

4096

The reason for using `4096` as the Gauge maximum is that Grafana's Gauge Max is a field configuration value and is not simply another PromQL query.

The dynamically calculated total:

DCGM_FI_DEV_FB_USED

+

DCGM_FI_DEV_FB_FREE

+

DCGM_FI_DEV_FB_RESERVED

can still be used as a separate query if needed.

For this single fixed GTX 1650 Ti, using:

Max = 4096 MiB

is the simplest and clearest solution.

---

# 7. Final Dashboard Layout

Grafana uses a **24-column grid**.

## Row 1 — Current GPU State

**Height: 4**

Four panels, each **6 columns wide**.

┌──────────────┬──────────────┬──────────────┬──────────────┐

│ GPU Util     │ VRAM Used    │ Temperature  │ Power        │

│              │              │              │              │

└──────────────┴──────────────┴──────────────┴──────────────┘

      6              6              6              6

### Panel 1 — GPU Utilization

Visualization:

**Stat**

Query:

DCGM_FI_DEV_GPU_UTIL

Unit:

Percent (0-100)

Suggested thresholds:

70  = Warning

90  = Critical

---

### Panel 2 — VRAM Used

Visualization:

**Gauge**

Query:

DCGM_FI_DEV_FB_USED

Unit:

MiB

Min:

0

Max:

4096

---

### Panel 3 — GPU Temperature

Visualization:

**Stat**

Query:

DCGM_FI_DEV_GPU_TEMP

Unit:

°C

Suggested thresholds:

75 °C = Warning

85 °C = Critical

---

### Panel 4 — GPU Power

Visualization:

**Stat**

Query:

DCGM_FI_DEV_POWER_USAGE

Unit:

W

---

# 8. Row 2 — GPU Workload

**Height: 7**

Two panels, each **12 columns wide**.

## GPU Utilization

Visualization:

**Time series**

Query:

DCGM_FI_DEV_GPU_UTIL

Unit:

Percent (0-100)

Min:

0

Max:

100

Legend:

GPU {{gpu}}

---

## VRAM Usage

Visualization:

**Time series**

Query:

DCGM_FI_DEV_FB_USED

Unit:

MiB

Legend:

GPU {{gpu}}

This shows actual framebuffer occupancy over time.

---

# 9. Row 3 — Thermal and Power

**Height: 7**

Two panels, each **12 columns wide**.

## GPU Temperature

Visualization:

**Time series**

Query:

DCGM_FI_DEV_GPU_TEMP

Unit:

°C

Suggested thresholds:

75 °C = Warning

85 °C = Critical

Legend:

GPU {{gpu}}

---

## GPU Power

Visualization:

**Time series**

Query:

DCGM_FI_DEV_POWER_USAGE

Unit:

W

Legend:

GPU {{gpu}}

---

# 10. Row 4 — GPU Clocks

**Height: 6**

Two panels, each **12 columns wide**.

## GPU Clock

Visualization:

**Time series**

Query:

DCGM_FI_DEV_SM_CLOCK

Unit:

MHz

Legend:

GPU {{gpu}}

---

## Memory Clock

Visualization:

**Time series**

Query:

DCGM_FI_DEV_MEM_CLOCK

Unit:

MHz

Legend:

GPU {{gpu}}

---

# 11. Row 5 — Memory and Video Engines

**Height: 6**

Two panels, each **12 columns wide**.

## Memory Controller Utilization

Visualization:

**Time series**

Query:

DCGM_FI_DEV_MEM_COPY_UTIL

Unit:

Percent (0-100)

Min:

0

Max:

100

This is intentionally separate from VRAM usage.

---

## Encoder / Decoder Utilization

The GPU exposes both:

DCGM_FI_DEV_ENC_UTIL

and:

DCGM_FI_DEV_DEC_UTIL

Visualization:

**Time series**

Queries:

DCGM_FI_DEV_ENC_UTIL

DCGM_FI_DEV_DEC_UTIL

Unit:

Percent (0-100)

Legend:

Encoder

Decoder

This is useful for observing video workloads such as browser/video playback, encoding, and decoding.

---

# 12. Row 6 — Optional GPU Information

**Height: 5**

Full width:

**24 columns**

Possible information:

- GPU model
- Driver version
- PCIe replay counter
- Total energy consumption

The available labels identify the GPU as:

modelName="NVIDIA GeForce GTX 1650 Ti"

and driver:

DCGM_FI_DRIVER_VERSION="595.84"

This section is optional because the dashboard's primary purpose is monitoring rather than static hardware information.

---

# 13. Final Layout Summary

NVIDIA GPU

────────────────────────────────────────────────────────────

  

ROW 1                                      24 × 4

  

┌──────────────┬──────────────┬──────────────┬──────────────┐

│ GPU Util     │ VRAM Used    │ Temperature  │ Power        │

│    24%       │    32 MiB    │    45°C      │    4.4 W     │

└──────────────┴──────────────┴──────────────┴──────────────┘

  

  

ROW 2                                      24 × 7

  

┌────────────────────────────┬────────────────────────────┐

│ GPU Utilization            │ VRAM Usage                 │

│                            │                            │

└────────────────────────────┴────────────────────────────┘

  

  

ROW 3                                      24 × 7

  

┌────────────────────────────┬────────────────────────────┐

│ GPU Temperature            │ GPU Power                  │

│                            │                            │

└────────────────────────────┴────────────────────────────┘

  

  

ROW 4                                      24 × 6

  

┌────────────────────────────┬────────────────────────────┐

│ GPU Clock                  │ Memory Clock               │

│                            │                            │

└────────────────────────────┴────────────────────────────┘

  

  

ROW 5                                      24 × 6

  

┌────────────────────────────┬────────────────────────────┐

│ Memory Controller          │ Encoder / Decoder          │

│ Utilization                │ Utilization                │

└────────────────────────────┴────────────────────────────┘

  

  

ROW 6                                      24 × 5

  

┌──────────────────────────────────────────────────────────┐

│ GPU Information / PCIe / Energy                         │

└──────────────────────────────────────────────────────────┘

---

# 14. Final Metric Mapping

|Panel|Visualization|Metric|Unit|Grid|
|---|---|---|---|---|
|GPU Utilization|Stat|`DCGM_FI_DEV_GPU_UTIL`|%|6×4|
|VRAM Used|Gauge|`DCGM_FI_DEV_FB_USED`|MiB|6×4|
|Temperature|Stat|`DCGM_FI_DEV_GPU_TEMP`|°C|6×4|
|Power|Stat|`DCGM_FI_DEV_POWER_USAGE`|W|6×4|
|GPU Utilization|Time series|`DCGM_FI_DEV_GPU_UTIL`|%|12×7|
|VRAM Usage|Time series|`DCGM_FI_DEV_FB_USED`|MiB|12×7|
|Temperature|Time series|`DCGM_FI_DEV_GPU_TEMP`|°C|12×7|
|Power|Time series|`DCGM_FI_DEV_POWER_USAGE`|W|12×7|
|GPU Clock|Time series|`DCGM_FI_DEV_SM_CLOCK`|MHz|12×6|
|Memory Clock|Time series|`DCGM_FI_DEV_MEM_CLOCK`|MHz|12×6|
|Memory Controller|Time series|`DCGM_FI_DEV_MEM_COPY_UTIL`|%|12×6|
|Encoder/Decoder|Time series|`DCGM_FI_DEV_ENC_UTIL`, `DCGM_FI_DEV_DEC_UTIL`|%|12×6|
|GPU Info|Stat/Text/Table|Various|Various|24×5|

---

# 15. Key Decisions

1. **No GPU process table for now**
    - DCGM Exporter does not expose process-level metrics in the current configuration.
    - Do not fabricate process attribution.
2. **VRAM total includes reserved memory**
    - Correct calculation:

DCGM_FI_DEV_FB_USED

+

DCGM_FI_DEV_FB_FREE

+

DCGM_FI_DEV_FB_RESERVED

3. **VRAM Gauge uses 4096 MiB as Max**
    - The GTX 1650 Ti has approximately 4 GiB VRAM.
    - Grafana Gauge Max is not dynamically supplied by another PromQL query.
4. **VRAM usage and memory utilization are different**
    - `FB_USED` = occupied VRAM
    - `MEM_COPY_UTIL` = memory subsystem activity
5. **Memory temperature is excluded**
    - `DCGM_FI_DEV_MEMORY_TEMP` reports `0` and should not be treated as a real temperature.
6. **Encoder/decoder metrics are included**
    - They provide useful visibility into video workloads.
7. **Dashboard remains device-focused**
    - The current DCGM metric set is sufficient for GPU health, workload, thermal, power, clock, VRAM, memory activity, and video-engine monitoring.