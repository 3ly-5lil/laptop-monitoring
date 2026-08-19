# Home Laptop Observability — Grafana Dashboard Setup Summary

## 1. Final Architecture

The observability stack is organized as:

```text
                         ┌─────────────────────┐
                         │       Grafana       │
                         │        :3000        │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
             ┌──────▼──────┐                 ┌──────▼──────┐
             │ Prometheus  │                 │    Loki     │
             │    :9090    │                 │    :3100    │
             └──────┬──────┘                 └──────▲──────┘
                    │                               │
        ┌───────────┼────────────┐                  │
        │           │            │                  │
   Node Exporter  Process     DCGM Exporter      Alloy
                  Exporter
        │           │            │                  │
        └───────────┴────────────┴──────────────────┘
                            │
                     Ubuntu Laptop
```

Grafana uses:

- Prometheus for metrics
- Loki for logs

Prometheus collects metrics from:

- Node Exporter
- Process Exporter
- cAdvisor
- NVIDIA DCGM Exporter

Alloy collects logs and sends them to Loki.

---

## 2. Goal

The goal was to create a unified Grafana dashboard for laptop observability, covering:

- CPU
- Memory
- Storage
- Network
- NVIDIA GPU
- Processes
- Logs

The approach was to build the dashboard section-by-section using the actual metrics available from the user's exporters.

---

# 3. CPU Dashboard

CPU monitoring was started with Node Exporter.

### Total CPU usage

100 * (1 - avg by (instance) (

  rate(node_cpu_seconds_total{mode="idle"}[5m])

))

Recommended visualization:

- Gauge for current CPU usage
- Time series for CPU usage over time

Settings:

- Unit: Percent (0-100)
- Min: 0
- Max: 100
- Thresholds:
    - 70 = Warning
    - 90 = Critical

### Per-core CPU

100 * (1 - rate(node_cpu_seconds_total{mode="idle"}[5m]))

Legend:

CPU {{cpu}}

This exposes individual logical CPU utilization.

---

# 4. Memory Dashboard

### Memory usage

100 * (

  1 -

  node_memory_MemAvailable_bytes

  /

  node_memory_MemTotal_bytes

)

Recommended:

- Gauge for current memory usage
- Time series for history

Unit:

- Percent (0-100)

Thresholds:

- 70 = Warning
- 90 = Critical

### Memory used

node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

Unit:

- Bytes (IEC)

### Memory breakdown

Used:

node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

Available:

node_memory_MemAvailable_bytes

Cached:

node_memory_Cached_bytes

Buffers:

node_memory_Buffers_bytes

Unit:

- Bytes (IEC)

### Swap

A safe swap query was discussed so a system without swap does not produce misleading results:

100 *

(

  1 -

  node_memory_SwapFree_bytes

  /

  node_memory_SwapTotal_bytes

)

and

node_memory_SwapTotal_bytes > 0

---

# 5. Storage Dashboard

The actual filesystem layout was verified from Node Exporter.

## Actual storage layout

/dev/nvme0n1p1  → /          → ext4 → 501809635328 bytes

/dev/nvme0n1p2  → /boot/efi → vfat → 1124999168 bytes

/dev/sda1       → /home     → ext4 → 491107536896 bytes

Temporary filesystems were also visible:

tmpfs → /run

tmpfs → /run/snapd/ns

tmpfs → /run/user/1000

tmpfs → /tmp

For the dashboard, only `/` and `/home` were selected as the primary storage filesystems.

## Root filesystem usage

100 * (

  1 -

  node_filesystem_avail_bytes{

    instance="node-exporter:9100",

    mountpoint="/",

    fstype="ext4"

  }

  /

  node_filesystem_size_bytes{

    instance="node-exporter:9100",

    mountpoint="/",

    fstype="ext4"

  }

)

Visualization:

- Gauge

Unit:

- Percent (0-100)

Thresholds:

- 70 = Warning
- 90 = Critical

## Home filesystem usage

100 * (

  1 -

  node_filesystem_avail_bytes{

    instance="node-exporter:9100",

    mountpoint="/home",

    fstype="ext4"

  }

  /

  node_filesystem_size_bytes{

    instance="node-exporter:9100",

    mountpoint="/home",

    fstype="ext4"

  }

)

## Root capacity

node_filesystem_size_bytes{

  instance="node-exporter:9100",

  mountpoint="/",

  fstype="ext4"

}

Unit:

- Bytes (IEC)

## Home capacity

node_filesystem_size_bytes{

  instance="node-exporter:9100",

  mountpoint="/home",

  fstype="ext4"

}

Unit:

- Bytes (IEC)

## Root available

node_filesystem_avail_bytes{

  instance="node-exporter:9100",

  mountpoint="/",

  fstype="ext4"

}

## Home available

node_filesystem_avail_bytes{

  instance="node-exporter:9100",

  mountpoint="/home",

  fstype="ext4"

}

## IOPS

The user already had IOPS covered in the dashboard for:

- `nvme0n1` read IOPS
- `sda` read IOPS
- `nvme0n1` write IOPS
- `sda` write IOPS

Therefore, no duplicate IOPS panels were added.

The planned additional storage metrics were:

- Disk throughput
- Disk utilization

Suggested throughput queries:

Read:

rate(node_disk_read_bytes_total{device=~"nvme0n1|sda"}[5m])

Write:

rate(node_disk_written_bytes_total{device=~"nvme0n1|sda"}[5m])

Suggested disk utilization:

100 *

rate(node_disk_io_time_seconds_total{device=~"nvme0n1|sda"}[5m])

---

# 6. Networking Problem

Initially, querying:

node_network_receive_bytes_total

only showed interfaces such as:

eth0

lo

This was incorrect for the intended dashboard because the physical Wi-Fi interface was:

wlp5s0

The host itself showed:

lo

enp4s0

wlp5s0

virbr0

br-...

docker0

docker_gwbridge

veth...

The active physical interface was:

wlp5s0

## Root cause

Node Exporter was running inside the Docker Compose network:

monitoring_default

This was confirmed with:

docker inspect node-exporter --format '{{.HostConfig.NetworkMode}}'

which returned:

monitoring_default

Therefore Node Exporter saw the container's network namespace and reported its Docker `eth0`, instead of the host's actual interfaces.

---

# 7. Networking Solution

Node Exporter was moved to the host network namespace.

The final Node Exporter configuration was changed from using Docker bridge networking to:

node-exporter:

  image: prom/node-exporter:latest

  container_name: node-exporter

  restart: unless-stopped

  network_mode: host

  pid: host

  command:

    - "--path.rootfs=/host"

  volumes:

    - "/:/host:ro,rslave"

The old port mapping:

ports:

  - ":9100"

was removed because host networking makes Node Exporter directly available on the host at port 9100.

---

# 8. Prometheus Networking Fix

Because Node Exporter was no longer attached to the Docker Compose network, Prometheus could no longer use:

targets: ['node-exporter:9100']

Prometheus was changed to use:

targets: ['host.docker.internal:9100']

The Prometheus Docker service was given a host gateway entry:

extra_hosts:

  - "host.docker.internal:host-gateway"

The final Node Exporter scrape configuration became:

scrape_configs:

  - job_name: 'node'

    static_configs:

      - targets: ['host.docker.internal:9100']

  

  - job_name: "process"

    static_configs:

      - targets:

          - "process-exporter:9256"

  

  - job_name: "cadvisor"

    static_configs:

      - targets:

          - "cadvisor:8080"

This preserved the existing Docker-network communication for Process Exporter and cAdvisor while allowing Node Exporter to monitor the actual host.

---

# 9. Verification

After restarting the stack:

docker compose down

docker compose up -d

Node Exporter was verified from the host with:

curl http://localhost:9100/metrics | grep 'node_network_receive_bytes_total'

The important result was that the host interface appeared:

device="wlp5s0"

This confirmed that Node Exporter was now monitoring the Ubuntu host network namespace correctly.

---

# 10. Final Networking Dashboard

The active interface is:

wlp5s0

## Network throughput

RX:

rate(node_network_receive_bytes_total{device="wlp5s0"}[5m])

TX:

rate(node_network_transmit_bytes_total{device="wlp5s0"}[5m])

Visualization:

- Time series

Unit:

- bytes/sec (IEC)

## Current RX

rate(node_network_receive_bytes_total{device="wlp5s0"}[5m])

Visualization:

- Stat

Unit:

- bytes/sec (IEC)

## Current TX

rate(node_network_transmit_bytes_total{device="wlp5s0"}[5m])

Visualization:

- Stat

Unit:

- bytes/sec (IEC)

## Packet rate

RX:

rate(node_network_receive_packets_total{device="wlp5s0"}[5m])

TX:

rate(node_network_transmit_packets_total{device="wlp5s0"}[5m])

Unit:

- packets/sec

## Errors and drops

RX errors:

rate(node_network_receive_errs_total{device="wlp5s0"}[5m])

TX errors:

rate(node_network_transmit_errs_total{device="wlp5s0"}[5m])

RX drops:

rate(node_network_receive_drop_total{device="wlp5s0"}[5m])

TX drops:

rate(node_network_transmit_drop_total{device="wlp5s0"}[5m])

Unit:

- ops/sec

Healthy behavior is generally close to zero for errors and drops.

---

# 11. Current Dashboard Progress

The dashboard currently has these major areas:

Laptop Overview

│

├── CPU

│   ├── Total CPU usage

│   └── Per-core CPU usage

│

├── Memory

│   ├── Memory usage

│   ├── Memory used

│   ├── Memory breakdown

│   └── Swap

│

├── Storage

│   ├── / usage

│   ├── /home usage

│   ├── Root capacity

│   ├── Home capacity

│   ├── Root available

│   ├── Home available

│   ├── IOPS

│   ├── Throughput

│   └── Disk utilization

│

├── Network

│   ├── Current RX

│   ├── Current TX

│   ├── Throughput

│   ├── Packet rate

│   └── Errors & drops

│

├── NVIDIA GPU       ← NEXT

│

├── Processes

│

└── Logs / Loki

---

# 12. Next Recommended Section

The next section to build is the **NVIDIA GPU dashboard**, because DCGM Exporter is already part of the stack.

Recommended GPU panels:

1. GPU Utilization
2. GPU Memory Usage
3. GPU Memory %
4. GPU Temperature
5. GPU Power
6. Graphics/SM Clock
7. Memory Clock
8. Encoder utilization
9. Decoder utilization
10. GPU processes

The dashboard should prioritize GPU utilization, VRAM usage, temperature, and power on the main overview.