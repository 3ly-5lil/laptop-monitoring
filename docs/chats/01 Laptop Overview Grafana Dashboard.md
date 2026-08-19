# Laptop Overview Grafana Dashboard — Conversation Summary

## 1. Goal

Build a clean **Grafana Laptop Overview dashboard** that answers:

> **How is my laptop doing right now?**

The dashboard is intended to provide a quick health overview first, followed by CPU diagnostics. The monitoring stack already includes Prometheus, Grafana, node-exporter, process-exporter, cAdvisor, Loki, and NVIDIA DCGM exporter.

---

# 2. Dashboard Architecture

## Current Health

The intended top section is a set of Stat panels:

- CPU Usage
- RAM Usage
- GPU Usage
- Disk Usage
- Load
- CPU Temperature
- GPU Temperature
- Uptime

The current dashboard screenshot contains these panels and is already visually clean.

## CPU

The second section contains:

- CPU Utilization
- CPU Usage by Mode
- CPU Load — 1m / 5m / 15m
- CPU Frequency

The dashboard is intended to help detect:

- CPU saturation
- thermal throttling
- background workloads
- compilation workloads
- Docker workloads
- VM workloads

---

# 3. Problem: `label_values()` PromQL Error

## Problem

An initial dashboard-variable suggestion used:

```promql
label_values(node_uname_info, instance)
```

Grafana returned:

invalid parameter "query": 1:1: parse error: unknown function with name "label_values"

## Cause

`label_values()` is **not a PromQL function**. It is associated with Grafana's variable-query syntax and should not be entered into a normal Prometheus PromQL editor.

## Solution

For a Grafana Prometheus variable, use the variable editor's **Label values** query type and select:

- Metric: `node_uname_info`
- Label: `instance`

If the variable editor only provides a PromQL-style query box, a fallback is:

query_result(node_uname_info)

with an appropriate regex to extract the instance label.

## Final decision

Because this is currently a single-laptop monitoring setup, an `instance` variable is not necessary yet. It is cleaner to first build the dashboard against the actual exporter instance and introduce variables later if multiple machines are monitored.

---

# 4. CPU Usage

## Stat query

100 * (

  1 - avg(

    rate(node_cpu_seconds_total{mode="idle"}[5m])

  )

)

This calculates overall CPU utilization as:

100 × (1 − idle CPU percentage)

## Time-series query

100 * (

  1 -

  avg(

    rate(node_cpu_seconds_total{mode="idle"}[5m])

  )

)

Recommended visualization:

- Visualization: Time series
- Unit: Percent (0-100)
- Minimum: 0
- Maximum: 100
- Legend: CPU Usage

Recommended thresholds:

- 0–70%: normal
- 70–90%: warning
- 90–100%: critical

---

# 5. RAM Usage

Use `MemAvailable` rather than `MemFree`, because Linux uses free memory for cache.

100 * (

  1 - (

    node_memory_MemAvailable_bytes

    /

    node_memory_MemTotal_bytes

  )

)

Recommended unit:

Percent (0-100)

Suggested thresholds:

- 0–70%: normal
- 70–85%: warning
- 85–95%: high
- 95–100%: critical

---

# 6. GPU Usage

The NVIDIA DCGM exporter provides the GPU utilization metric:

DCGM_FI_DEV_GPU_UTIL

For a single GPU:

max(DCGM_FI_DEV_GPU_UTIL)

Recommended:

- Visualization: Stat
- Unit: Percent (0-100)

The actual labels exposed by the DCGM exporter should be inspected before adding unnecessary label filtering.

---

# 7. Disk Usage

For the root filesystem:

100 * (

  1 -

  node_filesystem_avail_bytes{

    mountpoint="/",

    fstype!="tmpfs"

  }

  /

  node_filesystem_size_bytes{

    mountpoint="/",

    fstype!="tmpfs"

  }

)

The laptop has separate storage locations, including `/` and `/home`.

## Final dashboard recommendation

Instead of titles such as:

Disk Usage (nvme0n1)

Disk Usage (sda)

use human-oriented titles:

Root Disk (/)

Home Disk (/home)

The dashboard screenshot showed approximately:

- Root/NVMe: 63.6%
- Home/SATA: 87.5%

The 87.5% value is useful as a warning because the home disk is getting relatively full.

---

# 8. Load

Available node-exporter metrics:

node_load1

node_load5

node_load15

The CPU Load panel uses all three:

### 1 minute

node_load1

Legend:

1m

### 5 minutes

node_load5

Legend:

5m

### 15 minutes

node_load15

Legend:

15m

## Important visualization fix

The dashboard initially showed the load series as a **stacked area chart**.

That is misleading because stacking makes the displayed height represent:

1m + 5m + 15m

rather than each load value independently.

### Final solution

Set:

Stack series = Off

The three load values should be displayed independently on the same axis.

---

# 9. CPU Usage by Mode

The panel uses:

100 *

sum by (mode) (

  rate(

    node_cpu_seconds_total[5m]

  )

)

Useful modes include:

- `user`
- `system`
- `iowait`
- `idle`

A filtered version is:

100 *

sum by (mode) (

  rate(

    node_cpu_seconds_total{

      mode=~"user|system|iowait|idle"

    }[5m]

  )

)

## Important dashboard issue

The screenshot showed:

idle       90.4%

iowait      0.1%

system      2.5%

user        7.0%

but `idle` was displayed in red.

That is incorrect semantically: **high idle CPU usage is good**, not bad.

## Final recommendation

Do not apply generic high-value-is-bad thresholds to this panel.

Better options:

1. Remove thresholds entirely.
2. Give CPU modes distinct semantic colors.
3. Prefer a Time series visualization if historical behavior is more important than the current snapshot.

---

# 10. CPU Temperature

The laptop exposes multiple `coretemp` readings through node-exporter's hwmon metrics.

Example query:

node_hwmon_temp_celsius{chip=~".*coretemp.*"}

## Why there were 7 sensors

The Intel Core i7-10750H has 6 physical cores.

The Linux `coretemp` driver can expose:

- CPU/package temperature
- Core 0
- Core 1
- Core 2
- Core 3
- Core 4
- Core 5

Therefore, seeing approximately seven `coretemp` temperature channels is normal.

The exact mapping should be verified from the metric labels rather than assumed from sensor numbering.

## Dashboard recommendation

The overview should not display seven temperature panels.

Instead, use the hottest CPU temperature:

max(

  node_hwmon_temp_celsius{

    chip=~".*coretemp.*"

  }

)

This is useful for detecting thermal problems because the hottest core/package value is more important than an average.

## Suggested thresholds

- <75°C: normal
- 75–90°C: warning
- > 90°C: critical
    

The current dashboard showed approximately:

CPU Temp = 58°C

which looks healthy.

---

# 11. GPU Temperature

The DCGM exporter provides:

DCGM_FI_DEV_GPU_TEMP

For a single GPU:

max(

  DCGM_FI_DEV_GPU_TEMP

)

Recommended unit:

Celsius (°C)

Suggested thresholds:

- <70°C: normal
- 70–85°C: warning
- > 85–90°C: critical
    

The current dashboard showed approximately:

GPU Temp = 44°C

which is healthy.

---

# 12. Uptime

Query:

time() - node_boot_time_seconds

Recommended unit:

seconds (s)

and configure Grafana to display it as a duration.

The dashboard currently showed:

3.45 days

## Important visualization fix

Uptime was displayed in red.

That is not useful because uptime is not inherently a high-is-bad metric.

### Final solution

Remove thresholds from the uptime Stat and use a neutral/positive presentation.

---

# 13. CPU Frequency

The dashboard uses node-exporter's CPU frequency metrics when available.

Current frequency:

avg(

  node_cpu_scaling_frequency_hertz

) / 1e9

Maximum frequency:

avg(

  node_cpu_scaling_frequency_max_hertz

) / 1e9

Unit:

GHz

The CPU Frequency panel compares current CPU frequency with maximum frequency.

This is especially useful on a laptop because it can help identify possible thermal throttling.

For example:

High CPU usage

+ High temperature

+ Low CPU frequency

can indicate thermal throttling.

---

# 14. Final Dashboard Layout

The current final layout is:

LAPTOP OVERVIEW

────────────────────────────────────────────────────────────

  

CURRENT HEALTH

  

┌────────────┬────────────┬────────────┬────────────┐

│ Disk /     │ Disk /home │ RAM        │ Uptime     │

├────────────┼────────────┼────────────┼────────────┤

│ CPU        │ CPU Temp   │ GPU        │ GPU Temp   │

└────────────┴────────────┴────────────┴────────────┘

  

  

CPU

  

┌──────────────────────────┬──────────────────────────┐

│ CPU Utilization          │ CPU Usage by Mode        │

└──────────────────────────┴──────────────────────────┘

  

┌──────────────────────────┬──────────────────────────┐

│ CPU Load                 │ CPU Frequency            │

│ 1m / 5m / 15m            │ Current / Max            │

└──────────────────────────┴──────────────────────────┘

The dashboard screenshot shows this layout and it is considered a strong v1.

---

# 15. Final Edits Recommended

Before moving on from the Overview dashboard:

### Must fix

- [x]  Disable stacking on CPU Load.
- [x]  Remove red thresholds from Uptime.
- [x]  Fix CPU Usage by Mode so high idle is not shown as a problem.
- [x]  Use the hottest CPU temperature rather than treating all seven `coretemp` sensors as separate overview metrics.
- [x]  Rename disks to `/` and `/home` rather than only showing device names.

### Keep

- [x]  CPU Stat
- [x]  RAM Stat
- [x]  GPU Stat
- [x]  Disk Stats
- [x]  CPU Temperature Stat
- [x]  GPU Temperature Stat
- [x]  Uptime Stat
- [x]  CPU Utilization time series
- [x]  CPU Load
- [x]  CPU Frequency

---

# 16. Design Decision

The Overview dashboard should remain a **10-second health dashboard** rather than becoming a wall of metrics.

The intended hierarchy is:

Current health

      ↓

CPU behavior

      ↓

Detailed investigation

Detailed dashboards can handle:

- Memory
- Disk I/O
- GPU
- Network
- Processes
- Docker / Containers
- Thermals
- Logs

This keeps the Laptop Overview clean and useful.

---

# 17. Next Dashboard Section

The next recommended row/dashboard section is **Memory**.

Recommended panels:

RAM Usage

Memory Available

Swap Usage

Memory Breakdown

After that, build **Disk I/O**, where the two SSDs and CPU `iowait` can be correlated.