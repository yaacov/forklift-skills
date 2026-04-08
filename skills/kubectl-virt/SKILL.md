---
name: kubectl-virt
description: Use kubectl virt (or oc virt) to manage KubeVirt virtual machines. Use this skill when the user wants to create, start, stop, or manage VMs on OpenShift/Kubernetes.
---

# kubectl virt - KubeVirt VM Management

`kubectl virt` (or `oc virt`) is a kubectl plugin for managing KubeVirt virtual machines.

## Installation

> Installation, PATH setup, and shell completion: [ref-install.md](ref-install.md)

## Getting Help

Always use the built-in help for discovering subcommands and flags:

```bash
kubectl virt --help
kubectl virt <command> --help
kubectl virt <command> <subcommand> --help
```

## Instance Types and Preferences

Prefer using instance types over raw `--memory`/CPU settings. Instance types define
pre-configured VM sizing (CPU, memory, and compute behavior), while preferences define
OS-specific settings (boot order, device models, firmware).

### Discovering available instance types

```bash
kubectl get virtualmachineclusterinstancetype
kubectl get virtualmachineclusterpreference
```

Series include **U** (universal/general purpose), **O** (overcommitted/high density),
**CX** (compute exclusive), **M** (memory intensive), **N** (network/DPDK),
**RT** (realtime), and **D** (dedicated CPU). Sizes range from `nano` (1 vCPU) to
`8xlarge` (32 vCPUs). Naming pattern: `<series><version>.<size>` (e.g., `u1.medium`).

Preferences configure OS-specific settings (e.g., `fedora`, `rhel.9`, `windows.11.virtio`).

> Full series table, sizing table, and preferences list: [ref-types.md](ref-types.md)

### How to choose

- **Start with U series** (universal) for general workloads.
- Use **O series** if you need to pack more VMs on limited hardware (overcommits memory).
- Use **CX series** for CPU-bound workloads needing guaranteed performance.
- Use **M series** for databases or caches that need lots of memory.
- Use **D series** for enterprise apps needing dedicated CPU without hugepages.
- When a DataSource has instancetype/preference annotations, use `--infer-instancetype`
  and `--infer-preference` to pick them automatically.

## VM Disk Sources

VMs need boot disks. The `--volume-import` flag supports source types: `registry`, `ds`
(DataSource/golden image), `pvc`, `http`, `blank`, `s3`, `gcs`, and `snapshot`.

On OpenShift, golden images are available as DataSources in `openshift-virtualization-os-images`:

```bash
kubectl get datasource -n openshift-virtualization-os-images
```

> Full disk source table, containerdisk images, and DataSource details: [ref-types.md](ref-types.md)

## Creating VMs

The primary way to create VMs is with `kubectl virt create vm`. The command generates
a VirtualMachine manifest that you pipe to `kubectl apply`.

Prefer using `--instancetype` and `--preference` instead of `--memory`. Only fall back
to `--memory` when instance types are not available on the cluster.

### From a registry containerdisk (with instance type)

```bash
kubectl virt create vm \
  --name=my-fedora \
  --instancetype=u1.medium \
  --preference=fedora \
  --volume-import=type:registry,url:docker://quay.io/containerdisks/fedora:latest,size:30Gi \
  --user=fedora \
  --ssh-key="$(cat ~/.ssh/id_rsa.pub)" \
  | kubectl apply -f -
```

### From a cluster DataSource (inferred sizing)

```bash
kubectl virt create vm \
  --name=my-vm \
  --volume-import=type:ds,src:openshift-virtualization-os-images/fedora,size:30Gi \
  --infer-instancetype --infer-preference \
  --user=fedora \
  --ssh-key="$(cat ~/.ssh/id_rsa.pub)" \
  | kubectl apply -f -
```

### From an HTTP URL

```bash
kubectl virt create vm \
  --name=my-vm \
  --instancetype=u1.medium \
  --preference=fedora \
  --volume-import=type:http,url:https://example.com/my-image.qcow2,size:30Gi \
  --user=cloud-user \
  --ssh-key="$(cat ~/.ssh/id_rsa.pub)" \
  | kubectl apply -f -
```

### Fallback: raw memory (no instance types on cluster)

Note: `create vm` only supports `--memory`, there is no `--cpu` flag.
This is another reason to prefer instance types which set both CPU and memory.

```bash
kubectl virt create vm \
  --name=my-vm \
  --memory=2Gi \
  --run-strategy=Always \
  --volume-import=type:registry,url:docker://quay.io/containerdisks/fedora:latest,size:30Gi \
  --user=fedora \
  --ssh-key="$(cat ~/.ssh/id_rsa.pub)" \
  | kubectl apply -f -
```

### Key flags for `create vm`

- `--name` - VM name
- `--instancetype` - Instance type (e.g., `u1.medium`). Preferred over `--memory`.
- `--preference` - OS preference (e.g., `fedora`, `rhel.9`, `windows.11.virtio`)
- `--infer-instancetype` - Infer instancetype from the first boot disk annotations (default: true)
- `--infer-preference` - Infer preference from the first boot disk annotations (default: true)
- `--memory` - Guest memory fallback when no instance types available (e.g., 2Gi, 4Gi)
- `--run-strategy` - RunStrategy: Always, Manual, Halted, RerunOnFailure
- `--volume-import` - Import a volume (see Disk Sources table above). Can be repeated.
- `--volume-containerdisk` - Ephemeral containerdisk (not persisted). Format: `src:<image>`
- `--volume-pvc` - Use existing PVC directly (no clone). Format: `src:<pvc-name>`
- `--user` - Cloud-init user
- `--ssh-key` - SSH public key for cloud-init

### Important notes

- `kubectl virt create vm` outputs YAML to stdout - you must pipe it to `kubectl apply -f -`
- `--instancetype` is mutually exclusive with `--memory` -- use one or the other
- `--infer-instancetype` and `--infer-preference` default to true, so they work automatically when the boot disk DataSource has the right annotations
- The command does NOT set accessModes or storageClassName on the DataVolume. CDI infers these from the StorageProfile of the default StorageClass.
- If no default StorageClass is set, DataVolumes will get stuck with:
  `ErrClaimNotValid: PVC spec is missing accessMode and no storageClass to choose profile`
- Fix: ensure a default StorageClass exists:
  `kubectl annotate storageclass <name> storageclass.kubernetes.io/is-default-class=true`

### Creating multiple VMs (loop pattern)

```bash
SSH_KEY="$(cat ~/.ssh/id_rsa.pub)"
for i in 1 2 3 4; do
  kubectl virt create vm \
    --name="test-vm-${i}" \
    --instancetype=u1.medium \
    --preference=fedora \
    --volume-import=type:registry,url:docker://quay.io/containerdisks/fedora:latest,size:30Gi \
    --user=fedora \
    --ssh-key="$SSH_KEY" \
    | kubectl apply -f -
done
```

## SSH Access

There are multiple ways to access a VM over SSH, depending on the use case.

### Method 1: virtctl ssh (quick access via API server)

The simplest method. Tunnels SSH through the Kubernetes API server -- no service needed.
Best for troubleshooting and occasional access. Not recommended for high-traffic or production use
as it adds load to the API server.

```bash
# SSH into a VM (uses your default SSH key)
kubectl virt ssh fedora@my-vm

# Specify identity file
kubectl virt ssh fedora@my-vm --identity-file=~/.ssh/id_rsa

# Run a command
kubectl virt ssh fedora@my-vm --command="uname -a"

# SCP files to/from a VM
kubectl virt scp myfile.bin fedora@vmi/my-vm:myfile.bin
kubectl virt scp fedora@vmi/my-vm:remote-file.bin ./local-file.bin
kubectl virt scp --recursive ~/mydir/ fedora@vmi/my-vm:./mydir
```

Other methods include port-forward (SSH config integration), NodePort Service
(external access), ClusterIP Service (cluster-internal access), and secondary networks.

> Methods 2-5 with full examples: [ref-types.md](ref-types.md)

## Common VM Operations

For any command you are unsure about, run `kubectl virt <command> --help` first.

```bash
# List VMs
kubectl get vm
kubectl get vm -n <namespace>

# Start/stop/restart
kubectl virt start <vm-name>
kubectl virt stop <vm-name>
kubectl virt restart <vm-name>

# Pause/unpause
kubectl virt pause vm <vm-name>
kubectl virt unpause vm <vm-name>

# Console access
kubectl virt console <vm-name>
kubectl virt vnc <vm-name>

# Migration (live migrate)
kubectl virt migrate <vm-name>

# Get VM details
kubectl describe vm <vm-name>
kubectl get vmi <vm-name>
```

## Self-Learning Rule

When you encounter an unfamiliar `kubectl virt` subcommand or need to verify flags, always run:

```bash
kubectl virt <command> --help
```

This ensures you use the correct and current syntax.
