---
name: observe-metrics-reference
description: Reference for Prometheus metrics queries, available labels, and metrics tables for storage (Ceph/ODF), network traffic, pod/container statistics, and Forklift/MTV migrations. Use alongside the observe-metrics skill for detailed per-domain queries and label filters.
---

# Metrics Reference

Per-domain queries, available labels, and metrics tables for OpenShift clusters with ODF, OVN-Kubernetes, KubeVirt, and Forklift/MTV.

All examples assume `$THANOS_URL` and `$TOKEN` are set (see the `observe-metrics` skill, Step 1).

---

## Storage Metrics (Ceph / ODF)

### Cluster-wide storage health

```bash
# Ceph health status (0=OK, 1=WARN, 2=ERR)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=ceph_health_status'
```

### Storage capacity

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=ceph_cluster_total_bytes'

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=ceph_cluster_total_used_bytes'
```

### Pool-level statistics

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=ceph_pool_percent_used * 100'
```

### Pool I/O rates

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=rate(ceph_pool_rd[5m])'

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=rate(ceph_pool_wr[5m])'
```

### OSD operation latency

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=rate(ceph_osd_op_latency_sum[5m]) / rate(ceph_osd_op_latency_count[5m])'
```

### Placement group health

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=ceph_pg_total'

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=ceph_pg_degraded'
```

### Available labels on ceph_* metrics

| Label | Description | Example values |
|-------|-------------|----------------|
| `pool_id` | Ceph pool identifier (pool-level metrics) | `1`, `2`, `3`, `4` |
| `ceph_daemon` | OSD daemon name (OSD-level metrics) | `osd.0`, `osd.1`, `osd.2` |
| `namespace` | Storage operator namespace | `openshift-storage` |
| `managedBy` | Managing resource | `ocs-storagecluster` |
| `job` | Scrape job | `rook-ceph-mgr`, `rook-ceph-exporter` |

### Storage metrics reference

| Metric | Description |
|--------|-------------|
| `ceph_health_status` | Overall cluster health (0=OK, 1=WARN, 2=ERR) |
| `ceph_cluster_total_bytes` | Total cluster capacity |
| `ceph_cluster_total_used_bytes` | Used cluster capacity |
| `ceph_pool_percent_used` | Per-pool usage percentage |
| `ceph_pool_stored` | Bytes stored per pool |
| `ceph_pool_max_avail` | Available bytes per pool |
| `ceph_pool_rd`, `ceph_pool_wr` | Read/write IOPS per pool |
| `ceph_pool_rd_bytes`, `ceph_pool_wr_bytes` | Read/write bytes per pool |
| `ceph_osd_op_latency_sum/count` | OSD operation latency (use as rate ratio) |
| `ceph_pg_total`, `ceph_pg_active`, `ceph_pg_degraded` | Placement group counts |
| `node_filesystem_avail_bytes`, `node_filesystem_size_bytes` | Node filesystem capacity |

---

## Network Traffic Metrics

### Network traffic by namespace

```bash
# Receive bytes/sec by namespace (top 10)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(10, sort_desc(sum by (namespace)(rate(container_network_receive_bytes_total[5m]))))'

# Transmit bytes/sec by namespace (top 10)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(10, sort_desc(sum by (namespace)(rate(container_network_transmit_bytes_total[5m]))))'
```

### Network traffic by pod in a namespace

```bash
# Replace TARGET_NAMESPACE with the namespace to inspect
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(10, sort_desc(sum by (pod)(rate(container_network_receive_bytes_total{namespace="TARGET_NAMESPACE"}[5m]))))'

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(10, sort_desc(sum by (pod)(rate(container_network_transmit_bytes_total{namespace="TARGET_NAMESPACE"}[5m]))))'
```

### Network errors and drops by namespace

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(10, sum by (namespace)(rate(container_network_receive_errors_total[5m])) + sum by (namespace)(rate(container_network_transmit_errors_total[5m])))'
```

### Node-level network throughput

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=instance:node_network_receive_bytes_excluding_lo:rate1m + instance:node_network_transmit_bytes_excluding_lo:rate1m'
```

### Available labels on network metrics

| Label | Description | Example values |
|-------|-------------|----------------|
| `namespace` | Pod namespace | `openshift-storage`, `konveyor-forklift` |
| `pod` | Pod name | `forklift-controller-6df77f6bf5-jtt7q` |
| `interface` | Network interface (per-pod metrics) | `eth0` |
| `instance` | Node instance (node-level metrics) | `10.0.0.5:9100` |
| `node` | Node name (node-level metrics) | `worker-0` |

### Network metrics reference

| Metric | Description |
|--------|-------------|
| `container_network_receive_bytes_total` | Bytes received per pod/namespace |
| `container_network_transmit_bytes_total` | Bytes transmitted per pod/namespace |
| `container_network_receive_errors_total` | Receive errors per pod/namespace |
| `container_network_transmit_errors_total` | Transmit errors per pod/namespace |
| `container_network_receive_packets_dropped_total` | Dropped receive packets |
| `container_network_transmit_packets_dropped_total` | Dropped transmit packets |
| `node_network_receive_bytes_total` | Bytes received per node/interface |
| `node_network_transmit_bytes_total` | Bytes transmitted per node/interface |
| `instance:node_network_receive_bytes_excluding_lo:rate1m` | Pre-computed node receive rate |

---

## Pod and Container Statistics

### Pod count by namespace

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(15, count by (namespace)(kube_pod_info))'
```

### Pod phase summary

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=count by (phase)(kube_pod_status_phase == 1)'
```

### Container CPU usage by namespace

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(10, sort_desc(sum by (namespace)(namespace:container_cpu_usage:sum)))'
```

### Container memory usage by namespace

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(10, sort_desc(sum by (namespace)(namespace:container_memory_usage_bytes:sum)))'
```

### Container restart counts (instability indicator)

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(10, sort_desc(kube_pod_container_status_restarts_total))'
```

### Available labels on pod/container metrics

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

### Pod/container metrics reference

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

---

## Forklift / MTV Migration Metrics

### Available labels on mtv_* metrics

All `mtv_*` metrics share these labels for filtering and grouping:

| Label | Description | Example values |
|-------|-------------|----------------|
| `provider` | Source provider type | `vsphere`, `ovirt`, `openstack`, `ova`, `ec2` |
| `mode` | Migration mode | `Cold`, `Warm` |
| `target` | Target cluster | `Local` (host cluster) or remote cluster name |
| `owner` | User who owns the migration | `admin@example.com` |
| `plan` | Migration plan UUID | `363ce137-dace-4fb4-b815-759c214c9fec` |
| `namespace` | Forklift operator namespace | `konveyor-forklift`, `openshift-mtv` |
| `status` | Migration/plan status (on status metrics) | `Succeeded`, `Failed`, `Executing` |

### MTV migration metrics reference

| Metric | Description |
|--------|-------------|
| `mtv_migrations_status_total` | Migration counts by status (succeeded/failed/running) |
| `mtv_plans_status` | Plan-level status counts |
| `mtv_migration_data_transferred_bytes` | Total bytes migrated per plan |
| `mtv_migration_net_throughput` | Migration network throughput |
| `mtv_migration_storage_throughput` | Migration storage throughput |
| `mtv_migration_duration_seconds` | Migration duration per plan |
| `mtv_plan_alert_status` | Alerts on migration plans |
| `mtv_workload_migrations_status_total` | Per-workload migration status (per plan + status) |
| `kubevirt_vmi_migrations_in_pending_phase` | Live VMI migrations pending |
| `kubevirt_vmi_migrations_in_running_phase` | Live VMI migrations in progress |

### Migration status overview

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migrations_status_total'
```

### Migration plan status

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_plans_status'
```

### Migration data transfer and throughput

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migration_data_transferred_bytes'

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migration_net_throughput'

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migration_storage_throughput'
```

### Migration duration

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migration_duration_seconds'
```

### Migration alerts

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_plan_alert_status'
```

### Narrowing migration metrics with label filters

Use `{label="value"}` in PromQL to narrow results. Filters can be combined.

```bash
# Only vSphere migrations
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migration_data_transferred_bytes{provider="vsphere"}'

# Only cold migrations
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migration_data_transferred_bytes{mode="Cold"}'

# Only warm migrations from oVirt
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migration_data_transferred_bytes{provider="ovirt", mode="Warm"}'

# A specific plan by UUID
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migration_data_transferred_bytes{plan="PLAN_UUID"}'

# Only failed migrations
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migrations_status_total{status="Failed"}'

# Failed workload migrations for a specific plan
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_workload_migrations_status_total{plan="PLAN_UUID", status="Failed"}'
```

### Grouping migration metrics with sum by / count by

Use `sum by (label)` to aggregate across dimensions.

```bash
# Total bytes transferred grouped by provider type
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=sum by (provider)(mtv_migration_data_transferred_bytes)'

# Total bytes transferred grouped by migration mode (Cold vs Warm)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=sum by (mode)(mtv_migration_data_transferred_bytes)'

# Total bytes transferred grouped by provider and mode
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=sum by (provider, mode)(mtv_migration_data_transferred_bytes)'

# Migration counts grouped by status and provider
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=sum by (status, provider)(mtv_migrations_status_total)'

# Average migration duration grouped by provider
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=avg by (provider)(mtv_migration_duration_seconds)'

# Workload migration status counts grouped by plan and status
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=sum by (plan, status)(mtv_workload_migrations_status_total)'

# Plan status grouped by provider (how many plans succeeded/failed per provider)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=sum by (provider, status)(mtv_plans_status)'
```

### Network traffic of migration pods

During active Forklift migrations, data-transfer pods (virt-v2v, populator, importer) run in the target namespace:

```bash
# Find namespaces with active migration pods
kubectl get pods --all-namespaces --no-headers 2>/dev/null \
  | grep -E 'virt-v2v|populator|importer' | awk '{print $1}' | sort -u
```

```bash
# Migration pod network receive (bytes/sec)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(20, sort_desc(sum by (namespace,pod)(rate(container_network_receive_bytes_total{pod=~".*virt-v2v.*|.*populator.*|.*importer.*|.*cdi-upload.*"}[5m]))))'

# Migration pod network transmit (bytes/sec)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(20, sort_desc(sum by (namespace,pod)(rate(container_network_transmit_bytes_total{pod=~".*virt-v2v.*|.*populator.*|.*importer.*|.*cdi-upload.*"}[5m]))))'
```

```bash
# Migration pod traffic in a specific namespace only
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=sum by (pod)(rate(container_network_receive_bytes_total{namespace="TARGET_NAMESPACE", pod=~".*virt-v2v.*|.*populator.*|.*importer.*"}[5m]))'

# Total migration pod traffic grouped by namespace
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=sum by (namespace)(rate(container_network_receive_bytes_total{pod=~".*virt-v2v.*|.*populator.*|.*importer.*|.*cdi-upload.*"}[5m]))'
```

### Network traffic of the Forklift operator itself

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=sum by (pod)(rate(container_network_receive_bytes_total{pod=~"forklift.*"}[5m]))'
```

### KubeVirt VMI migration metrics

These track live VM migrations (vMotion-style), not Forklift cold migrations:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=kubevirt_vmi_migrations_in_pending_phase'

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=kubevirt_vmi_migrations_in_scheduling_phase'

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=kubevirt_vmi_migrations_in_running_phase'
```

---

## Quick Health Dashboard

Run key queries at once for an overview:

```bash
THANOS_URL=$(kubectl get route thanos-querier -n openshift-monitoring -o jsonpath='{.status.ingress[0].host}')
TOKEN=$(oc create token prometheus-k8s -n openshift-monitoring)

echo "=== Node CPU Usage ==="
curl -sk -H "Authorization: Bearer $TOKEN" "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=100 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m]))*100'

echo ""
echo "=== Node Memory Usage ==="
curl -sk -H "Authorization: Bearer $TOKEN" "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=(1 - node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes)*100'

echo ""
echo "=== Ceph Health ==="
curl -sk -H "Authorization: Bearer $TOKEN" "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=ceph_health_status'

echo ""
echo "=== Top 5 Namespaces by Network RX ==="
curl -sk -H "Authorization: Bearer $TOKEN" "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=topk(5, sum by (namespace)(rate(container_network_receive_bytes_total[5m])))'

echo ""
echo "=== MTV Migrations ==="
curl -sk -H "Authorization: Bearer $TOKEN" "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=mtv_migrations_status_total'
```
