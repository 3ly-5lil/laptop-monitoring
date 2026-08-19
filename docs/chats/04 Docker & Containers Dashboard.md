# Docker / Containers Dashboard — Conversation Summary

## 1. Goal

Build a dedicated Grafana **Docker / Containers Dashboard** using **cAdvisor** metrics, with:

- Top-level container health/resource statistics
- Current CPU and memory consumers
- CPU and memory usage over time
- Container network RX/TX
- Container disk I/O
- Dynamic RAM thresholds based on total host RAM

The dashboard is intended for a laptop observability stack using Prometheus, Grafana, cAdvisor, and Node Exporter.

---

# 2. Final Dashboard Layout

```text
┌────────────────┬────────────────┬────────────────┬────────────────┐
│   Containers   │ Container CPU  │    Host CPU    │ Container RAM  │
│      12        │      37%       │      3.1%      │     8.4 GiB    │
└────────────────┴────────────────┴────────────────┴────────────────┘

┌───────────────────────────────────────┬────────────────────────────┐
│          Container CPU               │      Container Memory       │
│                                       │                            │
│ prometheus ███████████  8.2%         │ prometheus █████  1.8 GiB  │
│ grafana    █████        4.1%         │ loki       ████   1.2 GiB  │
│ loki       ████         3.8%         │ grafana    ██      640 MiB  │
│ chrome     ██           2.1%         │ cadvisor   █       180 MiB  │
└───────────────────────────────────────┴────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    Container CPU Usage                              │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                   Container Memory Usage                            │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────┬──────────────────────────────────┐
│       Container Network          │       Container Disk I/O         │
│                                  │                                  │
│ RX ─────────────────────         │ Read ─────────────────────       │
│ TX ─────────────────             │ Write ───────────────────        │
└──────────────────────────────────┴──────────────────────────────────┘
```

# 3. Top Row

## 3.1 Containers

**Visualization:** Stat

**Query:**

count(count by (name) (

  container_start_time_seconds{name!=""}

))

**Unit:** None

**Purpose:** Count the currently visible/running containers represented by cAdvisor.

---

## 3.2 Container CPU

**Visualization:** Stat

**Query:**

sum(

  rate(container_cpu_usage_seconds_total{name!=""}[5m])

) * 100

**Unit:** Percent (0-100)

### CPU interpretation

The cAdvisor metric:

container_cpu_usage_seconds_total

is a counter containing accumulated CPU time.

Using:

rate(...[5m])

converts it into CPU usage in **CPU cores**.

Therefore:

0.08 = 8% of one CPU core

0.50 = 50% of one CPU core

1.00 = 100% of one CPU core

2.00 = 200% = two fully utilized CPU cores

The dashboard uses the convention:

> **100% Container CPU = one fully utilized CPU core.**

---

## 3.3 Host CPU

This represents the percentage of the **entire host CPU capacity** consumed by containers.

For the laptop, the host has 12 logical CPUs, but instead of hardcoding `12`, use Node Exporter to determine the CPU count dynamically.

**Query:**

100 *

sum(rate(container_cpu_usage_seconds_total{name!=""}[5m]))

/

scalar(

  count(count by (cpu) (

    node_cpu_seconds_total{mode="idle"}

  ))

)

**Unit:** Percent (0-100)

This means:

container CPU cores used

------------------------ × 100

total host logical CPUs

Example:

0.37 CPU cores / 12 CPUs × 100 ≈ 3.08%

---

## 3.4 Container RAM

**Visualization:** Stat

**Query:**

sum(

  container_memory_working_set_bytes{name!=""}

)

**Unit:** Bytes (IEC)

Example:

8.4 GiB

### Why `working_set_bytes`?

Use:

container_memory_working_set_bytes

because it is more useful for answering:

> How much memory is this container actively consuming?

It is preferred for this dashboard over the raw:

container_memory_usage_bytes

---

# 4. Current Container CPU Consumers

**Visualization:** Bar Gauge

**Query:**

topk(

  10,

  sum by (name) (

    rate(container_cpu_usage_seconds_total{name!=""}[5m])

  ) * 100

)

**Unit:** Percent (0-100)

**Legend:**

{{name}}

### Recommended settings

- Horizontal orientation
- Min: 0
- Max: 100
- Show values
- Sort descending
- Do not stack

Example:

prometheus    8.2%

grafana       4.1%

loki          3.8%

chrome        2.1%

docker        1.7%

---

# 5. Current Container Memory Consumers

**Visualization:** Bar Gauge

**Query:**

topk(

  10,

  sum by (name) (

    container_memory_working_set_bytes{name!=""}

  )

)

**Unit:** Bytes (IEC)

**Legend:**

{{name}}

Example:

prometheus    1.8 GiB

loki          1.2 GiB

grafana       640 MiB

cadvisor      180 MiB

alloy         120 MiB

---

# 6. Container CPU Usage Over Time

**Visualization:** Time Series

**Query:**

topk(

  10,

  sum by (name) (

    rate(container_cpu_usage_seconds_total{name!=""}[5m])

  ) * 100

)

**Unit:** Percent (0-100)

**Legend:**

{{name}}

### Recommended display

- Lines
- Low fill opacity
- No points
- No stacking
- Top 10 containers

Do not stack the series because the objective is to compare individual container CPU usage.

---

# 7. Container Memory Usage Over Time

**Visualization:** Time Series

**Query:**

topk(

  10,

  sum by (name) (

    container_memory_working_set_bytes{name!=""}

  )

)

**Unit:** Bytes (IEC)

**Legend:**

{{name}}

This panel helps identify:

- Memory leaks
- Gradual memory growth
- Sudden memory spikes
- Resource-heavy containers

---

# 8. Container Network RX/TX

**Visualization:** Time Series

## RX

topk(

  10,

  sum by (name) (

    rate(container_network_receive_bytes_total{

      name!="",

      interface!="lo"

    }[5m])

  )

)

## TX

topk(

  10,

  sum by (name) (

    rate(container_network_transmit_bytes_total{

      name!="",

      interface!="lo"

    }[5m])

  )

)

**Unit:** Bytes/sec (IEC)

**Legends:**

{{name}} RX

{{name}} TX

Exclude:

interface="lo"

so loopback traffic does not pollute the network statistics.

---

# 9. Container Disk I/O

**Visualization:** Time Series

## Read

topk(

  10,

  sum by (name) (

    rate(container_fs_reads_bytes_total{name!=""}[5m])

  )

)

## Write

topk(

  10,

  sum by (name) (

    rate(container_fs_writes_bytes_total{name!=""}[5m])

  )

)

**Unit:** Bytes/sec (IEC)

**Legends:**

{{name}} Read

{{name}} Write

### Important

The availability and usefulness of:

container_fs_reads_bytes_total

container_fs_writes_bytes_total

depends on the cAdvisor version/configuration. If they are absent, the disk panel should be adapted to the metrics actually exposed by cAdvisor rather than forcing these queries.

---

# 10. Dynamic RAM Threshold Problem

The goal was to make the Container RAM threshold depend on a **percentage of the host's maximum/total RAM**.

For example:

< 60%       → normal

60%–80%     → warning

> 80%       → critical

On a 32 GB machine this corresponds approximately to:

60% = 19.2 GB

80% = 25.6 GB

The desired behavior is that these thresholds automatically scale on a different machine.

---

# 11. Problem Encountered

The initial query attempted to combine cAdvisor and Node Exporter directly:

100 *

sum(container_memory_working_set_bytes{name!=""})

/

node_memory_MemTotal_bytes

The cAdvisor metric comes from one exporter/source, while:

node_memory_MemTotal_bytes

comes from Node Exporter.

PromQL **can combine metrics from different exporters**. The problem is not that Prometheus cannot access multiple exporters.

The issue is **PromQL vector matching**.

The two expressions have different labels, so a direct binary operation such as:

vector_A / vector_B

requires Prometheus to determine which series on the left corresponds to which series on the right.

For a single-host dashboard, the desired denominator is simply one scalar value: total host RAM.

---

# 12. Solution: `scalar()`

Convert the single Node Exporter result into a scalar:

100 *

sum(container_memory_working_set_bytes{name!=""})

/

scalar(node_memory_MemTotal_bytes)

This removes the label-matching problem.

Conceptually:

cAdvisor

container memory

      ↓

     sum()

      ↓

8.4 GiB

      ↓

     ÷

32 GiB from Node Exporter

      ↓

26.25%

The result can then be used as a percentage for thresholding.

---

# 13. Dynamic Container RAM Percentage

The percentage query is:

100 *

sum(

  container_memory_working_set_bytes{name!=""}

)

/

scalar(node_memory_MemTotal_bytes)

**Unit:** Percent (0-100)

Recommended thresholds:

Base:     0

Warning: 60

Critical: 80

This makes the threshold relative to the total host RAM instead of hardcoding RAM sizes.

---

# 14. Important Grafana Consideration

If the goal is to display:

8.4 GiB

but color the value based on:

26.25% of host RAM

then the displayed value and the threshold-driving value are different quantities.

A clean approach is to keep:

### Actual RAM

sum(container_memory_working_set_bytes{name!=""})

for the displayed value.

### RAM percentage

100 *

sum(container_memory_working_set_bytes{name!=""})

/

scalar(node_memory_MemTotal_bytes)

for the threshold logic.

If the specific Grafana visualization cannot use one query/field to drive thresholds for another field, use a transformation or a percentage-based Stat panel. The exact approach depends on the Grafana panel configuration.

---

# 15. Final PromQL Reference

## Containers

count(count by (name) (

  container_start_time_seconds{name!=""}

))

## Container CPU

sum(

  rate(container_cpu_usage_seconds_total{name!=""}[5m])

) * 100

## Host CPU percentage used by containers

100 *

sum(rate(container_cpu_usage_seconds_total{name!=""}[5m]))

/

scalar(

  count(count by (cpu) (

    node_cpu_seconds_total{mode="idle"}

  ))

)

## Container RAM

sum(

  container_memory_working_set_bytes{name!=""}

)

## Container RAM percentage of total host RAM

100 *

sum(

  container_memory_working_set_bytes{name!=""}

)

/

scalar(node_memory_MemTotal_bytes)

## Top container CPU

topk(

  10,

  sum by (name) (

    rate(container_cpu_usage_seconds_total{name!=""}[5m])

  ) * 100

)

## Top container RAM

topk(

  10,

  sum by (name) (

    container_memory_working_set_bytes{name!=""}

  )

)

## Container CPU over time

topk(

  10,

  sum by (name) (

    rate(container_cpu_usage_seconds_total{name!=""}[5m])

  ) * 100

)

## Container memory over time

topk(

  10,

  sum by (name) (

    container_memory_working_set_bytes{name!=""}

  )

)

## Container network RX

topk(

  10,

  sum by (name) (

    rate(container_network_receive_bytes_total{

      name!="",

      interface!="lo"

    }[5m])

  )

)

## Container network TX

topk(

  10,

  sum by (name) (

    rate(container_network_transmit_bytes_total{

      name!="",

      interface!="lo"

    }[5m])

  )

)

## Container disk read

topk(

  10,

  sum by (name) (

    rate(container_fs_reads_bytes_total{name!=""}[5m])

  )

)

## Container disk write

topk(

  10,

  sum by (name) (

    rate(container_fs_writes_bytes_total{name!=""}[5m])

  )

)

---

# 16. Final Dashboard Design

|Row|Panel|Visualization|Unit|
|---|---|---|---|
|Top|Containers|Stat|None|
|Top|Container CPU|Stat|Percent (0-100)|
|Top|Host CPU|Stat|Percent (0-100)|
|Top|Container RAM|Stat|Bytes (IEC)|
|Row 2|Container CPU|Bar Gauge|Percent (0-100)|
|Row 2|Container Memory|Bar Gauge|Bytes (IEC)|
|Row 3|Container CPU Usage|Time Series|Percent (0-100)|
|Row 4|Container Memory Usage|Time Series|Bytes (IEC)|
|Row 5|Container Network RX/TX|Time Series|Bytes/sec|
|Row 5|Container Disk Read/Write|Time Series|Bytes/sec|

## Core lessons

1. **CPU is a counter** → use `rate()` or `irate()`.
2. **RAM is a gauge** → use it directly.
3. cAdvisor and Node Exporter metrics **can be combined in PromQL**.
4. Direct binary operations can fail because of **label/vector matching**.
5. For a single host, use `scalar()` when the denominator is a single value.
6. Dynamic RAM thresholds are easiest when RAM usage is converted to **percentage of `node_memory_MemTotal_bytes`**.
7. For container CPU, `100%` represents **one fully utilized CPU core** in the chosen dashboard convention.
8. For host CPU percentage, divide container CPU cores by the total logical CPU count.