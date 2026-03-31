# Pod and Container Metrics

Queries, labels, and metrics for pod/container statistics. See [SKILL.md](SKILL.md) for general usage.

## Pod status and restarts (presets)

```
metrics_read  command: "preset"  flags: {name: "cluster_pod_status"}
metrics_read  command: "preset"  flags: {name: "pod_restarts_top10"}
```

## CPU and memory by namespace (presets)

```
metrics_read  command: "preset"  flags: {name: "namespace_cpu_usage"}
metrics_read  command: "preset"  flags: {name: "namespace_memory_usage"}
```

## Pod count by namespace (ad-hoc)

```
metrics_read  command: "query"  flags: {query: "topk(15, count by (namespace)(kube_pod_info))"}
```

## Pod phase summary

```
metrics_read  command: "query"  flags: {query: "count by (phase)(kube_pod_status_phase == 1)"}
```

## Container restart counts (instability indicator)

```
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(kube_pod_container_status_restarts_total))"}
```

## Available labels on pod/container metrics

| Label | Description | Example values |
|-------|-------------|----------------|
| `namespace` | Pod namespace | `konveyor-forklift`, `openshift-cnv` |
| `pod` | Pod name | `forklift-controller-6df77f6bf5-jtt7q` |
| `container` | Container name | `main`, `inventory`, `extract` |
| `node` | Node the pod runs on | `worker-0`, `worker-1` |
| `phase` | Pod phase (on status metrics) | `Running`, `Pending`, `Failed`, `Succeeded` |
| `uid` | Pod UID | `793fb1cb-3e58-4eef-b95a-733f237365a3` |
| `created_by_kind` | Owner resource kind (on kube_pod_info) | `ReplicaSet`, `DaemonSet`, `StatefulSet` |
| `created_by_name` | Owner resource name (on kube_pod_info) | `forklift-controller-6df77f6bf5` |
| `host_ip` | Node IP (on kube_pod_info) | `192.168.0.77` |
| `pod_ip` | Pod IP (on kube_pod_info) | `10.129.3.3` |

## Metrics reference

| Metric | Description |
|--------|-------------|
| `kube_pod_info` | Pod metadata (node, namespace, IPs, owner) |
| `kube_pod_status_phase` | Pod phase (Running/Pending/Failed/Succeeded) |
| `kube_pod_container_status_restarts_total` | Container restart count |
| `kube_pod_container_status_waiting_reason` | Waiting reason (CrashLoopBackOff, ImagePullBackOff, etc.) |
| `container_cpu_usage_seconds_total` | Container CPU usage |
| `container_memory_working_set_bytes` | Container memory usage |
| `namespace:container_cpu_usage:sum` | Pre-aggregated CPU by namespace |
| `namespace:container_memory_usage_bytes:sum` | Pre-aggregated memory by namespace |
