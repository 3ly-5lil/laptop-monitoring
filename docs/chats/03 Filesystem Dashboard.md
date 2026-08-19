# Filesystem Dashboard — Conversation Summary

## Overview

This conversation focused on building and refining the **Filesystem / Disk observability dashboard** in Grafana for a laptop monitored with Prometheus and Node Exporter.

The goal was to create a compact dashboard that clearly shows:

- Filesystem capacity usage
- Disk read/write throughput
- Disk IOPS
- Disk latency
- Disk I/O utilization

The dashboard is intended to help identify storage bottlenecks during workloads such as Docker builds, Maven builds, Kubernetes, Prometheus/Loki activity, local LLM workloads, and VM workloads.

---

## Initial Dashboard Design

The proposed filesystem section was initially structured around:

1. Filesystem usage for `/`, `/home`, Docker, and other storage.
2. Disk read/write IOPS and throughput.
3. Disk latency/utilization.

The important distinction was that **Docker should not automatically be treated as a separate filesystem**. If Docker stores data under `/var/lib/docker`, it is consuming the filesystem containing that path rather than necessarily being a separate filesystem.

A Docker-specific storage panel can be added later to measure Docker storage consumption separately.

---

## Problem / Constraint: `$instance`

The initial PromQL examples used an `$instance` Grafana variable.

Because this dashboard monitors a single laptop, the variable was unnecessary and was removed.

For example, instead of:

```promql
node_disk_read_bytes_total{
  instance=~"$instance",
  device=~"nvme[0-9]+|sd[a-z]+"
}
```

the dashboard uses:

node_disk_read_bytes_total{

  device=~"nvme[0-9]+|sd[a-z]+"

}

This makes the queries simpler and avoids unnecessary dashboard-variable filtering.

# Final PromQL

## 1. Filesystem Usage

The filesystem percentage is calculated from available bytes and total filesystem size:

100 *

(

  1 -

  node_filesystem_avail_bytes{

    fstype!~"tmpfs|overlay|squashfs|vfat|efivarfs"

  }

  /

  node_filesystem_size_bytes{

    fstype!~"tmpfs|overlay|squashfs|vfat|efivarfs"

  }

)

### Purpose

Shows filesystem capacity utilization while excluding pseudo-filesystems and other filesystems that are not useful for the main storage view.

The resulting dashboard currently shows:

- `/` — approximately 63.9%
- `/home` — approximately 87.5%

The `/home` filesystem is notably close to full and should be monitored.

### Recommended improvement

Show both percentage and absolute usage, for example:

/          63.9%

148 GB / 414 GB

  

/home      87.5%

~350 GB / ~400 GB

This gives more context than percentage alone.

---

## 2. Disk Read Throughput

rate(node_disk_read_bytes_total{

  device=~"nvme[0-9]+|sd[a-z]+"

}[5m])

## 3. Disk Write Throughput

rate(node_disk_written_bytes_total{

  device=~"nvme[0-9]+|sd[a-z]+"

}[5m])

### Recommended visualization

Use a **Time series** panel titled:

**Disk Throughput**

Unit:

bytes/sec (B/s)

The dashboard can show both NVMe and SATA devices, with Read/Write separated in the legend.

---

# Disk IOPS

## Read IOPS

rate(node_disk_reads_completed_total{

  device=~"nvme[0-9]+|sd[a-z]+"

}[5m])

## Write IOPS

rate(node_disk_writes_completed_total{

  device=~"nvme[0-9]+|sd[a-z]+"

}[5m])

### Recommended visualization

Use a **Time series** panel titled:

**Disk IOPS**

Unit:

ops/sec

The current dashboard shows read/write activity for both:

- `nvme0n1`
- `sda`

---

# Disk Latency

The dashboard calculates average I/O latency from total I/O time divided by completed operations.

## Read Latency

Current query:

1000 *

rate(node_disk_read_time_seconds_total{

  device=~"nvme0n[0-9]+|sd[a-z]+"

}[5m])

/

rate(node_disk_reads_completed_total{

  device=~"nvme0n[0-9]+|sd[a-z]+"

}[5m])

Multiplication by `1000` converts seconds to milliseconds.

## Write Latency

The equivalent write query is:

1000 *

rate(node_disk_write_time_seconds_total{

  device=~"nvme0n[0-9]+|sd[a-z]+"

}[5m])

/

rate(node_disk_writes_completed_total{

  device=~"nvme0n[0-9]+|sd[a-z]+"

}[5m])

### Device filtering

The final dashboard specifically uses a regex matching the laptop's storage devices:

nvme0n[0-9]+|sd[a-z]+

This matches devices such as:

nvme0n1

sda

and avoids unrelated devices such as loop, RAM, or optical devices.

### Recommended visualization

Use two panels:

- **NVMe Latency**
- **SATA Latency**

Each panel shows:

- Read
- Write

Unit:

milliseconds (ms)

Suggested latency thresholds:

|Latency|State|
|---|---|
|< 1 ms|Good|
|1–5 ms|Normal|
|5–20 ms|High|
|> 20 ms|Critical|

The observed values in the current dashboard were approximately:

NVMe Read:   1.80 ms

NVMe Write:  2.83 ms

  

SATA Read:   1.67 ms

SATA Write:  0.834 ms

These values were not considered alarming.

---

# Disk I/O Utilization

The final utilization query is:

100 *

rate(node_disk_io_time_seconds_total{

  device=~"nvme[0-9]+|sd[a-z]+"

}[5m])

### Purpose

Shows how busy each physical disk is from an I/O perspective.

This is different from filesystem capacity utilization.

For example:

Filesystem utilization:

"/ is 64% full"

  

Disk I/O utilization:

"nvme0n1 is currently 80% busy"

These answer completely different questions.

### Recommended visualization

Use a **Time series** panel titled:

**Disk I/O Utilization**

Unit:

Percent (0-100)

Y-axis:

0–100%

---

# Final Dashboard Layout

The dashboard was refined into a compact two-row layout.

┌──────────────────────┬────────────────────────┬──────────────────────┐

│ Filesystem Usage     │ Disk Throughput        │ Disk IOPS            │

│                      │                        │                      │

│ /          63.9%     │ Read / Write           │ Read / Write          │

│ 148 / 414 GB         │ NVMe + SATA            │ NVMe + SATA           │

│                      │                        │                      │

│ /home      87.5%     │                        │                      │

│ ~350 / ~400 GB       │                        │                      │

└──────────────────────┴────────────────────────┴──────────────────────┘

  

┌──────────────────────┬────────────────────────┬──────────────────────┐

│ NVMe Latency         │ SATA Latency            │ Disk I/O Utilization │

│                      │                        │                      │

│ Read      1.80 ms    │ Read       1.67 ms     │ nvme0n1    ~0%       │

│ Write     2.83 ms    │ Write      0.83 ms     │ sda        ~0%       │

│                      │                        │                      │

└──────────────────────┴────────────────────────┴──────────────────────┘

## Final panel names

1. **Filesystem Usage**
2. **Disk Throughput**
3. **Disk IOPS**
4. **NVMe Latency**
5. **SATA Latency**
6. **Disk I/O Utilization**

These names are preferred over less descriptive names such as:

- `Disk Read / Write`
- `Latency On nvme01`
- `Latency On sda`

because they make the dashboard easier to scan.

---

# Final Dashboard Recommendations

## Keep

- Two-row layout
- Three panels in the first row
- Three panels in the second row
- NVMe/SATA separation for latency
- Read/Write separation
- Time-series visualization for throughput, IOPS, and utilization
- Percentage visualization for filesystem capacity

## Change

### Filesystem Usage

Add absolute capacity information in addition to percentage.

Example:

/          63.9%

148 GB / 414 GB

  

/home      87.5%

350 GB / 400 GB

### Panel names

Use:

Filesystem Usage

Disk Throughput

Disk IOPS

NVMe Latency

SATA Latency

Disk I/O Utilization

### Default time range

A **Last 1 hour** default was recommended instead of Last 6 hours.

Reason:

- Filesystem capacity is mostly a current-state metric.
- Throughput is easier to read over a shorter interval.
- IOPS spikes are more visible.
- Latency changes are easier to identify.
- Disk utilization spikes are easier to correlate with activity.

Longer ranges such as 6h or 24h can still be selected manually for investigations.

---

# Key Design Decision

The dashboard intentionally avoids creating a separate generic "Docker filesystem" panel.

If Docker uses:

/var/lib/docker

then its storage is part of whichever filesystem contains that directory.

A future **Docker Storage** panel should instead measure Docker-specific storage consumption, rather than duplicating filesystem capacity information.

---

# Final Assessment

The dashboard structure is considered strong and compact.

The main remaining improvement is to show **absolute filesystem usage alongside percentages**, especially because `/home` is already around **87.5% full**.

The final dashboard focuses on the questions that matter during storage-heavy workloads:

1. **How full is my storage?**
2. **Which disk is reading/writing?**
3. **How many I/O operations are happening?**
4. **How long do I/O operations take?**
5. **Is the physical disk actually busy?**

This makes the dashboard useful for diagnosing situations where CPU and RAM look normal but storage performance is the actual bottleneck.