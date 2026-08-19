# Process Exporter Investigation — Conversation Summary

## 1. Problem

The Grafana process dashboard appeared to show memory values that did not exactly match the memory values shown by Ubuntu's GNOME System Monitor.

Example from the screenshot:

| Process | GNOME System Monitor | Grafana |
|---|---:|---:|
| `python3` | 518.65 MB | 493 MiB |
| `dcgm-exporter` | 484.97 MB | 429 MiB |
| `alloy` | Not visible in the shown list | 228 MiB |
| `cadvisor` | Not visible in the shown list | 153 MiB |
| `process-exporter` | Not visible in the shown list | 18.6 MiB |

GNOME System Monitor also showed many individual `brave` processes, while the Grafana process table was displaying grouped processes such as `python3`, `dcgm-exporter`, `alloy`, and `cadvisor`.

## 2. Investigation

The relevant process-exporter metric is:

```promql
namedprocess_namegroup_memory_bytes
```

For resident memory, the important selector is:

memtype="resident"

A basic query to inspect the exported values is:

namedprocess_namegroup_memory_bytes{memtype="resident"}

A more targeted query is:

namedprocess_namegroup_memory_bytes{

  groupname=~".*(python3|dcgm-exporter|alloy|cadvisor).*",

  memtype="resident"

}

The process count can also be checked with:

namedprocess_namegroup_num_procs

## 3. Key Finding

The differences do **not** indicate that process-exporter is necessarily broken.

There are two important reasons for the apparent mismatch:

### A. Different sampling times

GNOME System Monitor and Prometheus/process-exporter do not necessarily read the process memory at exactly the same moment.

Memory usage can change between samples, particularly for:

- Python processes
- `dcgm-exporter`
- Browsers such as Brave
- Other long-running services

Therefore, values such as:

- `python3`: 518.65 MB vs 493 MiB
- `dcgm-exporter`: 484.97 MB vs 429 MiB

can occur without indicating an exporter failure.

### B. Process-exporter groups processes

Process-exporter can group multiple OS processes under one `groupname`.

For example, several Python processes may be represented by:

groupname="python3"

The resulting metric represents the memory for the process group, rather than necessarily one individual PID.

This explains why GNOME System Monitor can show many individual `brave` processes while Grafana can show grouped process entries.

## 4. Recommended Grafana Query

For a "Top Processes" table showing current resident memory, use:

sort_desc(

  namedprocess_namegroup_memory_bytes{memtype="resident"}

)

For a panel intended to show the current values, enable Grafana's **Instant** query option so that the table receives the current point rather than a time-series range.

Avoid adding `sum()` unless there is a specific need to combine duplicate series, because process-exporter already aggregates processes according to its configured groups.

## 5. Validation Steps

To verify the exporter independently of Grafana:

1. Open Prometheus.
2. Run:

namedprocess_namegroup_memory_bytes{memtype="resident"}

3. Check the values for `python3`, `dcgm-exporter`, and other groups.
4. Compare those values with GNOME System Monitor at approximately the same time.
5. Check:

namedprocess_namegroup_num_procs

to determine how many processes belong to each group.  
6. Confirm the process-exporter grouping configuration if a group contains unexpected processes.

## 6. Final Conclusion

The evidence from the screenshot does **not** point to a broken process-exporter.

The observed discrepancy is primarily explained by:

- different sampling times between GNOME System Monitor and Prometheus,
- process grouping performed by process-exporter,
- and potentially Grafana's query mode/time range.

The recommended final Grafana approach is:

sort_desc(

  namedprocess_namegroup_memory_bytes{memtype="resident"}

)

with the panel configured as an **Instant** query when the goal is to display the current top processes.

If the values still differ substantially after comparing Prometheus and GNOME System Monitor at the same time, the next item to inspect is the exact process-exporter configuration and the exact Grafana query/transformations.