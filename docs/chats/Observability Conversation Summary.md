# Observability Conversation Summary

## Scope

This document summarizes the Grafana / Prometheus / Node Exporter / DCGM issues discussed, the investigation, the causes identified, and the final configuration decisions.

## 1. Node Exporter hostname mismatch in Grafana

### Problem

Grafana appeared to show both:

- `host.docker.internal:9100`
- the old `node-exporter:9100`

Prometheus, however, showed only:

```text
host.docker.internal:9100
```

This initially looked like Grafana might have stale configuration, a dashboard variable, or a transformation still referencing the deleted target.

### Investigation

We checked the following:

- Grafana dashboard variables
- Grafana panel transformations
- Grafana Prometheus datasource
- Grafana Explore queries
- Prometheus results directly through Grafana

The dashboard had:

- no variables involved in the issue
- no transformations involved in the issue

The following Explore queries behaved correctly:

up

and:

count by (instance) (node_uname_info)

The current target appeared as:

host.docker.internal:9100

with no current `node-exporter:9100` target.

### Root Cause

The old `node-exporter:9100` value was **historical Prometheus data**.

The dashboard had been viewed with a larger time range, so Grafana was querying samples from before the Prometheus target was changed.

Old samples:

instance="node-exporter:9100"

New samples:

instance="host.docker.internal:9100"

Prometheus retains the old samples until they fall outside its retention period.

### Final Result

No Grafana or Prometheus configuration change was required.

The monitoring pipeline was working correctly:

Node Exporter

    ↓

host.docker.internal:9100

    ↓

Prometheus

    ↓

Grafana

When using a long historical time range, both the old and new `instance` labels can legitimately appear.

For the laptop dashboard, shorter default ranges such as **Last 6 hours** or **Last 24 hours** are appropriate when focusing on current health.

---

## 2. DCGM `hostname` vs Prometheus `instance`

### Problem / Question

A DCGM metric looked like this:

DCGM_FI_DEV_GPU_UTIL{

  DCGM_FI_DRIVER_VERSION="595.84",

  UUID="GPU-f7e6811a-a5f0-df92-4c3c-66d7160d8838",

  device="nvidia0",

  gpu="0",

  hostname="ee891a0c4bd5",

  instance="dcgm-exporter:9400",

  job="dcgm",

  modelName="NVIDIA GeForce GTX 1650 Ti",

  pci_bus_id="00000000:01:00.0"

}

The question was why the `hostname` value could change.

### Important distinction

The following two labels have different meanings:

|Label|Meaning|
|---|---|
|`hostname`|Hostname seen by DCGM Exporter itself|
|`instance`|Prometheus scrape target|
|`job`|Prometheus scrape job|
|`UUID`|Physical GPU identity|

The value:

hostname="ee891a0c4bd5"

looks like a Docker container ID.

### Root Cause

DCGM Exporter is running inside Docker.

When a Docker container has no explicitly configured hostname, Docker commonly uses the container ID as its hostname.

Therefore, recreating the DCGM Exporter container can result in a new hostname:

old container

hostname = abc123...

  

       ↓ container recreation

  

new container

hostname = ee891a0c4bd5

For example, operations such as:

docker compose down

docker compose up -d

or:

docker compose up -d --force-recreate

can recreate the container and therefore change the reported `hostname`.

This does **not** mean that the physical laptop hostname or GPU identity changed.

### Stable GPU Identity

The GPU UUID is much more appropriate as the physical GPU identity:

UUID="GPU-f7e6811a-a5f0-df92-4c3c-66d7160d8838"

The UUID should be preferred over the Docker-derived `hostname` when identifying the GPU in dashboards.

---

## 3. Dropping the DCGM `hostname` label

### Requirement

The goal was to remove the unstable Docker-derived `hostname` label from DCGM queries.

### Query-level solution

A query can aggregate away the label:

max without(hostname) (

  DCGM_FI_DEV_GPU_UTIL

)

This removes `hostname` from the resulting label set while preserving the GPU utilization value, assuming the remaining labels uniquely identify the desired series.

For aggregation where summing is actually appropriate:

sum without(hostname) (

  DCGM_FI_DEV_GPU_UTIL

)

should be used instead.

`sum` should not be used merely to remove the label if the metric represents a value where aggregation is not intended.

### Prometheus scrape-time solution

If the `hostname` label is never needed, it can instead be dropped before storage using Prometheus metric relabeling:

metric_relabel_configs:

  - action: labeldrop

    regex: hostname

This prevents the unstable Docker hostname from being stored as part of the series.

### Recommended approach

For this laptop observability setup, if `hostname` is not used anywhere in the dashboards, dropping it at the Prometheus level is preferable.

This avoids unnecessary label churn when the DCGM Exporter container is recreated.

If the label might be useful for debugging, keep it and remove it only in dashboard queries.

---

## Final State

### Node Exporter

Current Prometheus target:

host.docker.internal:9100

The old:

node-exporter:9100

was historical data and does not indicate a current configuration problem.

### DCGM Exporter

Current scrape target:

dcgm-exporter:9400

The DCGM metric's:

hostname="ee891a0c4bd5"

is the Docker container hostname and may change whenever the container is recreated.

### Dashboard identity recommendations

Use:

- `instance` for the Prometheus scrape target
- `job` for exporter/job grouping
- `UUID` for stable GPU identity
- `gpu`, `device`, `modelName`, and `pci_bus_id` where useful

Avoid using DCGM's Docker-derived `hostname` as the identity of the physical laptop or GPU.

### Key Lessons

1. Grafana showing an old target does not necessarily mean Grafana is misconfigured; historical Prometheus samples can contain old label values.
2. Prometheus `instance` and exporter-provided `hostname` are different concepts.
3. A Docker container's hostname can change when the container is recreated.
4. DCGM GPU UUID is a better stable identity than the Docker-derived hostname.
5. Unneeded labels can be removed at query time with `without(...)` aggregation or permanently with Prometheus `labeldrop`.