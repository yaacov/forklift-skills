---
name: observe-metrics
description: Observe cluster metrics via Prometheus/Thanos. Use when the user wants to check cluster metrics, monitor network traffic, storage I/O, pod resource usage, VM migration throughput, or discover available Prometheus metrics. Covers metric discovery, storage (Ceph/ODF), network traffic by namespace/pod, pod statistics, and Forklift/MTV migration monitoring.
---

# Observe Cluster Metrics

Use this guide to discover and query Prometheus/Thanos metrics on an OpenShift cluster using the `metrics_read` MCP tool. The MCP server handles authentication and routing automatically.

**Important — combine related metrics:** When the user asks about related metrics
(e.g. network RX and TX, CPU and memory, storage read and write), always use a single
`query_range` call with arrays in the `query` and `name` flags. This produces aligned
timestamps, a single multi-column result, and requires only one MCP call.

For detailed per-domain queries, labels, and metrics tables:
- Storage (Ceph/ODF): [ref-storage.md](ref-storage.md)
- Network traffic: [ref-network.md](ref-network.md)
- Pods and containers: [ref-pods.md](ref-pods.md)
- KubeVirt VMs: [ref-vms.md](ref-vms.md)
- Forklift/MTV migrations: [ref-mtv.md](ref-mtv.md)

## Required MCP Servers

This skill requires: `metrics_read` (from the kubectl-metrics MCP server).

If `metrics_read` is not available in your environment, inform the user and refer them to the `mcp-setup` skill for installation instructions. Do not attempt bash fallback.

## Getting Help

Before querying, call `metrics_help` to learn available subcommands and flags:

```
metrics_help                                -- overview of all subcommands
metrics_help  command: "query"              -- flags for instant queries
metrics_help  command: "query_range"        -- flags for range queries
metrics_help  command: "discover"           -- flags for metric discovery
metrics_help  command: "promql"             -- PromQL syntax reference
```

## Step 1: Discover Available Metrics

### List all metric names (or search by keyword)

```
metrics_read  command: "discover"
metrics_read  command: "discover"  flags: {keyword: "ceph"}
metrics_read  command: "discover"  flags: {keyword: "kubevirt"}
metrics_read  command: "discover"  flags: {keyword: "mtv"}
```

### Group metric names by prefix

```
metrics_read  command: "discover"  flags: {keyword: "mtv", group_by_prefix: true}
```

### List labels for a specific metric

```
metrics_read  command: "labels"  flags: {metric: "container_network_receive_bytes_total"}
```

## Step 2: Instant Queries

Use instant queries for point-in-time health checks:

```
metrics_read  command: "query"  flags: {query: "up"}
metrics_read  command: "query"  flags: {query: "ceph_health_status"}
metrics_read  command: "query"  flags: {query: "count by (phase)(kube_pod_status_phase == 1)"}
```

## Step 3: Range Queries (Time-Series Trends)

Use `query_range` for time-series data. Pass `query` and `name` as arrays to fetch
multiple related metrics in a single call.

### Single metric trend

```
metrics_read  command: "query_range"  flags: {
  query: "rate(http_requests_total[5m])",
  start: "-1h",
  step: "60s"
}
```

### Multi-metric trend (preferred for related metrics)

Combine related metrics in one call — each query gets its own named column:

```
metrics_read  command: "query_range"  flags: {
  query: ["sum(rate(container_network_receive_bytes_total{namespace=\"TARGET_NS\"}[5m]))",
          "sum(rate(container_network_transmit_bytes_total{namespace=\"TARGET_NS\"}[5m]))"],
  name: ["rx_bytes_per_sec", "tx_bytes_per_sec"],
  start: "-1h",
  step: "60s"
}
```

```
metrics_read  command: "query_range"  flags: {
  query: ["sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)",
          "sum(container_memory_working_set_bytes) by (namespace)"],
  name: ["cpu", "mem"],
  start: "-1h"
}
```

### Filtering results

Use PromQL label selectors directly in the query to narrow results:

```
metrics_read  command: "query_range"  flags: {
  query: "rate(container_network_receive_bytes_total{namespace=\"konveyor-forklift\"}[5m])",
  start: "-1h"
}
```

Selector operators: `=` (equal), `!=` (not equal), `=~` (regex), `!~` (negative regex). Combine with commas: `namespace="prod",pod=~"nginx.*"`.

## Quick Health Dashboard

Run these queries for a cluster overview:

```
metrics_read  command: "query_range"  flags: {
  query: ["avg(instance:node_cpu:ratio) * 100",
          "(1 - sum(node_memory_MemAvailable_bytes) / sum(node_memory_MemTotal_bytes)) * 100"],
  name: ["cpu_pct", "mem_pct"],
  start: "-1h"
}
metrics_read  command: "query"  flags: {query: "sum(kube_node_status_condition{condition='Ready',status='true'})"}
metrics_read  command: "query"  flags: {query: "count by (phase)(kube_pod_status_phase == 1)"}
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(kube_pod_container_status_restarts_total))"}
metrics_read  command: "query"  flags: {query: "ceph_health_status"}
```

## Visualizing Range Queries with gnuplot

When the user asks for a chart, graph, or visualization of metrics, use `gnuplot` to
open an interactive window. Use the `filename` flag so the MCP server writes TSV directly
to a temp file — the LLM never needs to see or copy the raw data.

### Steps

1. Run the range query with `output: "tsv"` and `filename: "metrics.tsv"`.
   The MCP server writes the data to a temp file and returns a short summary
   with the full file path, row count, and column names.
2. Extract the full file path from the summary and build a gnuplot script that
   reads from it. Run `gnuplot -p`.

### Example metrics call

```
metrics_read  command: "query_range"  flags: {
  query: ["sum(rate(container_network_receive_bytes_total{namespace=\"konveyor-forklift\"}[5m]))",
          "sum(rate(container_network_transmit_bytes_total{namespace=\"konveyor-forklift\"}[5m]))"],
  name: ["rx_bytes_per_sec", "tx_bytes_per_sec"],
  start: "-24h",
  step: "5m",
  output: "tsv",
  filename: "metrics.tsv"
}
```

The response will be short, e.g.: `Wrote 288 rows to /var/folders/.../T/metrics.tsv\nColumns: timestamp  rx_bytes_per_sec  tx_bytes_per_sec`

Use the full path from the response in the gnuplot script.

### gnuplot template

Replace `FILE_PATH` with the full path from the metrics_read response:

```bash
gnuplot -p <<'GP'
set terminal qt size 900,500 font "Helvetica,11"
set datafile separator "\t"
set xdata time
set timefmt "%s"
set format x "%H:%M"
set xlabel "Time"
set ylabel "UNIT"
set title "TITLE"
set grid
set key outside right top
plot "FILE_PATH" using 1:2 with lines lw 2 title "COL2", \
     "FILE_PATH" using 1:3 with lines lw 2 title "COL3"
GP
```

### Adapting the template

- Replace `FILE_PATH` with the full path returned by `metrics_read` in its summary.
- Replace `TITLE`, `UNIT`, and column titles with descriptive values from the query.
- Use the column names from the summary returned by `metrics_read` for the plot titles.
- Add one `using 1:N` clause per data column (skip the header row automatically).
- For a single data column, drop the `\` continuation and use only one `plot` entry.
- Use `set format x "%m/%d %H:%M"` when the range spans multiple days.
- The `qt` terminal requires GUI access. If running in a sandbox, request unsandboxed
  execution (e.g., `required_permissions: ["all"]`), otherwise the window will fail silently.
- If `gnuplot` or the `qt` terminal is not available, fall back to `set terminal dumb size 120 30` for ASCII output in the shell.
- Multi-query range results produce multi-column TSV — one column per named query.
- **Always pass the `filename` flag** for range queries intended for gnuplot. This keeps
  the MCP response small and avoids slow token generation.

## PromQL Quick Reference

### Selecting metrics

```
metric_name                          all time series for this metric
metric_name{label="value"}           filter by exact label match
metric_name{label=~"pattern.*"}      filter by regex match
metric_name{label!="value"}          exclude a label value
metric_name{l1="a", l2="b"}         combine multiple filters
```

### Rate and increase (for counters)

Counters only go up. Use `rate` or `increase` to get meaningful values:

```
rate(metric[5m])                     per-second rate over 5 minutes
increase(metric[1h])                 total increase over 1 hour
```

### Aggregation

```
sum(metric)                          total across all series
sum by (label)(metric)               total grouped by label
avg by (label)(metric)               average grouped by label
count by (label)(metric)             count of series grouped by label
topk(10, metric)                     top 10 series by value
sort_desc(metric)                    sort descending
```

### Arithmetic

```
metric_a / metric_b                  ratio of two metrics
metric * 100                         scale a metric
1 - (available / total)              compute used percentage
```

### Common patterns

```
topk(10, sort_desc(sum by (namespace)(rate(container_network_receive_bytes_total[5m]))))
rate(ceph_osd_op_latency_sum[5m]) / rate(ceph_osd_op_latency_count[5m])
100 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
```

## Self-Learning Rule

When you need to discover available flags, build custom queries, or verify syntax:

```
metrics_help  command: "query"
metrics_help  command: "query_range"
metrics_help  command: "discover"
metrics_help  command: "promql"
```
