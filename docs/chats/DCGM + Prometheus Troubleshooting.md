# DCGM + Prometheus Troubleshooting Summary

## Context

The goal was to validate that NVIDIA DCGM metrics were working correctly and being collected by Prometheus for the GPU monitoring dashboard.

The monitoring stack includes:

- NVIDIA GPU: GeForce GTX 1650 Ti, 4 GB VRAM
- NVIDIA Driver: 595.84
- CUDA: 13.2
- DCGM Exporter: `dcgm-exporter:4.8.0-4.7.2-ubuntu22.04`
- Prometheus
- Grafana

## Problem / Objective

The main question was how to verify that DCGM was functioning correctly all the way through to Prometheus.

The important distinction was that there are several separate layers:

```text
NVIDIA GPU
    ↓
DCGM
    ↓
dcgm-exporter :9400
    ↓
Prometheus scrape
    ↓
PromQL
    ↓
Grafana
```

A metric appearing in Grafana is only useful if the underlying exporter and Prometheus scrape are working correctly.

## Validation Procedure

### 1. Validate the GPU

Check that NVIDIA can see the GPU:

nvidia-smi

A more focused check:

nvidia-smi --query-gpu=index,name,temperature.gpu,utilization.gpu,memory.used,memory.total,power.draw --format=csv

This establishes that the GPU and NVIDIA driver are functioning.

### 2. Validate DCGM

If the DCGM CLI is available:

dcgmi discovery -l

The GPU should be detected.

Health/diagnostic checks can also be performed:

dcgmi health -g 0 -c

dcgmi diag -r 1

### 3. Validate DCGM Exporter

DCGM Exporter normally exposes Prometheus metrics on port `9400`.

Check the endpoint directly:

curl --fail http://localhost:9400/metrics

To inspect DCGM metrics specifically:

curl -s http://localhost:9400/metrics | grep '^DCGM_' | head -50

A particularly useful smoke test is:

curl -s http://localhost:9400/metrics | grep 'DCGM_FI_DEV_GPU_UTIL'

Expected output is a Prometheus metric similar to:

DCGM_FI_DEV_GPU_UTIL{gpu="0",UUID="GPU-..."} 0

### 4. Validate Prometheus Scraping

In the Prometheus UI, open:

http://localhost:9090

Check whether the DCGM Exporter target is up.

PromQL:

up{job="dcgm-exporter"}

Expected result:

1

If the job label is different, inspect all targets with:

up

and identify the target corresponding to DCGM Exporter.

## Viewing DCGM Metrics Directly in Prometheus

Once the exporter is successfully scraped, individual DCGM metrics can be queried directly from the Prometheus UI.

### GPU Utilization

DCGM_FI_DEV_GPU_UTIL

### GPU Temperature

DCGM_FI_DEV_GPU_TEMP

### GPU Power Usage

DCGM_FI_DEV_POWER_USAGE

### SM Clock

DCGM_FI_DEV_SM_CLOCK

### Framebuffer Memory Used

DCGM_FI_DEV_FB_USED

### Memory Copy Utilization

DCGM_FI_DEV_MEM_COPY_UTIL

### View All DCGM Metrics

{__name__=~"DCGM_.*"}

This is useful for confirming exactly which DCGM metrics Prometheus has received.

## Stronger End-to-End Test

Simply seeing a metric is not enough to prove that telemetry is updating correctly.

A stronger test is to open:

DCGM_FI_DEV_GPU_UTIL

in Prometheus Graph view, then generate GPU workload.

The expected behavior is:

GPU idle → low utilization

GPU workload → utilization increases

Workload stops → utilization decreases

This validates that the metric is not merely present but is actively being updated through the monitoring pipeline.

## Cross-Check Against nvidia-smi

The exporter values can be compared with:

nvidia-smi --query-gpu=utilization.gpu,temperature.gpu,power.draw --format=csv

and:

curl -s http://localhost:9400/metrics | \

grep -E 'DCGM_FI_DEV_GPU_UTIL|DCGM_FI_DEV_GPU_TEMP|DCGM_FI_DEV_POWER_USAGE'

The values do not have to be exactly identical at the same instant because sampling/caching intervals can differ, but they should be reasonably consistent.

## Final Validation Criteria

The DCGM → Prometheus pipeline can be considered working correctly when all of the following are true:

1. `nvidia-smi` detects the GPU.
2. DCGM detects the GPU, if `dcgmi` is installed.
3. `http://localhost:9400/metrics` responds successfully.
4. DCGM metrics such as `DCGM_FI_DEV_GPU_UTIL` exist on the exporter endpoint.
5. Prometheus reports the DCGM Exporter target as `UP`.
6. `up{job="dcgm-exporter"}` returns `1` when that job label is used.
7. Prometheus can query `DCGM_FI_DEV_GPU_UTIL` and other DCGM metrics.
8. Metrics change appropriately when GPU workload is generated.

## Final Result

The recommended workflow for validating the monitoring setup is:

1. nvidia-smi

       ↓

2. dcgmi discovery -l

       ↓

3. curl http://localhost:9400/metrics

       ↓

4. Prometheus: up{job="dcgm-exporter"}

       ↓

5. Prometheus: DCGM_FI_DEV_GPU_UTIL

       ↓

6. Generate GPU load and observe metric changes

The key Prometheus query for directly viewing GPU utilization is:

DCGM_FI_DEV_GPU_UTIL

And the broad query for discovering all DCGM metrics available in Prometheus is:

{__name__=~"DCGM_.*"}

This establishes whether DCGM Exporter is not only running, but successfully exposing GPU telemetry to Prometheus for use by the Grafana GPU dashboard.