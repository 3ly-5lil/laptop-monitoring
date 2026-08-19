# Grafana CPU Core Ordering — Conversation Summary

## Environment

- Grafana: **v13.1.3 (45a27d64b6)**
- Data source: **Prometheus**
- Visualization: **Time series**
- Display style: **stacked lines**
- Metric: `node_cpu_seconds_total`
- CPU count in the discussed system: **12 logical CPUs (`0`–`11`)**

---

## Problem

The panel uses this display name:
```text
CPU ${__field.labels.cpu}
```

and this PromQL query:

100 * (1 - rate(node_cpu_seconds_total{mode="idle"}[5m]))

Prometheus returns time series such as:

{cpu="0", instance="node-exporter:9100", job="node", mode="idle"}

{cpu="1", instance="node-exporter:9100", job="node", mode="idle"}

{cpu="10", instance="node-exporter:9100", job="node", mode="idle"}

{cpu="11", instance="node-exporter:9100", job="node", mode="idle"}

{cpu="2", instance="node-exporter:9100", job="node", mode="idle"}

{cpu="3", instance="node-exporter:9100", job="node", mode="idle"}

Grafana therefore displays the series in lexicographic/string order:

CPU 0

CPU 1

CPU 10

CPU 11

CPU 2

CPU 3

...

The desired order is numeric:

CPU 0

CPU 1

CPU 2

CPU 3

CPU 4

CPU 5

CPU 6

CPU 7

CPU 8

CPU 9

CPU 10

CPU 11

### Root cause

The `cpu` value is a **Prometheus label**, so Grafana treats the label value as a string when determining series/field ordering. String sorting places `"10"` and `"11"` before `"2"`.

The PromQL itself is correct; the issue is the ordering of the resulting series in Grafana.

---

## Attempted transformation workaround

A transformation chain was tried:

1. **Labels to fields**
2. **Convert field type**
    - `cpu` → `Number`
3. **Sort by**
    - field: `cpu`
    - ascending

The configuration looked approximately like:

Labels to fields

  Mode: Columns

  Labels: cpu

  

Convert field type

  Field: cpu

  As: Number

  

Sort by

  Field: cpu

  Reverse: off

### Result

This did successfully expose `cpu` as a numeric field and sort rows numerically, but it changed the data-frame structure.

The Time series panel then displayed rows similar to:

cpu

Value

cpu

Value

cpu

Value

...

instead of treating each CPU as an independent time-series field.

### Why this did not solve the problem

`Labels to fields` converts labeled series into a table-like structure. `Sort by` sorts **rows in the data frame**, not the ordering of independent time-series series in the way required by the Time series visualization.

Therefore, this transformation chain is not appropriate for the desired **Time series + stacked lines** result.

---

## Important clarification

The following idea was considered:

sort_by_label(...)

but this is not a reliable solution for this panel because the Grafana Time series panel uses range queries, while Prometheus's `sort_by_label()` behavior is intended for instant-vector results and does not provide the required ordering control for range-query time-series fields.

Similarly, simply converting the `cpu` label to a numeric field does not automatically reorder the independent Prometheus series in the Time series visualization.

---

## Current/final conclusion

The core problem is:

Prometheus label:

cpu="10"

is a string label, and Grafana's Time series series ordering is therefore effectively lexical:

0, 1, 10, 11, 2, 3, ...

The attempted:

Labels to fields

→ Convert field type

→ Sort by

approach should **not** be used for this Time series panel because it changes the data into a table-like structure.

The original query should remain:

100 * (1 - rate(node_cpu_seconds_total{mode="idle"}[5m]))

and the display name should remain:

CPU ${__field.labels.cpu}

---

## Potential clean solutions

### 1. Zero-pad the CPU label

The most robust general concept is to give Grafana a sortable representation:

cpu_sort="00"

cpu_sort="01"

cpu_sort="02"

...

cpu_sort="09"

cpu_sort="10"

cpu_sort="11"

Lexicographic sorting then produces the correct numeric order.

The visible CPU label can remain unpadded:

CPU 0

CPU 1

CPU 2

...

CPU 10

CPU 11

However, PromQL labels are still strings, so creating a padded label alone does not automatically force Grafana's range-query Time series series order unless that label is used by the relevant Grafana ordering mechanism.

### 2. Manual series ordering

If the dashboard is specifically for a fixed 12-core machine, manually controlling series order can be practical if the installed Grafana visualization exposes a series-order control.

The desired explicit order is:

CPU 0

CPU 1

CPU 2

CPU 3

CPU 4

CPU 5

CPU 6

CPU 7

CPU 8

CPU 9

CPU 10

CPU 11

Do not create many field overrides unless the Grafana version actually exposes a property that controls **series order**. A normal field override does not inherently reorder the Time series data.

### 3. Change the data representation upstream

For a reusable dashboard, the ideal design is to have the query/exporter pipeline expose a sortable series identifier while retaining the original CPU label for display.

For example:

cpu="0",  cpu_sort="00"

cpu="1",  cpu_sort="01"

...

cpu="9",  cpu_sort="09"

cpu="10", cpu_sort="10"

cpu="11", cpu_sort="11"

This separates:

- **display identity:** `cpu`
- **sorting identity:** `cpu_sort`

This is preferable to changing the original `cpu` label.

---

## Final recommended state for the current panel

Keep:

### PromQL

100 * (1 - rate(node_cpu_seconds_total{mode="idle"}[5m]))

### Display name

CPU ${__field.labels.cpu}

### Visualization

Time series

### Style

Stacked lines

### Transformations

Remove the attempted:

Labels to fields

Convert field type

Sort by

because that transformation chain converts the data into a table-oriented structure and does not provide the desired numeric ordering of independent Time series.

---

## Key takeaway

The issue is **not CPU calculation** and **not PromQL correctness**.

It is a **series-ordering problem caused by the `cpu` label being a string**:

"0", "1", "10", "11", "2", ...

The desired behavior requires controlling the ordering of the actual Time series fields, rather than sorting rows after converting labels to fields.