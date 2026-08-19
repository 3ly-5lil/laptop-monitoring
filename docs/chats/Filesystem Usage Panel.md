# Filesystem Usage Panel — Conversation Summary

## Goal

Build the left-most panel of a Grafana storage dashboard:

```text
┌──────────────────────┬────────────────────────┬──────────────────────┐
│ Filesystem Usage     │ Disk Throughput        │ Disk IOPS            │
│                      │                        │                      │
│ /          63.9%     │ Read / Write           │ Read / Write          │
│ 148 / 414 GB         │ NVMe + SATA            │ NVMe + SATA           │
│                      │                        │                      │
│ /home      87.5%     │                        │                        │
│ ~350 / ~400 GB       │                        │                        │
└──────────────────────┴────────────────────────┴──────────────────────┘
```

The desired filesystem panel should show, for `/` and `/home`, both:

- Usage percentage
- Used capacity / total capacity
- A visual gauge/bar
- Both filesystems in a single panel

---

## Problem Faced

The initial idea was to calculate filesystem usage percentage with PromQL and display it in Grafana.

The percentage query was:

100 *

(

  node_filesystem_size_bytes{mountpoint="/", fstype!~"tmpfs|overlay"}

  -

  node_filesystem_avail_bytes{mountpoint="/", fstype!~"tmpfs|overlay"}

)

/

node_filesystem_size_bytes{mountpoint="/", fstype!~"tmpfs|overlay"}

For both filesystems, the query can be generalized to:

100 *

(

  node_filesystem_size_bytes{mountpoint=~"/|/home", fstype!~"tmpfs|overlay"}

  -

  node_filesystem_avail_bytes{mountpoint=~"/|/home", fstype!~"tmpfs|overlay"}

)

/

node_filesystem_size_bytes{mountpoint=~"/|/home", fstype!~"tmpfs|overlay"}

The difficulty was that a Grafana gauge/bar gauge naturally has one primary numeric value. The desired display combines three pieces of information:

63.9%

148 GB / 414 GB

[usage bar]

Simply querying the percentage does not automatically give the used/total capacity text.

---

## Relevant Prometheus Queries

### Used bytes

node_filesystem_size_bytes{mountpoint=~"/|/home", fstype!~"tmpfs|overlay"}

-

node_filesystem_avail_bytes{mountpoint=~"/|/home", fstype!~"tmpfs|overlay"}

### Total bytes

node_filesystem_size_bytes{mountpoint=~"/|/home", fstype!~"tmpfs|overlay"}

### Usage percentage

100 *

(

  node_filesystem_size_bytes{mountpoint=~"/|/home", fstype!~"tmpfs|overlay"}

  -

  node_filesystem_avail_bytes{mountpoint=~"/|/home", fstype!~"tmpfs|overlay"}

)

/

node_filesystem_size_bytes{mountpoint=~"/|/home", fstype!~"tmpfs|overlay"}

The `fstype!~"tmpfs|overlay"` filter avoids showing temporary/container overlay filesystems.

---

## Solution / Recommended Visualization

Use a **Bar gauge** for the filesystem panel.

The bar itself represents the usage percentage, while the panel also exposes the used and total capacity.

Conceptually:

Filesystem Usage

  

/       63.9%

        148 GB / 414 GB

██████████████░░░░░░░░

  

/home   87.5%

        350 GB / 400 GB

██████████████████░░░░

This is preferable to using two separate gauges because `/` and `/home` can be compared immediately in the same panel.

### Suggested Grafana settings

**Visualization**

- Bar gauge
- Horizontal orientation
- Show name + value
- Unit: Percent (0–100)
- Min: 0
- Max: 100
- Decimals: 1

**Thresholds**

- 0: normal
- 70: warning
- 90: critical

**Field display names**

- `/`
- `/home`

---

## Important Design Decision

A pure gauge is not ideal for this use case because the panel needs to communicate both:

1. How full the filesystem is
2. How much storage is actually being used

For example:

63.9%

148 GB / 414 GB

The percentage should be the primary visualization value because it determines the bar/gauge position.

The capacity query provides the supporting used/total information.

---

## Final Recommended Panel

The final filesystem panel should communicate:

┌──────────────────────────────┐

│ Filesystem Usage             │

│                              │

│ /                 63.9%      │

│ 148 GB / 414 GB              │

│ ██████████████░░░░░░░░       │

│                              │

│ /home             87.5%      │

│ 350 GB / 400 GB              │

│ ██████████████████░░░░       │

└──────────────────────────────┘

### Final recommendation

Use **one horizontal Bar gauge panel** containing `/` and `/home`, with the **usage percentage as the main value** and **used/total capacity as secondary information**.

This gives the dashboard a compact storage overview while keeping the three-column layout:

Filesystem Usage | Disk Throughput | Disk IOPS