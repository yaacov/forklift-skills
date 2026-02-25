---
name: kubectl-mtv
description: Use kubectl mtv (or oc mtv) to manage MTV/Forklift VM migrations. Use this skill when the user wants to migrate VMs from vSphere, oVirt, OpenStack, OVA, EC2, or HyperV to OpenShift/KubeVirt.
---

# kubectl mtv - Migration Toolkit for Virtualization CLI

`kubectl mtv` (or `oc mtv`) is a kubectl plugin for migrating VMs from VMware vSphere, oVirt (RHV), OpenStack, OVA, EC2, and HyperV to OpenShift Virtualization (KubeVirt).

## Getting Help

Always use the built-in help for discovering subcommands and flags:

```bash
kubectl mtv --help
kubectl mtv <command> --help
kubectl mtv <command> <subcommand> --help
```

## Typical Migration Workflow

### 1. Check system health

```bash
kubectl mtv health
kubectl mtv health --all-namespaces
kubectl mtv health --skip-logs
```

### 2. Configure settings (e.g., VDDK image for vSphere)

```bash
kubectl mtv settings set --setting vddk_image --value <registry-url>/vddk
kubectl mtv settings get --setting vddk_image
kubectl mtv settings unset --setting vddk_image
kubectl mtv settings --all
```

### 3. Create providers

For vSphere providers, the VDDK init image is required for migration. Before creating a
vSphere provider, check if the VDDK image is already configured globally:

```bash
kubectl mtv settings get --setting vddk_image
```

If a global VDDK image is set, you do NOT need `--vddk-init-image` on the provider.
If it is not set, prefer setting it globally (if you have permissions):

```bash
kubectl mtv settings set --setting vddk_image --value "quay.io/kubev2v/vddk:latest"
```

Only use `--vddk-init-image` on the provider as a fallback when you cannot set the global setting.

```bash
# OpenShift target (host cluster)
kubectl mtv create provider --name host --type openshift -n <namespace>

# vSphere source (VDDK image already set globally)
kubectl mtv create provider --name my-vsphere \
  --type vsphere \
  --url "https://vcenter.example.com/sdk" \
  --username "admin@vsphere.local" \
  --password "$PASSWORD" \
  -n <namespace>

# vSphere source (fallback: VDDK image not set globally and no permissions to set it)
kubectl mtv create provider --name my-vsphere \
  --type vsphere \
  --url "https://vcenter.example.com/sdk" \
  --username "admin@vsphere.local" \
  --password "$PASSWORD" \
  --vddk-init-image "quay.io/kubev2v/vddk:latest" \
  -n <namespace>

# oVirt source
kubectl mtv create provider --name my-ovirt \
  --type ovirt \
  --url "https://rhv-manager.example.com/ovirt-engine/api" \
  --username "admin@internal" \
  --password "$PASSWORD" \
  -n <namespace>

# EC2 source
kubectl mtv create provider --name my-ec2 \
  --type ec2 \
  --ec2-region us-east-1 \
  --username "$EC2_KEY" \
  --password "$EC2_SECRET" \
  --auto-target-credentials \
  -n <namespace>

# HyperV source
kubectl mtv create provider --name my-hyperv \
  --type hyperv \
  --url "https://192.168.1.100" \
  --username Administrator \
  --password "$PASSWORD" \
  --smb-url "//192.168.1.100/VMShare" \
  -n <namespace>

# Use --provider-insecure-skip-tls to skip TLS verification
```

### 4. List providers and verify

```bash
kubectl mtv get provider -n <namespace>
kubectl mtv get provider --all-namespaces
```

### 5. Browse inventory VMs

```bash
# List all VMs
kubectl mtv get inventory vm --provider my-vsphere -n <namespace>

# Extended details
kubectl mtv get inventory vm --provider my-vsphere --extended -n <namespace>

# Filter with TSL queries
kubectl mtv get inventory vm --provider my-vsphere -q "where name ~= 'prod-.*'" -n <namespace>
kubectl mtv get inventory vm --provider my-vsphere -q "where powerState = 'poweredOn' and memoryMB > 4096" -n <namespace>
kubectl mtv get inventory vm --provider my-vsphere -q "where cpuCount > 4 and len(disks) > 1" -n <namespace>

# Export for plan creation
kubectl mtv get inventory vm --provider my-vsphere -q "where name ~= 'web-.*'" --output planvms -n <namespace>
```

### 6. Create a migration plan

Prefer omitting `--network-pairs` and `--storage-pairs` to let MTV auto-map.
Use `--default-target-network` and `--default-target-storage-class` for simple defaults.
Only use explicit mapping pairs when you need specific source-to-target mappings.

```bash
# Simple plan (auto-mapped network and storage)
kubectl mtv create plan --name my-migration \
  --source my-vsphere \
  --target host \
  --vms "vm1,vm2,vm3" \
  --default-target-network default \
  --default-target-storage-class standard \
  -n <namespace>

# With a TSL query to select VMs
kubectl mtv create plan --name my-migration \
  --source my-vsphere \
  --target host \
  --vms "where name ~= 'prod-.*'" \
  --default-target-network default \
  --default-target-storage-class standard \
  -n <namespace>

# With explicit mapping pairs (only when specific mappings are needed)
kubectl mtv create plan --name my-migration \
  --source my-vsphere \
  --target host \
  --vms "vm1,vm2,vm3" \
  --network-pairs "VM Network:default,Production:myns/br-ext" \
  --storage-pairs "datastore1:fast-ssd,datastore2:economy" \
  -n <namespace>

# Warm migration
kubectl mtv create plan --name my-warm \
  --source my-vsphere \
  --target host \
  --vms "critical-vm" \
  --migration-type warm \
  --default-target-network default \
  --default-target-storage-class standard \
  -n <namespace>
```

### 7. Start migration

```bash
kubectl mtv start plan --name my-migration -n <namespace>

# Start multiple
kubectl mtv start plan --name plan1,plan2 -n <namespace>

# Dry run (preview)
kubectl mtv start plan --name my-migration --dry-run -n <namespace>
```

### 8. Monitor migration

```bash
# Plan status
kubectl mtv get plan -n <namespace>
kubectl mtv get plan --name my-migration -n <namespace>

# VM-level status
kubectl mtv get plan --name my-migration --vms -n <namespace>

# Disk transfer progress
kubectl mtv get plan --name my-migration --disk -n <namespace>

# Both VM and disk
kubectl mtv get plan --name my-migration --vms --disk -n <namespace>

# VMs table across plans
kubectl mtv get plan --vms-table -n <namespace>
```

### 9. View logs

```bash
kubectl mtv health logs -n openshift-mtv
kubectl mtv health logs -n openshift-mtv --filter-plan my-migration
kubectl mtv health logs -n openshift-mtv --filter-plan my-migration --filter-level error
kubectl mtv health logs -n openshift-mtv --source inventory --filter-provider my-vsphere
```

### 10. Cleanup

```bash
kubectl mtv delete plan --name my-migration -n <namespace>
kubectl mtv delete provider --name my-vsphere -n <namespace>
```

## TSL Query Syntax (for --vms and -q flags)

TSL (Tree Search Language) filters VMs by their properties:

```
where <condition> [order by <field> [asc|desc]] [limit N]
```

### Operators

- Comparison: `=`, `!=`, `<`, `<=`, `>`, `>=`
- String: `like` (% wildcard), `ilike` (case-insensitive), `~=` (regex), `~!` (regex negation)
- Logical: `and`, `or`, `not`
- Set: `in [...]`, `not in [...]`, `between X and Y`
- Array: `len(field)`, `any(field[*].sub = 'val')`, `all(field[*].sub >= N)`
- SI units: `4Gi`, `512Mi`, `1Ti`

### Common fields (vSphere)

- `name`, `id`, `powerState`, `cpuCount`, `memoryMB`, `guestId`, `firmware`
- `len(disks)`, `len(nics)`, `disks[*].capacity`, `disks[*].shared`
- `concerns[*].category` (Critical, Warning, Information)
- `path` (folder path), `host`, `storageUsed`

### Examples

```
where name ~= 'prod-.*'
where powerState = 'poweredOn' and memoryMB > 4096
where cpuCount > 4 and len(disks) > 1
where any(concerns[*].category = 'Critical')
where name like '%web%' order by memoryMB desc limit 10
```

## Other Resources

```bash
# Network/storage mappings
kubectl mtv get mapping network -n <namespace>
kubectl mtv get mapping storage -n <namespace>
kubectl mtv create mapping network --name my-net --source my-vsphere --target host --network-pairs "VM Network:default" -n <namespace>
kubectl mtv create mapping storage --name my-store --source my-vsphere --target host --storage-pairs "datastore1:standard" -n <namespace>

# Hooks
kubectl mtv get hook -n <namespace>

# Hosts
kubectl mtv get host -n <namespace>

# Describe resources
kubectl mtv describe plan --name my-migration -n <namespace>
kubectl mtv describe mapping network --name my-net -n <namespace>
```

## Self-Learning Rule

When you encounter an unfamiliar `kubectl mtv` subcommand or need to verify flags, always run:

```bash
kubectl mtv <command> --help
kubectl mtv <command> <subcommand> --help
```

This ensures you use the correct and current syntax.
