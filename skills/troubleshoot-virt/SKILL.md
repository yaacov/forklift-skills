---
name: troubleshoot-virt
description: Troubleshoot stuck VMs and migrations in OpenShift Virtualization and MTV/Forklift. Use when VMs won't start, DataVolumes are stuck, migrations fail, or cluster resources are exhausted.
---

# Troubleshooting VMs and Migrations

Use this guide when VMs or migrations are stuck, failing, or behaving unexpectedly.

## Quick Triage Checklist

When something is stuck, check these in order:

1. **Node resources** -- is the cluster out of CPU/memory/pods?
2. **Storage** -- is the default StorageClass set? Are PVCs bound? Are DataVolumes progressing?
3. **VM status** -- what does the VM/VMI conditions say?
4. **Pod status** -- is the virt-launcher or importer pod stuck/erroring?
5. **Events** -- what do namespace events say?

## 1. Node Resources

```bash
# Allocatable resources per node
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.conditions[-1].type,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory,PODS:.status.allocatable.pods'

# Actual resource usage (requires metrics-server)
kubectl top nodes

# Check what's consuming resources on a specific node
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name> \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory'

# Check for node conditions (pressure, taints)
kubectl describe node <node-name> | grep -A5 -E 'Conditions:|Taints:'
```

If nodes show `MemoryPressure` or `DiskPressure`, VMs and migration pods cannot be scheduled.

## 2. Storage

### Default StorageClass

A default StorageClass is required for DataVolumes to work. Without it, PVCs won't provision.

```bash
# Check default StorageClass (look for "(default)" annotation)
kubectl get storageclass

# If none is default, set one:
kubectl annotate storageclass <name> storageclass.kubernetes.io/is-default-class=true
```

### StorageProfile

CDI uses StorageProfiles to determine accessModes and volumeMode for each StorageClass.
A misconfigured profile can cause DataVolumes to fail.

```bash
# List all storage profiles
kubectl get storageprofile

# Check a specific profile (look for claimPropertySets, cloneStrategy)
kubectl get storageprofile <storageclass-name> -o yaml
```

A healthy StorageProfile has `status.claimPropertySets` populated with accessModes and volumeMode.

### DataVolumes (DV)

DataVolumes manage the lifecycle of importing/cloning disk images into PVCs.

```bash
# List DataVolumes and their status
kubectl get dv -n <namespace>

# Check a stuck DataVolume
kubectl describe dv <dv-name> -n <namespace>

# Common DV phases:
#   ImportScheduled -> ImportInProgress -> Succeeded
#   CloneScheduled -> CloneInProgress -> SnapshotForSmartCloneInProgress -> Succeeded
#   Pending (stuck) -- usually a storage or scheduling problem
```

### PVCs

```bash
# Check PVC status (should be Bound)
kubectl get pvc -n <namespace>

# Stuck in Pending = no StorageClass, no capacity, or WaitForFirstConsumer binding
kubectl describe pvc <pvc-name> -n <namespace>
```

### CDI Importer/Cloner Pods

When a DataVolume is importing, CDI creates temporary pods. If those pods are stuck,
the DV won't progress.

```bash
# Find CDI importer/cloner pods in a namespace
kubectl get pods -n <namespace> | grep -E 'importer|clone|upload'

# Check pod status and events
kubectl describe pod <importer-pod> -n <namespace>

# Get logs from the importer
kubectl logs <importer-pod> -n <namespace>
```

## 3. VM Status

```bash
# VM status overview
kubectl get vm <vm-name> -n <namespace>

# VMI (running instance) status
kubectl get vmi <vm-name> -n <namespace>

# Detailed conditions (scheduling, volumes, readiness)
kubectl describe vm <vm-name> -n <namespace>
kubectl describe vmi <vm-name> -n <namespace>

# Common stuck reasons:
#   - Unschedulable: not enough CPU/memory on any node
#   - DataVolumeError: boot disk DV failed
#   - ErrImagePull: containerdisk image not found
#   - Guest agent not connected: VM running but no agent
```

## 4. Pod Status (virt-launcher)

Each running VM has a `virt-launcher` pod. If the pod is stuck, the VM won't start.

```bash
# Find the virt-launcher pod for a VM
kubectl get pods -n <namespace> -l kubevirt.io=virt-launcher

# Or find it by VM name
kubectl get pods -n <namespace> | grep virt-launcher

# Check pod status, events, and conditions
kubectl describe pod <virt-launcher-pod> -n <namespace>

# Get virt-launcher logs
kubectl logs <virt-launcher-pod> -n <namespace>

# If the pod has multiple containers, check each:
kubectl logs <virt-launcher-pod> -n <namespace> -c compute
kubectl logs <virt-launcher-pod> -n <namespace> -c guest-console-log
```

## 5. Events

Namespace events often reveal the root cause faster than anything else.

```bash
# Recent events in the namespace (sorted by time)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Filter for warnings only
kubectl get events -n <namespace> --field-selector type=Warning

# Events for a specific VM
kubectl get events -n <namespace> --field-selector involvedObject.name=<vm-name>
```

## 6. Migration Troubleshooting (MTV/Forklift)

### Quick health check

```bash
# MTV system health (checks operator, pods, providers, plans)
kubectl mtv health --all-namespaces

# Structured logs from the forklift controller
kubectl mtv health logs -n <forklift-namespace>
kubectl mtv health logs -n <forklift-namespace> --filter-plan <plan-name> --filter-level error
```

### Forklift pods

Forklift runs in its own namespace (commonly `openshift-mtv` or `konveyor-forklift`).

```bash
# Find the forklift namespace
kubectl get pods --all-namespaces -l app=forklift --no-headers | head -1 | awk '{print $1}'

# List forklift pods
kubectl get pods -n <forklift-namespace>

# Key pods to check:
#   forklift-controller   - main migration controller (2 containers: controller + inventory)
#   forklift-api          - API server
#   forklift-validation   - VM validation service
#   forklift-volume-populator-controller - manages volume population during migration

# Controller logs (migration reconciliation)
kubectl logs -n <forklift-namespace> deployment/forklift-controller -c main
kubectl logs -n <forklift-namespace> deployment/forklift-controller -c inventory
```

### Migration plan status

```bash
# Plan overview
kubectl mtv get plan -n <namespace>
kubectl mtv get plan --name <plan-name> -n <namespace>

# VM-level status within a plan
kubectl mtv get plan --name <plan-name> --vms -n <namespace>

# Disk transfer progress
kubectl mtv get plan --name <plan-name> --disk -n <namespace>

# Detailed plan description
kubectl mtv describe plan --name <plan-name> -n <namespace>
```

### Migration pods (per-VM)

During migration, Forklift creates pods for each VM being migrated. These are in the
target namespace (where VMs are being created), not the forklift operator namespace.

```bash
# Find migration-related pods in the target namespace
kubectl get pods -n <namespace> | grep -E 'virt-v2v|populator|importer'

# Check virt-v2v converter pod (does the actual disk conversion)
kubectl logs <virt-v2v-pod> -n <namespace>

# Check populator pods (populate volumes)
kubectl get pods -n <namespace> -l app=forklift-populator
```

### Provider connectivity

```bash
# Check provider status
kubectl mtv get provider -n <namespace>

# Describe for connection errors
kubectl mtv describe provider --name <provider-name> -n <namespace>
```

## 7. KubeVirt Operator Pods

The KubeVirt operator components run in `openshift-cnv` (OpenShift) or `kubevirt` namespace.

```bash
# Key operator pods
kubectl get pods -n openshift-cnv | grep -E 'virt-operator|virt-controller|virt-handler|virt-api|cdi-'

# virt-controller: manages VM lifecycle, scheduling
# virt-handler: per-node agent, manages virt-launcher pods
# virt-api: API server
# cdi-deployment: CDI controller for DataVolumes

# Check for pod restarts (sign of instability)
kubectl get pods -n openshift-cnv -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount' \
  | sort -t' ' -k3 -rn | head -10

# Logs from key components
kubectl logs -n openshift-cnv deployment/virt-controller
kubectl logs -n openshift-cnv deployment/cdi-deployment
```

## 8. Common Stuck Scenarios

### VM stuck in Scheduling
- **Cause**: Not enough CPU/memory on any schedulable node
- **Check**: `kubectl get nodes` resource columns, `kubectl describe vmi <vm>` for scheduling errors
- **Fix**: Free up node resources, scale cluster, or use a smaller instance type

### DataVolume stuck in Pending
- **Cause**: No default StorageClass, or StorageProfile misconfigured
- **Check**: `kubectl get storageclass` (look for default), `kubectl get storageprofile <sc> -o yaml`
- **Fix**: Set a default StorageClass, ensure StorageProfile has `claimPropertySets`

### DataVolume stuck in ImportInProgress
- **Cause**: Importer pod failing (network, auth, image not found)
- **Check**: `kubectl get pods -n <ns> | grep importer`, then `kubectl logs <importer-pod>`
- **Fix**: Check source URL, credentials, network policies

### Migration plan stuck
- **Cause**: Provider unreachable, disk transfer stalled, converter pod OOM
- **Check**: `kubectl mtv health`, `kubectl mtv get plan --name <plan> --vms --disk`, converter pod logs
- **Fix**: Check provider connectivity, increase converter memory via settings, check storage throughput

### VM stuck in Pending after migration
- **Cause**: Target PVCs not bound, insufficient resources for target VM
- **Check**: `kubectl get pvc -n <ns>`, `kubectl describe vmi <vm-name>`
- **Fix**: Ensure target storage has capacity, check node resources
