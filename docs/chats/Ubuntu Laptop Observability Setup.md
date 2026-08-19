# Ubuntu Laptop Observability Setup — Conversation Summary

## Goal

Build a local observability stack for an Ubuntu laptop that provides:

- Live host CPU, RAM, disk, network, and system metrics
- Per-process CPU and memory consumption
- Docker/container resource usage
- GPU metrics for an NVIDIA GTX 1650 Ti
- System/journal logs
- Grafana dashboards combining metrics and logs

Target architecture:

```text
Ubuntu Laptop
│
├── Node Exporter       → host metrics
├── Process Exporter    → per-process metrics
├── cAdvisor            → Docker/container metrics
├── DCGM Exporter       → NVIDIA GPU metrics
│
├── Alloy               → system/journal log collection
│      ↓
│     Loki              → log storage
│
├── Prometheus          → metrics storage
│
└── Grafana             → visualization
```

## 1. Node Exporter

### Problem

Prometheus initially reported:

Error scraping target: Get "http://localhost:9100/metrics":

dial tcp [::1]:9100: connect: connection refused

There was also confusion because clicking a metrics URL opened a Docker hostname such as:

http://4ab0e87f21ad:9100/metrics

instead of `localhost`.

### Explanation

The exporter was running in Docker, so `localhost` from inside a container refers to that container, not the Ubuntu host.

For Prometheus-to-exporter communication, Docker service names such as:

node-exporter:9100

should be used when both services are on the same Compose network.

### Verification

Node Exporter was successfully verified from the host:

curl http://localhost:9100/metrics | grep node_cpu_seconds_total | head

It returned:

node_cpu_seconds_total{cpu="0",mode="idle"} ...

node_cpu_seconds_total{cpu="0",mode="iowait"} ...

node_cpu_seconds_total{cpu="0",mode="system"} ...

node_cpu_seconds_total{cpu="0",mode="user"} ...

Memory was also verified:

curl http://localhost:9100/metrics | grep node_memory_MemTotal_bytes

Result:

node_memory_MemTotal_bytes 3.289485312e+10

This is approximately 32.9 GB and matches the laptop's 32 GB RAM.

### Conclusion

Node Exporter is correctly collecting host-level CPU and memory metrics.

It does not need `privileged: true` for the current host metrics configuration.

---

# 2. Loki

## Problem

Loki initially failed during startup with:

Get "http://localhost:8500/v1/kv/collectors/scheduler":

dial tcp [::1]:8500: connect: connection refused

and:

unable to initialise ring state

Loki was trying to contact Consul on port 8500 for KV/ring state.

## Explanation

Loki has a ring used for distributed coordination.

The ring tracks things such as:

- Loki instance membership
- token ownership
- responsibility for work
- replication state

The KV store is where coordination/ring information can be stored.

Possible approaches include:

- Consul
- etcd
- memberlist
- inmemory

For a single Loki instance on a laptop, Consul is unnecessary.

## Solution

Use an in-memory KV store:

common:

  path_prefix: /loki

  

  replication_factor: 1

  

  ring:

    kvstore:

      store: inmemory

  

  storage:

    filesystem:

      chunks_directory: /loki/chunks

      rules_directory: /loki/rules

This keeps the ring state local to the Loki process.

### Important clarification

`inmemory` does NOT mean the log data is stored in RAM.

The actual Loki log chunks are still stored on the filesystem:

storage:

  filesystem:

    chunks_directory: /loki/chunks

    rules_directory: /loki/rules

The in-memory setting only applies to the ring/KV coordination state.

## Memberlist discussion

Memberlist was considered as an alternative:

ring:

  kvstore:

    store: memberlist

Memberlist is useful when running multiple Loki instances because nodes can exchange membership/ring information directly through gossip.

For the current single-node laptop setup, `inmemory` was selected because it is simpler and sufficient.

## Loki readiness

After fixing the KV configuration, Loki initially returned:

Ingester not ready: waiting for 15s after being ready

This is a normal startup readiness delay.

After the startup period, Loki should return:

ready

from:

curl http://localhost:3100/ready

---

# 3. Permissions and privileges

A major concern was whether exporters and log collectors need root/privileged access.

The conclusion was:

> Do not give every monitoring component `privileged: true` by default.

Instead, provide only the host access each component actually requires.

General design:

Node Exporter       → host /proc and /sys access

Process Exporter    → host process visibility

cAdvisor            → Docker/cgroup visibility

Alloy               → read-only journal access

Loki                → its own storage

Prometheus          → exporter endpoints

Grafana             → its own storage

Permissions should be tested before adding broad privileges.

---

# 4. Alloy and system logs

Alloy is the component that reads the host logs.

Loki itself does not directly read the Ubuntu journal.

The intended pipeline is:

Ubuntu systemd journal

        ↓

      Alloy

        ↓

       Loki

        ↓

     Grafana

The host has both:

/var/log/journal

/run/log/journal

and they were checked with:

ls -ld /var/log/journal

ls -ld /run/log/journal

Both exist and are owned by:

root:systemd-journal

with ACLs enabled (`+` at the end of the permissions).

The important next validation is to check Alloy itself:

docker compose logs alloy --tail=100

and determine which user it runs as:

docker exec alloy id

The journal ACLs can be inspected with:

getfacl /var/log/journal

getfacl /run/log/journal

No permission changes or `privileged: true` should be added until an actual permission problem is confirmed.

A useful end-to-end test is:

logger "Monitoring test: Loki and Alloy are working"

Then verify:

journalctl -n 20 | grep "Monitoring test"

and query Loki for the corresponding stream/log entry.

---

# 5. Process Exporter

Process Exporter is intended to provide per-process resource metrics.

Important metrics include:

namedprocess_namegroup_cpu_seconds_total

namedprocess_namegroup_memory_bytes

namedprocess_namegroup_num_procs

Useful PromQL:

### Process CPU

100 *

sum by (groupname) (

  rate(namedprocess_namegroup_cpu_seconds_total[5m])

)

This gives CPU usage in percentage-style units.

### Process memory

sum by (groupname) (

  namedprocess_namegroup_memory_bytes

)

The main permission concern is whether Process Exporter can see all host processes through `/proc`.

The approach is to verify it first rather than automatically making the container privileged.

Useful checks:

curl -s http://localhost:9256/metrics | grep namedprocess_namegroup_num_procs | head

and:

docker compose logs process-exporter --tail=50

---

# 6. cAdvisor

cAdvisor was tested and successfully worked WITHOUT `privileged: true`.

This is an important final result.

The purpose of cAdvisor is Docker/container monitoring.

Important metrics include:

container_cpu_usage_seconds_total

container_memory_working_set_bytes

container_network_receive_bytes_total

container_network_transmit_bytes_total

Useful PromQL:

### Container CPU

sum by (name) (

  rate(container_cpu_usage_seconds_total[5m])

)

### Container memory

sum by (name) (

  container_memory_working_set_bytes

)

### Network receive

sum by (name) (

  rate(container_network_receive_bytes_total[5m])

)

### Network transmit

sum by (name) (

  rate(container_network_transmit_bytes_total[5m])

)

cAdvisor should be exposed to Prometheus as:

- job_name: "cadvisor"

  static_configs:

    - targets:

        - "cadvisor:8080"

Prometheus should report:

cadvisor    UP

---

# 7. NVIDIA GPU monitoring

The laptop has:

GPU: NVIDIA GeForce GTX 1650 Ti

VRAM: 4096 MiB

Driver: 595.84

CUDA: 13.2

`nvidia-smi` showed:

NVIDIA GeForce GTX 1650 Ti

32 MiB / 4096 MiB

GPU Utilization: 2%

Temperature: 45C

Power: 3W / 50W

NVIDIA container support is installed and working.

Docker reports:

cdi: nvidia.com/gpu=0

cdi: nvidia.com/gpu=GPU-f7e6811a-a5f0-df92-4c3c-66d7160d8838

cdi: nvidia.com/gpu=all

  

Runtimes: io.containerd.runc.v2 nvidia runc

NVIDIA Container Toolkit:

NVIDIA Container Toolkit CLI version 1.20.0

Therefore the system is ready for GPU-aware containers.

---

# 8. DCGM Exporter version correction

An earlier recommendation used:

4.8.0-4.7.2-ubuntu22.04

This was identified as an incorrect/outdated pairing and should NOT be treated as the chosen version.

The DCGM Exporter tag format is:

<DCGM-version>-<exporter-version>-<image-variant>

For example:

4.5.1-4.8.0-ubuntu22.04

A later current documented pairing discussed was:

4.5.3-4.8.2-ubuntu22.04

The important lesson is to verify the current NVIDIA-published pairing rather than blindly using an old image tag.

---

# 9. DCGM Exporter test

The next step is to test DCGM Exporter manually BEFORE putting it into Compose.

Pull:

docker pull nvcr.io/nvidia/k8s/dcgm-exporter:4.5.3-4.8.2-ubuntu22.04

Run:

docker run -d \

  --name dcgm-test \

  --gpus all \

  --cap-add SYS_ADMIN \

  -p 9400:9400 \

  nvcr.io/nvidia/k8s/dcgm-exporter:4.5.3-4.8.2-ubuntu22.04

Check:

docker ps --filter name=dcgm-test

docker logs dcgm-test

Test GPU access:

docker exec dcgm-test nvidia-smi

Test Prometheus metrics:

curl http://localhost:9400/metrics

Specific checks:

curl -s http://localhost:9400/metrics | grep DCGM_FI_DEV_GPU_UTIL

curl -s http://localhost:9400/metrics | grep DCGM_FI_DEV_GPU_TEMP

curl -s http://localhost:9400/metrics | grep DCGM_FI_DEV_FB_USED

The purpose is to establish:

GTX 1650 Ti

     ↓

NVIDIA Container Toolkit

     ↓

DCGM Exporter

     ↓

:9400/metrics

before adding the exporter permanently to Docker Compose and Prometheus.

The `SYS_ADMIN` capability is being used for the initial official-style test, but the longer-term goal remains to minimize privileges where possible.

---

# 10. Current status

## Confirmed working

- Node Exporter
- Host CPU metrics
- Host memory metrics
- Prometheus architecture
- Loki startup after fixing ring/KV configuration
- cAdvisor
- cAdvisor without privileged mode
- NVIDIA host driver
- NVIDIA Container Toolkit
- Docker NVIDIA runtime/CDI

## In progress / needs final verification

- Alloy → Loki journal ingestion
- Process Exporter complete host-process visibility
- DCGM Exporter on GTX 1650 Ti
- Prometheus scraping of all exporters
- Grafana dashboard construction

---

# 11. Recommended next sequence

Continue in this order:

1. Finish Alloy

   Ubuntu journal → Alloy → Loki

  

2. Verify Process Exporter

   Host processes → CPU/RAM metrics

  

3. Finish DCGM Exporter

   GTX 1650 Ti → GPU metrics

  

4. Configure Prometheus targets

   Node Exporter

   Process Exporter

   cAdvisor

   DCGM Exporter

  

5. Configure Grafana

   Prometheus datasource

   Loki datasource

  

6. Build dashboards

   Overview

   CPU

   Memory

   Disk

   Network

   Processes

   Docker

   GPU

   Logs

  

7. Add alerting

   CPU

   RAM

   Disk

   Temperature

   GPU

   exporter health

  

8. Harden the stack

   Remove unnecessary privileges

   Minimize host mounts

   Use read-only mounts where possible

---

# Final architecture

                         ┌───────────────┐

                         │    Grafana    │

                         │     :3000     │

                         └───────┬───────┘

                                 │

                    ┌────────────┴────────────┐

                    │                         │

                    ▼                         ▼

             ┌─────────────┐           ┌─────────────┐

             │ Prometheus  │           │    Loki     │

             │    :9090    │           │    :3100    │

             └──────┬──────┘           └──────▲──────┘

                    │                         │

        ┌───────────┼───────────────┐         │

        │           │               │         │

        ▼           ▼               ▼         │

 Node Exporter  Process Exporter  cAdvisor    │

    :9100          :9256            :8080     │

        │           │               │         │

        └───────────┴───────┬───────┘         │

                            │                 │

                       Ubuntu Host            │

                                              │

                         ┌──────────────┐     │

                         │    Alloy     │─────┘

                         │ Journal logs │

                         └──────┬───────┘

                                │

                         Ubuntu systemd

                             journal

  

                       NVIDIA GPU

                           │

                           ▼

                     DCGM Exporter

                         :9400

                           │

                           ▼

                       Prometheus

The overall design deliberately avoids unnecessary privilege escalation and uses Docker service names for inter-container communication.