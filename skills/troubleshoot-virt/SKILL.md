---
name: troubleshoot-virt
description: Troubleshoot stuck VMs and migrations in OpenShift Virtualization and MTV/Forklift. Use when VMs won't start, DataVolumes are stuck, migrations fail, or cluster resources are exhausted.
---

# Troubleshooting VMs and Migrations

Use this guide when VMs or migrations are stuck, failing, or behaving unexpectedly.

## Required MCP Servers

This skill requires:
- `debug_read` (from the kubectl-debug-queries MCP server) -- for listing resources, logs, events
- `mtv_read` (from the kubectl-mtv MCP server) -- for MTV health, plans, providers
- `metrics_read` (from the kubectl-metrics MCP server) -- for node resource usage

If any of these tools are not available in your environment, inform the user and refer them to the `mcp-setup` skill for installation instructions. Do not attempt bash fallback.

## Quick Triage Checklist

When something is stuck, check these in order:

1. **Node resources** -- is the cluster out of CPU/memory/pods?
2. **Storage** -- is the default StorageClass set? Are PVCs bound? Are DataVolumes progressing?
3. **VM status** -- what does the VM/VMI conditions say?
4. **Pod status** -- is the virt-launcher or importer pod stuck/erroring?
5. **Events** -- what do namespace events say?

## 1. Node Resources

```
debug_read  command: "list"  flags: {resource: "nodes"}

metrics_read  command: "preset"  flags: {name: "cluster_cpu_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_memory_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_node_readiness"}
```

Check what's consuming resources on a specific node:

```
debug_read  command: "list"  flags: {resource: "pods", all_namespaces: true, query: "where spec.nodeName = '<node-name>'"}
```

Check for node conditions:

```
debug_read  command: "get"  flags: {resource: "node", name: "<node-name>"}
```

If nodes show `MemoryPressure` or `DiskPressure`, VMs and migration pods cannot be scheduled.

## 2. Storage

### Default StorageClass

A default StorageClass is required for DataVolumes to work. Without it, PVCs won't provision.

```
debug_read  command: "list"  flags: {resource: "storageclass"}
```

If none is default, set one (requires shell):

```bash
kubectl annotate storageclass <name> storageclass.kubernetes.io/is-default-class=true
```

### StorageProfile

CDI uses StorageProfiles to determine accessModes and volumeMode for each StorageClass. A misconfigured profile can cause DataVolumes to fail.

```
debug_read  command: "list"  flags: {resource: "storageprofile"}
debug_read  command: "get"   flags: {resource: "storageprofile", name: "<storageclass-name>", output: "yaml"}
```

A healthy StorageProfile has `status.claimPropertySets` populated with accessModes and volumeMode.

### DataVolumes (DV)

DataVolumes manage the lifecycle of importing/cloning disk images into PVCs.

```
debug_read  command: "list"  flags: {resource: "dv", namespace: "<namespace>"}
debug_read  command: "get"   flags: {resource: "dv", name: "<dv-name>", namespace: "<namespace>"}
```

Common DV phases: `ImportScheduled` -> `ImportInProgress` -> `Succeeded`. `Pending` (stuck) usually means a storage or scheduling problem.

### PVCs

```
debug_read  command: "list"  flags: {resource: "pvc", namespace: "<namespace>"}
debug_read  command: "get"   flags: {resource: "pvc", name: "<pvc-name>", namespace: "<namespace>"}
```

Stuck in Pending = no StorageClass, no capacity, or WaitForFirstConsumer binding.

### CDI Importer/Cloner Pods

When a DataVolume is importing, CDI creates temporary pods. If those pods are stuck, the DV won't progress.

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "<namespace>", query: "where name ~= '.*importer.*|.*clone.*|.*upload.*'"}
debug_read  command: "get"   flags: {resource: "pod", name: "<importer-pod>", namespace: "<namespace>"}
debug_read  command: "logs"  flags: {name: "<importer-pod>", namespace: "<namespace>"}
```

## 3. VM Status

```
debug_read  command: "get"  flags: {resource: "vm", name: "<vm-name>", namespace: "<namespace>"}
debug_read  command: "get"  flags: {resource: "vmi", name: "<vm-name>", namespace: "<namespace>"}
```

Common stuck reasons:
- Unschedulable: not enough CPU/memory on any node
- DataVolumeError: boot disk DV failed
- ErrImagePull: containerdisk image not found
- Guest agent not connected: VM running but no agent

## 4. Pod Status (virt-launcher)

Each running VM has a `virt-launcher` pod. If the pod is stuck, the VM won't start.

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "<namespace>", selector: "kubevirt.io=virt-launcher"}
debug_read  command: "get"   flags: {resource: "pod", name: "<virt-launcher-pod>", namespace: "<namespace>"}
debug_read  command: "logs"  flags: {name: "<virt-launcher-pod>", namespace: "<namespace>"}
debug_read  command: "logs"  flags: {name: "<virt-launcher-pod>", namespace: "<namespace>", container: "compute"}
```

## 5. Events

Namespace events often reveal the root cause faster than anything else.

```
debug_read  command: "events"  flags: {namespace: "<namespace>", sort_by: "time_desc"}
debug_read  command: "events"  flags: {namespace: "<namespace>", query: "where type = 'Warning'"}
debug_read  command: "events"  flags: {namespace: "<namespace>", name: "<vm-name>", resource: "VirtualMachine"}
```

## 6. Migration Troubleshooting (MTV/Forklift)

### Quick health check

The `health` command includes built-in log analysis by default:

```
mtv_read  command: "health"  flags: {all_namespaces: true}
mtv_read  command: "health"  flags: {namespace: "<forklift-namespace>", log_lines: 200}
mtv_read  command: "health"  flags: {all_namespaces: true, skip_logs: true}
```

For targeted error logs from specific pods:

```
debug_read  command: "logs"  flags: {name: "deployment/forklift-controller", namespace: "<forklift-namespace>", container: "main", tail: 100, query: "where level = 'ERROR'"}
```

### Forklift pods

Forklift runs in its own namespace (commonly `openshift-mtv` or `konveyor-forklift`).

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "<forklift-namespace>"}
debug_read  command: "logs"  flags: {name: "deployment/forklift-controller", namespace: "<forklift-namespace>", container: "main"}
debug_read  command: "logs"  flags: {name: "deployment/forklift-controller", namespace: "<forklift-namespace>", container: "inventory"}
```

Key pods: `forklift-controller` (main migration controller), `forklift-api`, `forklift-validation`, `forklift-volume-populator-controller`.

### Migration plan status

```
mtv_read  command: "get plan"  flags: {namespace: "<namespace>"}
mtv_read  command: "get plan"  flags: {name: "<plan-name>", namespace: "<namespace>"}
mtv_read  command: "get plan"  flags: {name: "<plan-name>", vms: true, namespace: "<namespace>"}
mtv_read  command: "get plan"  flags: {name: "<plan-name>", disk: true, namespace: "<namespace>"}
mtv_read  command: "describe plan"  flags: {name: "<plan-name>", namespace: "<namespace>"}
```

### Migration pods (per-VM)

During migration, Forklift creates pods in the target namespace (not the operator namespace):

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "<namespace>", query: "where name ~= '.*virt-v2v.*|.*populator.*|.*importer.*'"}
debug_read  command: "logs"  flags: {name: "<virt-v2v-pod>", namespace: "<namespace>"}
```

### Provider connectivity

```
mtv_read  command: "get provider"  flags: {namespace: "<namespace>"}
mtv_read  command: "describe provider"  flags: {name: "<provider-name>", namespace: "<namespace>"}
```

## 7. KubeVirt Operator Pods

The KubeVirt operator components run in `openshift-cnv` (OpenShift) or `kubevirt` namespace.

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-cnv", query: "where name ~= '.*virt-operator.*|.*virt-controller.*|.*virt-handler.*|.*virt-api.*|.*cdi-.*'"}
```

Check for pod restarts (sign of instability):

```
metrics_read  command: "preset"  flags: {name: "pod_restarts_top10"}
```

Logs from key components:

```
debug_read  command: "logs"  flags: {name: "deployment/virt-controller", namespace: "openshift-cnv"}
debug_read  command: "logs"  flags: {name: "deployment/cdi-deployment", namespace: "openshift-cnv"}
```

## 8. Common Stuck Scenarios

### VM stuck in Scheduling
- **Cause**: Not enough CPU/memory on any schedulable node
- **Check**: `debug_read` list nodes + `metrics_read` preset `cluster_cpu_utilization` + `debug_read` get vmi for scheduling errors
- **Fix**: Free up node resources, scale cluster, or use a smaller instance type

### DataVolume stuck in Pending
- **Cause**: No default StorageClass, or StorageProfile misconfigured
- **Check**: `debug_read` list storageclass (look for default), `debug_read` get storageprofile
- **Fix**: Set a default StorageClass, ensure StorageProfile has `claimPropertySets`

### DataVolume stuck in ImportInProgress
- **Cause**: Importer pod failing (network, auth, image not found)
- **Check**: `debug_read` list pods with query for importer, then `debug_read` logs
- **Fix**: Check source URL, credentials, network policies

### Migration plan stuck
- **Cause**: Provider unreachable, disk transfer stalled, converter pod OOM
- **Check**: `mtv_read` health, `mtv_read` get plan with vms+disk flags, converter pod logs
- **Fix**: Check provider connectivity, increase converter memory via settings, check storage throughput

### VM stuck in Pending after migration
- **Cause**: Target PVCs not bound, insufficient resources for target VM
- **Check**: `debug_read` list pvc, `debug_read` get vmi
- **Fix**: Ensure target storage has capacity, check node resources

## Self-Learning Rule

When you need to discover available flags or verify syntax, call the MCP help tools:

```
mtv_help  command: "<command>"
debug_help  command: "logs"
metrics_help  command: "preset"
```
