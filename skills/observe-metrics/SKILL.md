---
name: observe-metrics
description: Observe cluster metrics via Prometheus/Thanos. Use when the user wants to check cluster metrics, monitor network traffic, storage I/O, pod resource usage, VM migration throughput, or discover available Prometheus metrics. Covers metric discovery, storage (Ceph/ODF), network traffic by namespace/pod, pod statistics, and Forklift/MTV migration monitoring.
---

# Observe Cluster Metrics

Use this guide to discover and query Prometheus/Thanos metrics on an OpenShift cluster using the `metrics_read` MCP tool. The MCP server handles authentication and routing automatically.

For detailed per-domain queries, labels, and metrics tables see [reference.md](reference.md).

## Required MCP Servers

This skill requires: `metrics_read` (from the kubectl-metrics MCP server).

If `metrics_read` is not available in your environment, inform the user and refer them to the `mcp-setup` skill for installation instructions. Do not attempt bash fallback.

## Getting Help

Before querying, call `metrics_help` to learn available subcommands, flags, and presets:

```
metrics_help()                    -- overview of all subcommands and available presets
metrics_help("query")             -- flags for instant queries
metrics_help("query_range")       -- flags for range queries
metrics_help("discover")          -- flags for metric discovery
metrics_help("preset")            -- flags for preset queries
metrics_help("promql")            -- PromQL syntax reference
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

## Step 2: Query Metrics

### Instant query

```
metrics_read  command: "query"  flags: {query: "up"}
metrics_read  command: "query"  flags: {query: "ceph_health_status"}
```

### Range query (last 1 hour, 1-minute steps)

```
metrics_read  command: "query_range"  flags: {query: "rate(http_requests_total[5m])", start: "-1h", step: "60s"}
```

### Multi-query range (compare metrics side by side)

```
metrics_read  command: "query_range"  flags: {
  query: ["sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)",
          "sum(container_memory_working_set_bytes) by (namespace)"],
  name: ["cpu", "mem"],
  start: "-1h"
}
```

## Step 3: Use Presets

Presets are pre-configured named queries for common monitoring tasks. They work as both instant and range queries.

### Cluster health presets

```
metrics_read  command: "preset"  flags: {name: "cluster_cpu_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_memory_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_node_readiness"}
metrics_read  command: "preset"  flags: {name: "cluster_pod_status"}
```

### Namespace resource presets

```
metrics_read  command: "preset"  flags: {name: "namespace_cpu_usage"}
metrics_read  command: "preset"  flags: {name: "namespace_memory_usage"}
metrics_read  command: "preset"  flags: {name: "namespace_network_rx"}
metrics_read  command: "preset"  flags: {name: "namespace_network_tx"}
metrics_read  command: "preset"  flags: {name: "namespace_network_errors"}
```

### Pod presets

```
metrics_read  command: "preset"  flags: {name: "pod_restarts_top10"}
```

### MTV migration presets

```
metrics_read  command: "preset"  flags: {name: "mtv_migration_status"}
metrics_read  command: "preset"  flags: {name: "mtv_plan_status"}
metrics_read  command: "preset"  flags: {name: "mtv_migration_duration"}
metrics_read  command: "preset"  flags: {name: "mtv_avg_migration_duration"}
metrics_read  command: "preset"  flags: {name: "mtv_data_transferred"}
metrics_read  command: "preset"  flags: {name: "mtv_net_throughput"}
metrics_read  command: "preset"  flags: {name: "mtv_storage_throughput"}
metrics_read  command: "preset"  flags: {name: "mtv_migration_pod_rx"}
metrics_read  command: "preset"  flags: {name: "mtv_migration_pod_tx"}
metrics_read  command: "preset"  flags: {name: "mtv_populator_cpu"}
metrics_read  command: "preset"  flags: {name: "mtv_forklift_traffic"}
metrics_read  command: "preset"  flags: {name: "mtv_vmi_migrations_pending"}
metrics_read  command: "preset"  flags: {name: "mtv_vmi_migrations_running"}
```

### VM presets

```
metrics_read  command: "preset"  flags: {name: "vm_cpu_usage"}
metrics_read  command: "preset"  flags: {name: "vm_memory_usage"}
metrics_read  command: "preset"  flags: {name: "vm_network_rx"}
metrics_read  command: "preset"  flags: {name: "vm_network_tx"}
metrics_read  command: "preset"  flags: {name: "vm_storage_read"}
metrics_read  command: "preset"  flags: {name: "vm_storage_write"}
metrics_read  command: "preset"  flags: {name: "vm_storage_iops"}
```

### Preset as range query (time series trend)

Pass `start` to any preset to get a time-series trend instead of an instant value:

```
metrics_read  command: "preset"  flags: {name: "mtv_net_throughput", start: "-2h", step: "30s"}
metrics_read  command: "preset"  flags: {name: "cluster_cpu_utilization", start: "-1h"}
```

### Filtering preset results

Use `selector` to filter results by labels, and `group_by` to change aggregation:

```
metrics_read  command: "preset"  flags: {name: "mtv_migration_status", namespace: "mtv-test"}
metrics_read  command: "preset"  flags: {name: "mtv_migration_status", group_by: "namespace"}
metrics_read  command: "preset"  flags: {name: "namespace_cpu_usage", selector: "namespace=openshift-cnv"}
```

Selector operators: `=` (equal), `!=` (not equal), `=~` (regex), `!~` (negative regex). Combine with commas: `selector: "namespace=prod,pod=~nginx.*"`.

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
