# Personal Laptop Observability --- Conversation Summary

## 1. Project Goal

The project is to build a comprehensive Grafana observability setup for
a personal Ubuntu laptop.

The monitoring stack is centered around:

-   **Grafana 13.1.3** --- visualization and dashboards
-   **Prometheus** --- metrics collection/querying
-   **Loki 3.7.6** --- log aggregation
-   **Alloy** --- log collection/relabeling and forwarding to Loki
-   **node-exporter** --- host CPU, memory, filesystem, disk, network,
    and system metrics
-   **process-exporter** --- per-process/group CPU and memory metrics
-   **cAdvisor** --- Docker/container resource metrics
-   **DCGM exporter 4.8.0-4.7.2-ubuntu22.04** --- NVIDIA GPU metrics

The laptop hardware discussed:

-   Intel Core i7-10750H
-   6 cores / 12 threads
-   32 GB DDR4 RAM
-   NVIDIA GeForce GTX 1650 Ti
-   4 GB VRAM
-   NVMe SSD mounted at `/`
-   SATA SSD mounted at `/home`
-   Ubuntu 26.04

------------------------------------------------------------------------

# 2. Main Problems Faced

## 2.1 Node Exporter connectivity

An early problem was:

``` text
Error scraping target: Get "http://localhost:9100/metrics":
dial tcp [::1]:9100: connect: connection refused
```

There was also confusion because opening the exporter URL resulted in a
Docker/container hostname such as:

``` text
http://4ab0e87f21ad:9100/metrics
```

The underlying issue was that Prometheus/Grafana were running in
containers, so `localhost` referred to the container itself rather than
the host machine.

The working configuration used the host-accessible address and aligned
the Prometheus target with what the containerized monitoring stack could
actually reach.

------------------------------------------------------------------------

# 3. Process Exporter Problems

## 3.1 Repeated process metrics

Process-exporter produced repeated entries for processes such as Python,
and browser processes such as Brave appeared to have very large memory
values.

Example metric:

``` text
namedprocess_namegroup_memory_bytes{
  groupname="brave",
  instance="process-exporter:9256",
  job="process",
  memtype="proportionalResident"
}
```

The important distinction was between different process memory
measurements.

Process-exporter exposes multiple memory types, and values such as
virtual memory can be extremely large without representing actual RAM
consumption.

For dashboards, **resident/proportional resident memory** is more useful
for representing physical memory pressure than virtual memory alone.

The process dashboard therefore focuses on grouping processes and
selecting appropriate memory metrics rather than blindly displaying
every process-exporter series.

------------------------------------------------------------------------

# 4. Process Exporter AppArmor / ptrace Problem

A kernel audit message appeared:

``` text
audit: type=1400 audit(...):
apparmor="DENIED"
operation="ptrace"
class="ptrace"
profile="docker-default"
pid=4304
comm="process-exporte"
requested_mask="read"
denied_mask="read"
peer="unconfined"
```

This indicated that the process-exporter container was running under
Docker's default AppArmor profile and was being prevented from
performing the required `ptrace` operation.

The important conclusion was:

> The exporter itself could be running, but its visibility into host
> processes was restricted by the container security profile.

The solution direction was to give process-exporter the host-level
access it requires, rather than assuming that ordinary container access
is sufficient.

This is different from cAdvisor, which was able to collect its required
metrics without the same level of process inspection access.

------------------------------------------------------------------------

# 5. Alloy / Loki Problems

The Loki/Alloy pipeline also had configuration issues.

An Alloy configuration included:

``` alloy
loki.relabel "journal_relabel" {
  forward_to = [loki.write.local.receiver]

  rule {
    action        = "replace"
    source_labels = ["__journal__systemd_unit"]
    regex         = "unit"
  }

  rule {
    action        = "replace"
    source_labels = ["__journal_syslog_identifier__"]
    regex         = "syslog_identifier"
  }
}
```

Problems included Alloy reporting:

``` text
could not perform the initial load successfully
```

and Loki returning no data for the expected labels.

The troubleshooting focused on:

-   Correct Alloy relabel syntax
-   Correct journal label names
-   Ensuring relabeled entries were actually forwarded to Loki
-   Verifying that Loki was ready to receive data
-   Separating ingestion/configuration problems from Grafana query
    problems

Loki was also reporting readiness/ring-related issues during setup. The
configuration was being developed for a local, single-node laptop
monitoring environment rather than a distributed Loki deployment.

------------------------------------------------------------------------

# 6. Loki Design

The intended architecture is:

``` text
systemd journal
       │
       ▼
     Alloy
       │
       ▼
Loki (:3100)
       │
       ▼
Grafana
```

The goal is to expose useful labels such as:

-   systemd unit
-   syslog identifier
-   priority
-   hostname/host information where useful

The design avoids creating unnecessarily high-cardinality labels.

------------------------------------------------------------------------

# 7. NVIDIA / GPU Monitoring

The laptop contains a:

``` text
NVIDIA GeForce GTX 1650 Ti
4 GB VRAM
```

NVIDIA driver information showed approximately:

``` text
NVIDIA-SMI 595.84
Driver Version: 595.84
CUDA Version: 13.2
```

GPU memory usage at the time of checking was approximately:

``` text
32 MiB / 4096 MiB
```

The DCGM exporter was:

``` text
dcgm-exporter:4.8.0-4.7.2-ubuntu22.04
```

The available metrics included metrics such as:

``` text
DCGM_FI_DEV_DEC_UTIL
```

and other `DCGM_FI_DEV_*` metrics.

A key correction during dashboard design was:

> Do not assume a `PROC` metric exists in DCGM exporter output.

The actual exposed metric set must be used when designing the GPU
dashboard.

------------------------------------------------------------------------

# 8. CPU Dashboard Problems

The CPU dashboard used Prometheus metrics such as:

``` promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
```

and a CPU utilization calculation:

``` promql
100 * (
  1 - rate(node_cpu_seconds_total{mode="idle"}[5m])
)
```

Per-core metrics returned labels similar to:

``` text
cpu="0"
cpu="1"
cpu="2"
...
cpu="10"
cpu="11"
```

A Grafana presentation problem occurred because string-based sorting can
produce:

``` text
0
1
10
11
2
3
...
```

rather than numeric order.

The dashboard therefore needs a Grafana transformation/display approach
to force CPU labels into numeric order.

------------------------------------------------------------------------

# 9. Docker Monitoring

cAdvisor was used to monitor containers.

The Docker dashboard was designed around:

-   Container CPU
-   Container RAM
-   Container network traffic
-   Container status
-   Container restarts
-   Individual container resource usage

The user also wanted thresholds to depend on the laptop's total
available RAM.

A complication was discovered:

> PromQL expressions cannot simply treat metrics from unrelated
> exporters as if they were scalar variables.

For example, trying to use a total-memory query from one exporter
directly as a dynamic threshold for a container metric requires correct
PromQL vector matching/scalar handling.

The general solution is to calculate the desired percentage/limit using
PromQL itself, or use Grafana transformations/threshold configuration
where appropriate.

------------------------------------------------------------------------

# 10. Memory Dashboard

The memory dashboard focuses on:

-   Total RAM
-   Used RAM
-   Available RAM
-   RAM utilization percentage
-   Swap usage
-   Swap activity
-   Cached memory
-   Memory pressure

A core RAM utilization query was:

``` promql
100 * (
  1 -
  node_memory_MemAvailable_bytes /
  node_memory_MemTotal_bytes
)
```

This uses `MemAvailable` rather than simply subtracting free memory,
making it more representative of memory pressure on Linux.

------------------------------------------------------------------------

# 11. Storage Dashboard

The laptop has separate storage areas, notably:

``` text
/
 /home
```

The dashboard is intended to show:

-   Filesystem usage
-   Available space
-   Total space
-   Disk read throughput
-   Disk write throughput
-   IOPS
-   Disk utilization
-   Disk latency

A filesystem usage calculation used:

``` promql
100 * (
  1 -
  node_filesystem_avail_bytes{
    mountpoint="/",
    fstype!~"tmpfs|overlay"
  }
  /
  node_filesystem_size_bytes{
    mountpoint="/",
    fstype!~"tmpfs|overlay"
  }
)
```

------------------------------------------------------------------------

# 12. Network Dashboard

The planned network monitoring includes:

-   Download throughput
-   Upload throughput
-   Total received
-   Total transmitted
-   Packet errors
-   Dropped packets

Example receive throughput:

``` promql
rate(node_network_receive_bytes_total[5m]) * 8
```

Example transmit throughput:

``` promql
rate(node_network_transmit_bytes_total[5m]) * 8
```

The multiplication by `8` converts bytes/sec into bits/sec.

------------------------------------------------------------------------

# 13. Overall Dashboard Architecture

The final dashboard concept is not to put every metric onto one huge
screen.

Instead, use a high-level overview dashboard plus detailed dashboards.

## Overview

``` text
┌──────────────────────────────────────────────────────┐
│                 LAPTOP OVERVIEW                      │
│ CPU │ RAM │ GPU │ VRAM │ DISK │ LOAD │ UPTIME       │
├──────────────────────────────────────────────────────┤
│                    CPU                               │
│ Total CPU                    Per-core CPU            │
│ CPU frequency                Temperature             │
├──────────────────────────────────────────────────────┤
│                    MEMORY                            │
│ RAM usage                    Swap                    │
├──────────────────────────────────────────────────────┤
│                     GPU                              │
│ GPU utilization              VRAM                    │
│ Temperature                  Power                   │
├──────────────────────────────────────────────────────┤
│                 STORAGE / NETWORK                    │
│ Disk I/O                     Network traffic         │
├──────────────────────────────────────────────────────┤
│                   PROCESSES                          │
│ Top CPU processes            Top memory processes   │
└──────────────────────────────────────────────────────┘
```

Detailed dashboards can then cover:

1.  CPU
2.  Memory & Storage
3.  GPU
4.  Docker / Containers
5.  Processes
6.  Logs / Loki

------------------------------------------------------------------------

# 14. Final Monitoring Philosophy

The dashboard is intended to answer two levels of questions.

## Level 1 --- "Is my laptop healthy?"

The Overview dashboard should immediately show:

-   CPU utilization
-   RAM utilization
-   GPU utilization
-   GPU VRAM usage
-   Disk usage
-   Load
-   Uptime

## Level 2 --- "Why is it unhealthy?"

Detailed dashboards should allow investigation into:

-   Which CPU cores are busy?
-   Which process is consuming CPU?
-   Which process is consuming RAM?
-   Is the GPU saturated?
-   Is VRAM exhausted?
-   Is the disk overloaded?
-   Is network traffic unusually high?
-   Which Docker container is consuming resources?
-   What systemd/kernel/application logs explain the problem?

This creates a proper observability workflow:

``` text
Overview
   │
   ├── CPU ───────► processes / temperature / load
   │
   ├── Memory ────► processes / swap
   │
   ├── GPU ───────► utilization / VRAM / temperature
   │
   ├── Disk ──────► I/O / filesystem
   │
   ├── Docker ────► container resources
   │
   └── Logs ──────► Loki / Alloy / systemd journal
```

------------------------------------------------------------------------

# 15. Current Final State

The project has evolved from simply collecting metrics into designing a
complete personal-laptop observability system.

The main working/target architecture is:

``` text
                         ┌──────────────┐
                         │   Grafana    │
                         │    :3000     │
                         └──────┬───────┘
                                │
                  ┌─────────────┴─────────────┐
                  │                           │
           ┌──────▼──────┐             ┌──────▼──────┐
           │ Prometheus  │             │    Loki     │
           │    :9090    │             │    :3100    │
           └──────┬──────┘             └──────▲──────┘
                  │                           │
       ┌──────────┼────────────┐              │
       │          │            │              │
       ▼          ▼            ▼              │
 node-exporter process-    DCGM exporter     │
              exporter                       │
       │          │            │              │
       └──────────┴────────────┘              │
                  │                           │
             Host metrics                     │
                                              │
                         systemd journal ─► Alloy
```

Docker/container metrics are provided by cAdvisor and are incorporated
into the Prometheus side of the architecture.

The overall goal is a clean, practical observability system for a
developer workstation rather than a generic server dashboard.

The next logical step is to finish each Grafana dashboard
panel-by-panel, defining for every panel:

-   Visualization type
-   PromQL query
-   Unit
-   Legend
-   Field transformations
-   Thresholds
-   Value mappings
-   Panel title
-   Appropriate time range
