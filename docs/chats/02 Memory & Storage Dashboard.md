# Memory & Storage Dashboard — Conversation Summary

## Context

The goal was to build a Grafana 13.1.3 **Memory & Storage Dashboard** for a laptop monitoring stack using Prometheus and node-exporter, with process-exporter available for process-level memory information.

The dashboard should answer:

> **What's eating my RAM and disk?**

The design was initially expanded panel-by-panel, but we later identified that it was becoming too large and more like a monitoring report than a practical laptop dashboard.

---

# Initial Dashboard Concept

The original Memory & Storage dashboard was planned around:

- RAM Used
- RAM Available
- Memory Usage Over Time
- Memory pressure:
  - Used
  - Available
  - Cached
  - Buffers
  - Swap Used
  - Swap I/O
  - OOM events
- Process-level memory usage
- Filesystem usage
- Disk usage over time
- Potentially disk I/O

The important design principle was:

> RAM percentage alone is not enough to diagnose memory pressure.

For example, a laptop can show relatively high RAM utilization while still being healthy if there is no active swapping. Conversely, lower RAM utilization combined with sustained swap activity can indicate real memory pressure.

---

# Row 1 — Current Memory State

Four Stat panels were designed.

## RAM Used

PromQL:

```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

Grafana:

- Visualization: **Stat**
- Calculation: **Last**
- Unit: **bytes (IEC)**
- Decimals: **1**

Example:

19.2 GiB

The calculation deliberately uses `MemAvailable` instead of `MemFree`, because Linux filesystem cache is generally reclaimable.

---

## RAM Available

PromQL:

node_memory_MemAvailable_bytes

Grafana:

- Visualization: **Stat**
- Calculation: **Last**
- Unit: **bytes (IEC)**
- Decimals: **1**

---

## RAM Utilization

PromQL:

100 * (

  1 -

  node_memory_MemAvailable_bytes

  /

  node_memory_MemTotal_bytes

)

Grafana:

- Visualization: **Stat**
- Unit: **Percent (0-100)**
- Decimals: **0**

Suggested thresholds:

Base       Green

70%        Yellow

85%        Red

The thresholds are intentionally not extremely aggressive because high Linux RAM utilization is not automatically bad.

---

## Swap Used

PromQL:

node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes

Grafana:

- Visualization: **Stat**
- Unit: **bytes (IEC)**
- Decimals: **1**

Suggested thresholds:

Base       Green

1 GiB      Yellow

4 GiB      Red

Important caveat:

> Swap Used alone does not prove current memory pressure.

Linux can leave inactive pages in swap even after memory pressure has disappeared.

---

# Row 2 — Memory Usage Over Time

Visualization:

> **Time series**

Queries:

### Used

node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

Legend:

Used

### Available

node_memory_MemAvailable_bytes

Legend:

Available

### Cached

node_memory_Cached_bytes

Legend:

Cached

### Buffers

node_memory_Buffers_bytes

Legend:

Buffers

Grafana settings:

- Unit: **bytes (IEC)**
- Decimals: **1**
- Legend: **List**
- Placement: **Bottom**
- Tooltip: **All**
- Line width: approximately `2`
- Low fill opacity
- No stacking

The series should **not be stacked**, because Used, Available, Cached and Buffers are different views of memory rather than quantities that should be summed.

---

# Row 3 — Memory Pressure

Four Stat panels were initially proposed.

## Cached

node_memory_Cached_bytes

Unit:

bytes (IEC)

No threshold was recommended because cache usage is normally beneficial.

---

## Buffers

node_memory_Buffers_bytes

Unit:

bytes (IEC)

No threshold was recommended.

---

## Swap Used

node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes

Unit:

bytes (IEC)

---

## Swap Usage

PromQL:

100 *

(

  1 -

  node_memory_SwapFree_bytes

  /

  node_memory_SwapTotal_bytes

)

and

node_memory_SwapTotal_bytes > 0

Unit:

Percent (0-100)

Decimals:

0

Suggested thresholds:

Base       Green

25%        Yellow

50%        Red

The `node_memory_SwapTotal_bytes > 0` condition prevents division-by-zero when no swap is configured.

---

# Row 4 — Active Memory Pressure

The purpose of this row was to distinguish:

> "There is data in swap"

from:

> **"The machine is actively swapping and experiencing memory pressure."**

## Swap Activity

Visualization:

> **Time series**

### Swap In

rate(node_vmstat_pswpin[5m])

Legend:

Swap In (pages/s)

### Swap Out

rate(node_vmstat_pswpout[5m])

Legend:

Swap Out (pages/s)

These are counters, so `rate(...[5m])` converts them into an activity rate.

The desired normal state is approximately:

Swap In   ───────── 0

Swap Out  ───────── 0

Sustained activity in both directions, especially when Available RAM is low, is a much stronger indicator of memory pressure.

---

## OOM Kills

The proposed metric was:

node_vmstat_oom_kill

If available, the dashboard query should be:

increase(node_vmstat_oom_kill[1h])

This answers:

> How many OOM kills occurred during the last hour?

Stat settings:

- Unit: `none`
- Decimals: `0`
- Base: Green
- `1`: Red

If `node_vmstat_oom_kill` does not exist, the dashboard should not fabricate the metric. OOM detection can instead be implemented using systemd/journal information or another appropriate exporter.

---

# Row 5 — Top Processes by Memory

This was the most important row for answering:

> **What's eating my RAM?**

The process-exporter metric was verified to exist:

namedprocess_namegroup_memory_bytes{

  groupname="(sd-pam)",

  instance="process-exporter:9256",

  job="process",

  memtype="proportionalResident"

}

The relevant labels are:

- `groupname`
- `instance`
- `job`
- `memtype`

---

## Problem encountered: repeated processes

A simple query like:

topk(

  10,

  namedprocess_namegroup_memory_bytes{

    job="process",

    memtype="proportionalResident"

  }

)

was incorrect for the desired table.

It caused processes such as:

python 3

python 3

python 3

...

to appear repeatedly, sometimes dozens of times.

`dcgm-exporter` also appeared multiple times.

### Why?

`topk()` was selecting the individual returned series.

`groupname` was not unique across all the returned series.

Therefore the correct solution was to **aggregate by `groupname` first**.

---

## Correct query

topk(

  10,

  sum by (groupname) (

    namedprocess_namegroup_memory_bytes{

      job="process",

      memtype="proportionalResident"

    }

  )

)

This changes the behavior to:

python 3       → one row

dcgm-exporter  → one row

chrome         → one row

java            → one row

with the memory of all matching series combined.

This is the preferred interpretation for this dashboard because the question is:

> How much memory is this process group consuming?

rather than:

> Which individual PID is consuming memory?

---

## PSS / proportionalResident

The selected metric is:

memtype="proportionalResident"

This represents proportional resident memory / PSS-style accounting.

It is preferable for this dashboard because shared memory is proportionally attributed instead of being fully counted against every process.

---

## Sorting problem

PromQL `sort_desc()` was considered:

sort_desc(

  topk(

    10,

    sum by (groupname) (

      namedprocess_namegroup_memory_bytes{

        job="process",

        memtype="proportionalResident"

      }

    )

  )

)

But this was not effective for the final Grafana table presentation because Grafana was ultimately ordering the table based on its fields/labels.

### Final approach

Use Prometheus only to select the top 10:

topk(

  10,

  sum by (groupname) (

    namedprocess_namegroup_memory_bytes{

      job="process",

      memtype="proportionalResident"

    }

  )

)

Then sort in Grafana **after the memory value has become a table field**:

Transformations:

  

Labels to fields

        ↓

Organize fields

        ↓

Sort by → Memory → Descending

The desired final table is:

┌──────────────────────┬──────────────┐

│ Process              │ Memory       │

├──────────────────────┼──────────────┤

│ chrome               │ 4.8 GiB      │

│ python 3             │ 3.2 GiB      │

│ java                 │ 2.7 GiB      │

│ code                 │ 1.9 GiB      │

│ docker               │ 1.4 GiB      │

│ dcgm-exporter        │ 1.1 GiB      │

└──────────────────────┴──────────────┘

Memory should remain numeric so Grafana can sort it numerically.

---

# Grafana Table Issue

When first creating the table, Grafana showed:

- a `Time` row/column
- a process selection combo box
- a single selected process filling the table

This happened because Grafana was treating the Prometheus result as time-series fields rather than the desired table rows.

The intended fix was:

1. Set Prometheus query **Format → Table**
2. Use **Labels to fields**
3. Use **Organize fields**
4. Hide unnecessary fields such as:
    - Time
    - instance
    - job
    - memtype
5. Keep:
    - groupname
    - Value
6. Rename:
    - `groupname → Process`
    - `Value → Memory`
7. Use **Sort by → Memory → Descending**

---

# Dashboard Size Problem

After building Rows 1–5 and starting Storage, it became clear that the dashboard was getting too large.

The dashboard was beginning to look more like a detailed monitoring report than a practical laptop dashboard.

The final design direction was therefore changed to:

> **High signal, low clutter.**

The dashboard should answer the question quickly rather than expose every available metric.

---

# Final Recommended Dashboard

## 🧠 Memory

### Row 1 — Current State

Four Stat panels:

RAM Used

RAM Available

RAM Utilization

Swap Used

---

### Row 2 — Memory History

One full-width Time series:

Used

Available

Cached

Buffers can be omitted from the main graph to reduce clutter.

---

### Row 3 — Memory Diagnosis

Two panels:

┌──────────────────────────────────────┬─────────────────────┐

│ Top Processes by Memory              │ Swap Activity       │

│                                      │                     │

│ chrome       4.8 GiB                 │ Swap In             │

│ python 3     3.2 GiB                 │ Swap Out            │

│ java         2.7 GiB                 │                     │

│ code         1.9 GiB                 │ OOM: 0              │

└──────────────────────────────────────┴─────────────────────┘

This is much more useful than dedicating separate panels to Cached, Buffers, Swap Usage, and OOM.

---

# 💾 Storage

Instead of four separate storage Stat panels, use **two panels**:

┌────────────────────────────┬────────────────────────────┐

│ /                          │ /home                      │

│ 62% used                   │ 48% used                   │

│ 157 GiB free               │ 265 GiB free               │

└────────────────────────────┴────────────────────────────┘

Each panel can show both:

- Usage percentage
- Free space

This avoids having separate `/ Free` and `/home Free` panels.

---

## Storage history

One full-width Time series:

/

 /home

showing filesystem usage over time.

---

# Final Compact Layout

╔════════════════════════════════════════════════════════════════════╗

║ 🧠 MEMORY                                                        ║

╠══════════════╦══════════════╦══════════════╦══════════════════════╣

║ RAM Used     ║ RAM Available║ RAM Util.    ║ Swap Used            ║

╠══════════════╩══════════════╩══════════════╩══════════════════════╣

║                                                                    ║

║ Memory Usage                                                       ║

║ Used / Available / Cached                                          ║

║                                                                    ║

╠══════════════════════════════════════════╦═════════════════════════╣

║ Top Processes by Memory                 ║ Swap Activity           ║

║ chrome       4.8 GiB                    ║ Swap In                 ║

║ python 3     3.2 GiB                    ║ Swap Out                ║

║ java         2.7 GiB                    ║ OOM                     ║

║ code         1.9 GiB                    ║                         ║

╠══════════════════════════════════════════╩═════════════════════════╣

║ 💾 STORAGE                                                        ║

╠══════════════════════════════════════╦═════════════════════════════╣

║ /                                    ║ /home                       ║

║ 62% used • 157 GiB free             ║ 48% used • 265 GiB free    ║

╠══════════════════════════════════════╩═════════════════════════════╣

║ Storage Usage                                                     ║

║ /                                                                  ║

║ /home                                                              ║

╚════════════════════════════════════════════════════════════════════╝

---

# Key Lessons / Decisions

1. **Use `MemAvailable`, not `MemFree`, for application memory availability.**
2. **RAM utilization alone is not enough to diagnose memory pressure.**
3. **Swap Used is not the same as active swapping.**
4. **`rate(node_vmstat_pswpin[5m])` and `rate(node_vmstat_pswpout[5m])` are better indicators of active swap pressure.**
5. **Use PSS/proportional resident memory for process memory attribution.**
6. **Aggregate process-exporter data by `groupname` before using `topk()`** to prevent repeated entries such as many `python 3` or `dcgm-exporter` rows.
7. **Use `topk()` in PromQL to select the largest processes, but use Grafana's `Sort by → Memory → Descending` for table presentation.**
8. **Don't over-panel the dashboard.** A compact dashboard is preferable for a laptop overview.
9. Detailed metrics such as disk I/O, extensive OOM diagnostics, buffers, and additional filesystem statistics can belong in dedicated troubleshooting dashboards rather than the main Memory & Storage dashboard.

---

# Current Status

Completed conceptually:

- Row 1 — Current RAM state
- Row 2 — Memory history
- Row 3 — Memory pressure
- Row 4 — Swap activity / OOM
- Row 5 — Top processes by memory

Storage was started but deliberately **reduced in scope** after recognizing that the dashboard was becoming too large.

The next implementation should therefore continue with the **compact Storage design**, not the earlier four-panel-per-row design.