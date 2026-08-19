# Process Dashboard — Conversation Summary

## Overview

This conversation focused on building the **Process Dashboard** for the user's Grafana observability stack.

The dashboard uses **process-exporter** to provide process-level visibility that complements the existing host, memory, Docker, GPU, and logs dashboards.

The intended investigation path is:

```text
Laptop / Overview
      │
      ├── CPU / Memory problem
      │
      ▼
Process Dashboard
      │
      ├── Which process?
      │
      ├── How much CPU?
      │
      └── How much RAM?
      │
      ▼
Docker / Container Dashboard
      │
      └── Which container/application?      
```

# Problems Faced

## 1. Process dashboard was producing too many repeated processes

The user had processes such as:

- `python` repeated many times
- `dcgm-exporter` repeated multiple times
- Brave/browser processes appearing individually

This made the dashboard unnecessarily large and difficult to investigate.

### Solution

Use process-exporter's `groupname` label and aggregate by it:

sum by (groupname) (...)

This groups all matching processes into a single logical application/process group.

For example:

python

python

python

python

becomes:

python

with the values aggregated together.

---

# 2. Process CPU exceeded 100%

The user observed Brave using approximately:

101% CPU

At first this looked suspicious.

### Explanation

The process-exporter CPU metric is effectively measured in terms of CPU-core capacity.

The user's CPU has:

12 logical CPUs

Therefore:

100%  ≈ 1 logical CPU

200%  ≈ 2 logical CPUs

400%  ≈ 4 logical CPUs

1000% ≈ 10 logical CPUs

1200% ≈ 12 logical CPUs

Therefore:

Brave = 101%

is not inherently abnormal. It means Brave is using approximately one logical CPU.

### Dashboard implication

Do **not** cap the process CPU visualization at 100%.

A process can legitimately exceed 100% when using multiple CPU threads.

---

# 3. Process memory showed impossible values

The user observed:

Brave    54.3 TiB

Obsidian 2.92 TiB

This was clearly inconsistent with the user's system having approximately:

32 GB RAM

The process-exporter output revealed the cause.

For Brave:

memtype="virtual"

58091307737088 bytes

≈ 54.3 TiB

This is **virtual address space**, not physical RAM usage.

---

# 4. Understanding process-exporter memory types

The following metrics were returned:

memtype="proportionalResident"

memtype="proportionalSwapped"

memtype="resident"

memtype="swapped"

memtype="virtual"

Their meanings:

|memtype|Meaning|Dashboard use|
|---|---|---|
|`resident`|Physical RAM currently resident (RSS)|**Use for main RAM panel**|
|`proportionalResident`|PSS / proportional resident memory|Optional|
|`swapped`|Memory currently swapped out|Optional|
|`proportionalSwapped`|Proportional swapped memory|Optional|
|`virtual`|Virtual address space|**Do not use for RAM**|

The main problem was that querying:

namedprocess_namegroup_memory_bytes

without filtering `memtype` returned all memory types, including `virtual`.

---

# Solution

## Use resident memory for process RAM

The correct metric filter is:

namedprocess_namegroup_memory_bytes{

  memtype="resident"

}

For the top 10 processes:

topk(

  10,

  sum by (groupname) (

    namedprocess_namegroup_memory_bytes{

      memtype="resident"

    }

  )

)

### Visualization

**Table**

### Unit

**bytes (IEC)**

### Legend

{{groupname}}

Expected output:

Process       RAM

----------------------

brave         11.13 GiB

obsidian       2.92 GiB

java           ...

code           ...

python         ...

---

# Brave memory result

The actual process-exporter output for Brave was:

namedprocess_namegroup_memory_bytes{

  groupname="brave",

  memtype="resident"

}

11948605440

This is approximately:

11.13 GiB

So Brave was actually using approximately **11.1 GiB of resident RAM** according to process-exporter.

The huge:

54.3 TiB

value came from:

memtype="virtual"

and should not be interpreted as RAM consumption.

---

# Process RAM Percentage

To show how much of total system RAM each process uses, the recommended PromQL is:

topk(

  10,

  sum by (groupname) (

    namedprocess_namegroup_memory_bytes{

      memtype="resident"

    }

  )

)

/

scalar(

  node_memory_MemTotal_bytes

)

* 100

### Unit

Percent (0–100)

Example:

Process       RAM        System RAM

------------------------------------

brave         11.13 GiB    34.8%

obsidian       2.92 GiB     9.1%

java            ...         ...

The `scalar()` conversion is useful because total system memory is a single Prometheus value and needs to be treated as a scalar when dividing the per-process values.

---

# Combining Multiple Queries in a Grafana Table

The user wanted to create a table using values from multiple queries grouped by a field.

The intended table is:

Process       RAM        RAM %

--------------------------------

brave         11.13 GiB   34.8%

obsidian       2.92 GiB    9.1%

java            ...        ...

The common grouping field is:

groupname

### Grafana approach

Use:

Transformations

    ↓

Join by field

    ↓

Field: groupname

Then:

Organize fields

Rename:

groupname → Process

Value → RAM

Value → RAM %

---

# Important Join Consideration

Using `topk(10, ...)` independently in several queries can cause mismatched process sets.

For example:

Query A: top 10 by RAM

Query B: top 10 by threads

may contain different processes.

This can result in missing values after joining.

For RAM and RAM percentage, this is not a problem if both are based on the same resident-memory ranking.

For example:

### Query A — RAM

topk(

  10,

  sum by (groupname) (

    namedprocess_namegroup_memory_bytes{

      memtype="resident"

    }

  )

)

### Query B — RAM %

topk(

  10,

  sum by (groupname) (

    namedprocess_namegroup_memory_bytes{

      memtype="resident"

    }

  )

)

/

scalar(node_memory_MemTotal_bytes)

* 100

Both queries therefore return the same top 10 process groups.

---

# Grafana Binary Operation Discussion

The user asked whether an existing Grafana query for total RAM could be used in:

Add field from calculation

→ Binary operation

### Answer

Yes, if the total-RAM query is another query in the same Grafana panel, it can potentially be used in a binary operation.

However, Grafana needs the query results to be aligned correctly.

The desired calculation is:

Process RAM / Total RAM × 100

The complication is that:

Process RAM

contains multiple rows:

brave

obsidian

java

code

...

while:

Total RAM

is a single scalar value.

Grafana may therefore require additional transformations to repeat/align the total RAM value across process rows.

### Recommended approach

Do the percentage calculation in PromQL instead:

topk(

  10,

  sum by (groupname) (

    namedprocess_namegroup_memory_bytes{

      memtype="resident"

    }

  )

)

/

scalar(node_memory_MemTotal_bytes)

* 100

This is simpler and more reliable.

The existing total-RAM query can still be used separately for Grafana thresholds/configuration.

---

# Final Dashboard Layout

The final recommended Process Dashboard is:

┌─────────────────────────────────────────────────────────────────────┐

│ 🔥 PROCESS DASHBOARD                                                │

├─────────────────────────────────────────────────────────────────────┤

│                                                                     │

│ TOP PROCESSES                                                       │

│                                                                     │

│ ┌───────────────────────────────┬─────────────────────────────────┐ │

│ │ 🔥 CPU                       │ 🧠 MEMORY                       │ │

│ │                               │                                 │ │

│ │ Process       CPU             │ Process       RAM       % RAM   │ │

│ │ ───────────────────────────   │ ─────────────────────────────── │ │

│ │ java          584%            │ brave        11.13 GiB   34.8% │ │

│ │ brave         101%            │ obsidian      2.92 GiB    9.1% │ │

│ │ code           42%            │ java             ...      ...  │ │

│ │ python         18%            │ code             ...      ...  │ │

│ └───────────────────────────────┴─────────────────────────────────┘ │

│                                                                     │

│ TRENDS                                                              │

│                                                                     │

│ ┌────────────────────────────────┬────────────────────────────────┐ │

│ │ CPU by Process                 │ Memory by Process               │ │

│ │                                │                                │ │

│ │ Top 5 processes over time      │ Top 5 processes over time      │ │

│ │                                │                                │ │

│ └────────────────────────────────┴────────────────────────────────┘ │

│                                                                     │

│ ┌─────────────────────────────────────────────────────────────────┐ │

│ │ 🧵 Process Threads                                               │ │

│ │                                                                 │ │

│ │ java       ████████████████████████████████ 84                 │ │

│ │ code       █████████                           24               │ │

│ │ brave      ███████                             18               │ │

│ │ python     █████                               12               │ │

│ └─────────────────────────────────────────────────────────────────┘ │

└─────────────────────────────────────────────────────────────────────┘

---

# Final Panel Specifications

|Panel|Visualization|Unit|Main query|
|---|---|---|---|
|Top Processes by CPU|Table|Percent|`topk(10, sum by (groupname)(rate(namedprocess_namegroup_cpu_seconds_total[5m])) * 100)`|
|Top Processes by Memory|Table|bytes (IEC)|`topk(10, sum by (groupname)(namedprocess_namegroup_memory_bytes{memtype="resident"}))`|
|RAM % of System|Table field / separate query|Percent|Process resident RAM ÷ `node_memory_MemTotal_bytes` × 100|
|CPU by Process|Time series|Percent|Top 5 process CPU over time|
|Memory by Process|Time series|bytes (IEC)|Top 5 process resident memory over time|
|Process Threads|Bar gauge|none|`topk(10, sum by (groupname)(namedprocess_namegroup_num_threads))`|

---

# Key Decisions

1. **Group by `groupname`** to avoid dozens of duplicate process rows.
2. Use **`memtype="resident"`** for actual process RAM.
3. Never use **`memtype="virtual"`** as physical RAM.
4. Process CPU can legitimately exceed **100%** because it represents CPU-core utilization.
5. Do not cap process CPU at 100%.
6. Use `topk(10)` for the main process tables.
7. Use `topk(5)` for time-series process charts to keep the dashboard readable.
8. Use **PromQL** for RAM percentage calculation instead of relying on Grafana binary operations against a scalar total-RAM query.
9. Use Grafana **Join by field → `groupname`** when combining multiple process queries into a table.
10. Keep the dashboard focused on **investigation**, not every individual process.

## Recommended investigation workflow

Host Overview

     │

     ├── CPU high?

     │

     ▼

Process Dashboard

     │

     ├── Top CPU process

     ├── Top RAM process

     └── Thread count

     │

     ▼

Docker Dashboard

     │

     └── Identify container/application

This makes the Process Dashboard the bridge between **host-level symptoms** and the **specific application/process causing them**.