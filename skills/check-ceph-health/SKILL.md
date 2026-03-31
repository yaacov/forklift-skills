---
name: check-ceph-health
description: Check Ceph storage health on OpenShift OCS/ODF clusters. Use when PVCs are stuck in Pending, storage provisioning fails, Ceph is degraded, OSDs are full, or cluster storage needs diagnosis.
---

# Check Ceph Health

Use this guide to diagnose and remediate Ceph storage issues on OpenShift clusters running OCS/ODF (OpenShift Data Foundation).

## Required MCP Servers

This skill requires:
- `debug_read` (from the kubectl-debug-queries MCP server) -- for listing resources, logs, events
- `metrics_read` (from the kubectl-metrics MCP server) -- for Ceph metrics (health, capacity, OSD, PG)

If any of these tools are not available in your environment, inform the user and refer them to the `mcp-setup` skill for installation instructions. Do not attempt bash fallback.

## 1. Ceph Cluster Health

### Quick health status via metrics

```
metrics_read  command: "query"  flags: {query: "ceph_health_status"}
```

Health values: 0=OK, 1=WARN, 2=ERR.

### Capacity overview

```
metrics_read  command: "query"  flags: {query: "ceph_cluster_total_bytes"}
metrics_read  command: "query"  flags: {query: "ceph_cluster_total_used_bytes"}
metrics_read  command: "query"  flags: {query: "ceph_cluster_total_used_bytes / ceph_cluster_total_bytes * 100"}
```

### CephCluster CR status

```
debug_read  command: "get"  flags: {resource: "cephcluster", namespace: "openshift-storage", output: "json"}
```

Health states:
- `HEALTH_OK` -- cluster is healthy
- `HEALTH_WARN` -- degraded but functional (backfillfull, nearfull, degraded PGs)
- `HEALTH_ERR` -- critical, writes may be blocked (full OSDs, too few OSDs, down PGs)

## 2. OSD Status

### OSD metrics

```
metrics_read  command: "query"  flags: {query: "ceph_osd_stat_bytes"}
metrics_read  command: "query"  flags: {query: "ceph_osd_stat_bytes_used"}
metrics_read  command: "query"  flags: {query: "rate(ceph_osd_op_latency_sum[5m]) / rate(ceph_osd_op_latency_count[5m])"}
```

### OSD pods

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-storage", selector: "app=rook-ceph-osd"}
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-storage", query: "where name ~= '.*osd-prepare.*'"}
```

### OSD backing PVCs

```
debug_read  command: "list"  flags: {resource: "pvc", namespace: "openshift-storage", selector: "app=rook-ceph-osd"}
```

## 3. Placement Group Health

```
metrics_read  command: "query"  flags: {query: "ceph_pg_total"}
metrics_read  command: "query"  flags: {query: "ceph_pg_active"}
metrics_read  command: "query"  flags: {query: "ceph_pg_degraded"}
```

## 4. Pool Statistics

```
metrics_read  command: "query"  flags: {query: "ceph_pool_percent_used * 100"}
metrics_read  command: "query"  flags: {query: "rate(ceph_pool_rd[5m])"}
metrics_read  command: "query"  flags: {query: "rate(ceph_pool_wr[5m])"}
metrics_read  command: "query"  flags: {query: "ceph_pool_stored"}
metrics_read  command: "query"  flags: {query: "ceph_pool_max_avail"}
```

## 5. CSI Provisioner Pods

PVC provisioning is handled by CSI driver pods. If these are unhealthy, no volumes can be created.

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-storage", query: "where name ~= '.*rbd.*ctrlplugin.*'"}
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-storage", query: "where name ~= '.*cephfs.*ctrlplugin.*'"}
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-storage", query: "where name ~= '.*rbd.*nodeplugin.*'"}
```

Check CSI provisioner logs:

```
debug_read  command: "logs"  flags: {name: "<rbd-ctrlplugin-pod>", namespace: "openshift-storage", container: "csi-rbdplugin", tail: 50}
```

## 6. PVC and PV Diagnosis

```
debug_read  command: "list"  flags: {resource: "pvc", all_namespaces: true, query: "where status.phase = 'Pending'"}
debug_read  command: "get"   flags: {resource: "pvc", name: "<pvc-name>", namespace: "<namespace>"}
debug_read  command: "list"  flags: {resource: "pv", query: "where status.phase = 'Released'"}
debug_read  command: "list"  flags: {resource: "storageclass"}
```

## 7. Common Problems and Remediation

### OSDs Full (HEALTH_ERR: full osd(s))

**Symptoms**: PVCs stuck in Pending, provisioning errors with `DeadlineExceeded` or `operation already exists`.

**Diagnosis**:

```
metrics_read  command: "query"  flags: {query: "ceph_health_status"}
metrics_read  command: "query"  flags: {query: "ceph_osd_stat_bytes_used / ceph_osd_stat_bytes * 100"}
debug_read   command: "get"    flags: {resource: "cephcluster", namespace: "openshift-storage", output: "json"}
```

Look for `OSD_FULL` and `POOL_FULL` messages in the CephCluster status.

**Remediation**: See "Requires Shell" section below for `kubectl delete pv` and `ceph osd set-full-ratio`.

### OSDs Nearfull / Backfillfull (HEALTH_WARN)

**Symptoms**: Cluster functional but approaching full. Warnings about `nearfull` or `backfillfull` OSDs.

**Remediation**:
- Clean up unused PVCs and Released PVs
- Delete completed migration data no longer needed
- Plan capacity expansion before reaching full threshold (85%)

### Degraded PGs

**Symptoms**: `HEALTH_WARN` with messages about degraded or undersized placement groups.

**Diagnosis**:

```
metrics_read  command: "query"  flags: {query: "ceph_pg_degraded"}
metrics_read  command: "query"  flags: {query: "ceph_pg_total - ceph_pg_active"}
debug_read   command: "events"  flags: {namespace: "openshift-storage", query: "where type = 'Warning'"}
```

**Remediation**:
- If an OSD is down, check the OSD pod and its node
- If a node is down, Ceph will self-heal once the node returns
- If an OSD is permanently lost, Ceph will rebalance automatically (may take time)

### CSI Provisioner Not Responding

**Symptoms**: PVC events say "waiting for external provisioner" but no `ProvisioningFailed` errors.

**Diagnosis**:

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-storage", query: "where name ~= '.*ctrlplugin.*'"}
debug_read  command: "logs"  flags: {name: "<rbd-ctrlplugin-pod>", namespace: "openshift-storage", container: "csi-rbdplugin", tail: 100}
```

**Remediation**:
- Restart the CSI controller pod if it's stuck
- Check if the Ceph cluster is reachable from the CSI pod
- Verify the StorageClass references a valid pool and secret

### Pools Full but OSDs Not Full

**Symptoms**: `POOL_FULL` warning but individual OSDs have space.

**Diagnosis**:

```
metrics_read  command: "query"  flags: {query: "ceph_pool_percent_used * 100"}
metrics_read  command: "query"  flags: {query: "ceph_osd_stat_bytes_used / ceph_osd_stat_bytes * 100"}
```

**Remediation**:
- A pool may have a quota set -- check and raise it
- Rebalance may be needed if data is unevenly distributed

## 8. Operator Health

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-storage", query: "where name ~= '.*ocs-operator.*|.*odf-operator.*|.*rook-ceph-operator.*'"}
debug_read  command: "logs"  flags: {name: "deployment/rook-ceph-operator", namespace: "openshift-storage", tail: 50}
```

Check for high restart counts:

```
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(kube_pod_container_status_restarts_total))"}
```

## 9. Preventive Checks

Run these periodically to avoid surprise outages:

```
metrics_read  command: "query"  flags: {query: "ceph_cluster_total_used_bytes / ceph_cluster_total_bytes * 100"}
debug_read   command: "list"   flags: {resource: "pv", query: "where status.phase = 'Released'"}
debug_read   command: "list"   flags: {resource: "pvc", all_namespaces: true, query: "where status.phase = 'Pending'"}
```

Act when usage exceeds 70% -- start cleaning up or expanding capacity before hitting the 85% full threshold.

---

## Requires Shell

These remediation operations cannot be performed via MCP and require shell access:

### Delete Released PVs to reclaim space

```bash
kubectl get pv --field-selector status.phase=Released
kubectl delete pv <released-pv-names>
```

### Temporarily raise the full ratio (when Ceph is blocking all writes)

```bash
MON_POD=$(kubectl -n openshift-storage get pods -l app=rook-ceph-mon -o jsonpath='{.items[0].metadata.name}')
MON_ADDR=$(kubectl -n openshift-storage get pod $MON_POD -o jsonpath='{.spec.containers[0].env[?(@.name=="ROOK_CEPH_MON_HOST")].value}' | sed 's/\[//;s/\]//')

# Raise to 0.92 to unblock writes temporarily
kubectl -n openshift-storage exec $MON_POD -c mon -- \
  ceph -m $MON_ADDR --keyring /etc/ceph/keyring-store/keyring \
  osd set-full-ratio 0.92

# After space is freed, reset to default
kubectl -n openshift-storage exec $MON_POD -c mon -- \
  ceph -m $MON_ADDR --keyring /etc/ceph/keyring-store/keyring \
  osd set-full-ratio 0.85
```

## Self-Learning Rule

When you need to discover available flags, build custom Ceph queries, or verify syntax:

```
debug_help  command: "list"
debug_help  command: "logs"
metrics_help  command: "query"
metrics_help  command: "promql"
```
