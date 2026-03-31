# Forklift / MTV Migration Metrics

Queries, labels, and metrics for Forklift/MTV VM migrations. See [SKILL.md](SKILL.md) for general usage.

## Migration status and plans (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_migration_status"}
metrics_read  command: "preset"  flags: {name: "mtv_plan_status"}
```

## Migration data transfer and throughput (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_data_transferred"}
metrics_read  command: "preset"  flags: {name: "mtv_net_throughput"}
metrics_read  command: "preset"  flags: {name: "mtv_storage_throughput"}
```

## Migration duration (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_migration_duration"}
metrics_read  command: "preset"  flags: {name: "mtv_avg_migration_duration"}
```

## Migration pod CPU and network traffic (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_populator_cpu"}
metrics_read  command: "preset"  flags: {name: "mtv_migration_pod_rx"}
metrics_read  command: "preset"  flags: {name: "mtv_migration_pod_tx"}
metrics_read  command: "preset"  flags: {name: "mtv_forklift_traffic"}
```

## KubeVirt VMI migration metrics (presets)

```
metrics_read  command: "preset"  flags: {name: "mtv_vmi_migrations_pending"}
metrics_read  command: "preset"  flags: {name: "mtv_vmi_migrations_running"}
```

## Available labels on mtv_* metrics

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

## Metrics reference

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

## Narrowing migration metrics with label filters

Use PromQL label selectors or the `selector` flag to narrow results:

```
metrics_read  command: "query"  flags: {query: "mtv_migration_data_transferred_bytes{provider=\"vsphere\"}"}
metrics_read  command: "query"  flags: {query: "mtv_migration_data_transferred_bytes{mode=\"Cold\"}"}
metrics_read  command: "query"  flags: {query: "mtv_migration_data_transferred_bytes{provider=\"ovirt\", mode=\"Warm\"}"}
metrics_read  command: "query"  flags: {query: "mtv_migrations_status_total{status=\"Failed\"}"}
metrics_read  command: "query"  flags: {query: "mtv_workload_migrations_status_total{plan=\"PLAN_UUID\", status=\"Failed\"}"}
```

## Grouping migration metrics

```
metrics_read  command: "query"  flags: {query: "sum by (provider)(mtv_migration_data_transferred_bytes)"}
metrics_read  command: "query"  flags: {query: "sum by (mode)(mtv_migration_data_transferred_bytes)"}
metrics_read  command: "query"  flags: {query: "sum by (status, provider)(mtv_migrations_status_total)"}
metrics_read  command: "query"  flags: {query: "avg by (provider)(mtv_migration_duration_seconds)"}
metrics_read  command: "query"  flags: {query: "sum by (plan, status)(mtv_workload_migrations_status_total)"}
```
