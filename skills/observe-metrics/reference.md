# Metrics Reference

Per-domain queries, available labels, and metrics tables for OpenShift clusters with ODF, OVN-Kubernetes, KubeVirt, and Forklift/MTV.

All examples use `metrics_read` MCP calls. See the main [SKILL.md](SKILL.md) for setup and general usage.

---

## Storage Metrics (Ceph / ODF)

### Cluster-wide storage health

```
metrics_read  command: "query"  flags: {query: "ceph_health_status"}
```

Health values: 0=OK, 1=WARN, 2=ERR.

### Storage capacity

```
metrics_read  command: "query"  flags: {query: "ceph_cluster_total_bytes"}
metrics_read  command: "query"  flags: {query: "ceph_cluster_total_used_bytes"}
metrics_read  command: "query"  flags: {query: "ceph_cluster_total_used_bytes / ceph_cluster_total_bytes * 100"}
```

### Pool-level statistics

```
metrics_read  command: "query"  flags: {query: "ceph_pool_percent_used * 100"}
```

### Pool I/O rates

```
metrics_read  command: "query"  flags: {query: "rate(ceph_pool_rd[5m])"}
metrics_read  command: "query"  flags: {query: "rate(ceph_pool_wr[5m])"}
```

### OSD operation latency

```
metrics_read  command: "query"  flags: {query: "rate(ceph_osd_op_latency_sum[5m]) / rate(ceph_osd_op_latency_count[5m])"}
```

### Placement group health

```
metrics_read  command: "query"  flags: {query: "ceph_pg_total"}
metrics_read  command: "query"  flags: {query: "ceph_pg_degraded"}
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
| `ceph_osd_stat_bytes`, `ceph_osd_stat_bytes_used` | Per-OSD capacity and usage |
| `ceph_pg_total`, `ceph_pg_active`, `ceph_pg_degraded` | Placement group counts |
| `node_filesystem_avail_bytes`, `node_filesystem_size_bytes` | Node filesystem capacity |

---

## Network Traffic Metrics

### Network traffic by namespace (presets)

```
metrics_read  command: "preset"  flags: {name: "namespace_network_rx"}
metrics_read  command: "preset"  flags: {name: "namespace_network_tx"}
metrics_read  command: "preset"  flags: {name: "namespace_network_errors"}
```

### Network traffic by namespace (ad-hoc)

```
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(sum by (namespace)(rate(container_network_receive_bytes_total[5m]))))"}
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(sum by (namespace)(rate(container_network_transmit_bytes_total[5m]))))"}
```

### Network traffic by pod in a namespace

Replace `TARGET_NAMESPACE` with the namespace to inspect:

```
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(sum by (pod)(rate(container_network_receive_bytes_total{namespace=\"TARGET_NAMESPACE\"}[5m]))))"}
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(sum by (pod)(rate(container_network_transmit_bytes_total{namespace=\"TARGET_NAMESPACE\"}[5m]))))"}
```

### Network errors and drops by namespace

```
metrics_read  command: "query"  flags: {query: "topk(10, sum by (namespace)(rate(container_network_receive_errors_total[5m])) + sum by (namespace)(rate(container_network_transmit_errors_total[5m])))"}
```

### Node-level network throughput

```
metrics_read  command: "query"  flags: {query: "instance:node_network_receive_bytes_excluding_lo:rate1m + instance:node_network_transmit_bytes_excluding_lo:rate1m"}
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

### Pod status and restarts (presets)

```
metrics_read  command: "preset"  flags: {name: "cluster_pod_status"}
metrics_read  command: "preset"  flags: {name: "pod_restarts_top10"}
```

### CPU and memory by namespace (presets)

```
metrics_read  command: "preset"  flags: {name: "namespace_cpu_usage"}
metrics_read  command: "preset"  flags: {name: "namespace_memory_usage"}
```

### Pod count by namespace (ad-hoc)

```
metrics_read  command: "query"  flags: {query: "topk(15, count by (namespace)(kube_pod_info))"}
```

### Pod phase summary

```
metrics_read  command: "query"  flags: {query: "count by (phase)(kube_pod_status_phase == 1)"}
```

### Container restart counts (instability indicator)

```
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(kube_pod_container_status_restarts_total))"}
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

## KubeVirt VM Metrics

Metrics exposed by KubeVirt for each running VMI. Use `name` and `namespace` labels to target a specific VM.

### Top VMs by resource usage (presets)

```
metrics_read  command: "preset"  flags: {name: "vm_cpu_usage"}
metrics_read  command: "preset"  flags: {name: "vm_memory_usage"}
metrics_read  command: "preset"  flags: {name: "vm_network_rx"}
metrics_read  command: "preset"  flags: {name: "vm_network_tx"}
metrics_read  command: "preset"  flags: {name: "vm_storage_read"}
metrics_read  command: "preset"  flags: {name: "vm_storage_write"}
metrics_read  command: "preset"  flags: {name: "vm_storage_iops"}
```

### Per-VM queries (ad-hoc)

Replace `VM_NAME` and `VM_NAMESPACE` with actual values:

```
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_cpu_usage_seconds_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
metrics_read  command: "query"  flags: {query: "kubevirt_vmi_memory_resident_bytes{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_network_receive_bytes_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_network_transmit_bytes_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_storage_read_traffic_bytes_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_storage_write_traffic_bytes_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
```

### VM disk IOPS and latency

```
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_storage_iops_read_total{name=\"VM_NAME\"}[5m])"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_storage_read_times_seconds_total{name=\"VM_NAME\"}[5m]) / rate(kubevirt_vmi_storage_iops_read_total{name=\"VM_NAME\"}[5m])"}
```

### Available labels on kubevirt_vmi_* metrics

| Label | Description | Example values |
|-------|-------------|----------------|
| `name` | VM name | `my-rhel-vm`, `webserver-01` |
| `namespace` | VM namespace | `default`, `my-vms` |
| `node` | Node running the VMI | `worker-0`, `worker-1` |
| `pod` | Virt-launcher pod name | `virt-launcher-my-vm-abcde` |
| `owner` | Owner reference | `VirtualMachine/my-vm` |
| `interface` | Virtual NIC name (network metrics) | `default`, `net1` |
| `drive` | Virtual disk name (storage metrics) | `vda`, `vdb` |
| `id` | vCPU index (vcpu_seconds_total) | `0`, `1`, `2` |
| `state` | vCPU state (vcpu_seconds_total) | `running`, `halted` |

### KubeVirt VM metrics reference

| Metric | Description |
|--------|-------------|
| `kubevirt_vmi_cpu_usage_seconds_total` | Total CPU time consumed (counter) |
| `kubevirt_vmi_cpu_system_usage_seconds_total` | System CPU time (counter) |
| `kubevirt_vmi_cpu_user_usage_seconds_total` | User CPU time (counter) |
| `kubevirt_vmi_vcpu_seconds_total` | Per-vCPU time by state (counter) |
| `kubevirt_vmi_memory_resident_bytes` | Resident memory (gauge) |
| `kubevirt_vmi_memory_available_bytes` | Available memory (gauge) |
| `kubevirt_vmi_memory_used_bytes` | Used memory (gauge) |
| `kubevirt_vmi_memory_domain_bytes` | Total domain memory (gauge) |
| `kubevirt_vmi_network_receive_bytes_total` | Bytes received per interface (counter) |
| `kubevirt_vmi_network_transmit_bytes_total` | Bytes transmitted per interface (counter) |
| `kubevirt_vmi_network_receive_errors_total` | Receive errors (counter) |
| `kubevirt_vmi_network_transmit_errors_total` | Transmit errors (counter) |
| `kubevirt_vmi_storage_read_traffic_bytes_total` | Bytes read per drive (counter) |
| `kubevirt_vmi_storage_write_traffic_bytes_total` | Bytes written per drive (counter) |
| `kubevirt_vmi_storage_iops_read_total` | Read IOPS per drive (counter) |
| `kubevirt_vmi_storage_iops_write_total` | Write IOPS per drive (counter) |
| `kubevirt_vmi_storage_read_times_seconds_total` | Read latency per drive (counter) |
| `kubevirt_vmi_storage_write_times_seconds_total` | Write latency per drive (counter) |
| `kubevirt_vmi_info` | VMI metadata (labels: phase, os, flavor, workload) |
| `kubevirt_vmi_phase_count` | Count of VMIs by phase |

---

## Forklift / MTV Migration Metrics

### Migration status and plans (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_migration_status"}
metrics_read  command: "preset"  flags: {name: "mtv_plan_status"}
```

### Migration data transfer and throughput (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_data_transferred"}
metrics_read  command: "preset"  flags: {name: "mtv_net_throughput"}
metrics_read  command: "preset"  flags: {name: "mtv_storage_throughput"}
```

### Migration duration (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_migration_duration"}
metrics_read  command: "preset"  flags: {name: "mtv_avg_migration_duration"}
```

### Migration pod CPU and network traffic (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_populator_cpu"}
metrics_read  command: "preset"  flags: {name: "mtv_migration_pod_rx"}
metrics_read  command: "preset"  flags: {name: "mtv_migration_pod_tx"}
metrics_read  command: "preset"  flags: {name: "mtv_forklift_traffic"}
```

### KubeVirt VMI migration metrics (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_vmi_migrations_pending"}
metrics_read  command: "preset"  flags: {name: "mtv_vmi_migrations_running"}
```

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

### Narrowing migration metrics with label filters

Use PromQL label selectors or the `selector` flag to narrow results:

```
metrics_read  command: "query"  flags: {query: "mtv_migration_data_transferred_bytes{provider=\"vsphere\"}"}
metrics_read  command: "query"  flags: {query: "mtv_migration_data_transferred_bytes{mode=\"Cold\"}"}
metrics_read  command: "query"  flags: {query: "mtv_migration_data_transferred_bytes{provider=\"ovirt\", mode=\"Warm\"}"}
metrics_read  command: "query"  flags: {query: "mtv_migrations_status_total{status=\"Failed\"}"}
metrics_read  command: "query"  flags: {query: "mtv_workload_migrations_status_total{plan=\"PLAN_UUID\", status=\"Failed\"}"}
```

### Grouping migration metrics

```
metrics_read  command: "query"  flags: {query: "sum by (provider)(mtv_migration_data_transferred_bytes)"}
metrics_read  command: "query"  flags: {query: "sum by (mode)(mtv_migration_data_transferred_bytes)"}
metrics_read  command: "query"  flags: {query: "sum by (status, provider)(mtv_migrations_status_total)"}
metrics_read  command: "query"  flags: {query: "avg by (provider)(mtv_migration_duration_seconds)"}
metrics_read  command: "query"  flags: {query: "sum by (plan, status)(mtv_workload_migrations_status_total)"}
```

---

## Quick Health Dashboard

Run these presets together for a cluster overview:

```
metrics_read  command: "preset"  flags: {name: "cluster_cpu_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_memory_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_node_readiness"}
metrics_read  command: "preset"  flags: {name: "cluster_pod_status"}
metrics_read  command: "preset"  flags: {name: "pod_restarts_top10"}
metrics_read  command: "query"   flags: {query: "ceph_health_status"}
metrics_read  command: "preset"  flags: {name: "namespace_network_rx"}
metrics_read  command: "preset"  flags: {name: "mtv_migration_status"}
```
