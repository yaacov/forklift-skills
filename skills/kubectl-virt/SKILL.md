---
name: kubectl-virt
description: Use kubectl virt (or oc virt) to manage KubeVirt virtual machines. Use this skill when the user wants to create, start, stop, or manage VMs on OpenShift/Kubernetes.
---

# kubectl virt - KubeVirt VM Management

`kubectl virt` (or `oc virt`) is a kubectl plugin for managing KubeVirt virtual machines.

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

### Instance type series

| Series | Name | Use Case | CPU | Memory Ratio | Key Features |
|--------|------|----------|-----|--------------|--------------|
| **U** | Universal | General purpose | Burstable | 1:4 | Good default for most workloads |
| **O** | Overcommitted | Higher density | Burstable | 1:4 | Memory overcommitted for more VMs per node |
| **CX** | Compute Exclusive | Compute intensive | Dedicated | 1:2 | Hugepages, vNUMA, isolated emulator threads |
| **M** | Memory | Memory intensive | Burstable | 1:8 | Hugepages, high memory per vCPU |
| **N** | Network | DPDK/VNFs | Dedicated | 1:2 | Hugepages, requires DPDK-capable nodes |
| **RT** | Realtime | Realtime apps | Dedicated | 1:4 | Hugepages, isolated emulator threads |
| **D** | Dedicated | Predictable compute | Dedicated | 1:4 | Dedicated CPU, isolated emulator threads |

### Sizing

Each series comes in standard sizes:

| Size | vCPUs | Example (U series memory) |
|------|-------|--------------------------|
| nano | 1 | 512Mi |
| micro | 1 | 1Gi |
| small | 1 | 2Gi |
| medium | 1 | 4Gi |
| large | 2 | 8Gi |
| xlarge | 4 | 16Gi |
| 2xlarge | 8 | 32Gi |
| 4xlarge | 16 | 64Gi |
| 8xlarge | 32 | 128Gi |

Naming pattern: `<series><version>.<size>` (e.g., `u1.medium`, `cx1.xlarge`, `m1.2xlarge`).

### Preferences

Preferences configure OS-specific settings. Common ones:

- `fedora`, `centos.stream9`, `centos.stream10`, `ubuntu`, `debian`
- `rhel.9`, `rhel.10`, `rhel.8`, `rhel.7`
- `windows.11.virtio`, `windows.2k22.virtio`, `windows.10.virtio`
- `opensuse.tumbleweed`, `opensuse.leap`, `sles`

### How to choose

- **Start with U series** (universal) for general workloads.
- Use **O series** if you need to pack more VMs on limited hardware (overcommits memory).
- Use **CX series** for CPU-bound workloads needing guaranteed performance.
- Use **M series** for databases or caches that need lots of memory.
- Use **D series** for enterprise apps needing dedicated CPU without hugepages.
- When a DataSource has instancetype/preference annotations, use `--infer-instancetype`
  and `--infer-preference` to pick them automatically.

## VM Disk Sources

VMs need boot disks. The `--volume-import` flag on `kubectl virt create vm` supports these source types:

| Type | Description | Example |
|------|-------------|---------|
| `registry` | Container disk from a registry | `type:registry,url:docker://quay.io/containerdisks/fedora:latest,size:30Gi` |
| `ds` | DataSource (cluster golden image) | `type:ds,src:fedora,size:30Gi` or `type:ds,src:my-ns/my-ds` |
| `pvc` | Clone an existing PVC | `type:pvc,src:my-ns/my-pvc,size:30Gi` |
| `http` | Download from HTTP/HTTPS URL | `type:http,url:https://example.com/disk.qcow2,size:30Gi` |
| `blank` | Empty disk | `type:blank,size:50Gi` |
| `s3` | Download from S3 | `type:s3,url:s3://bucket/disk.qcow2,size:30Gi` |
| `gcs` | Download from GCS | `type:gcs,url:gcs://bucket/disk.qcow2,size:30Gi` |
| `snapshot` | Clone from VolumeSnapshot | `type:snapshot,src:my-ns/my-snap,size:30Gi` |

### Upstream containerdisk images (quay.io/containerdisks)

Official KubeVirt containerdisks maintained at `quay.io/containerdisks`:

- `quay.io/containerdisks/fedora` (amd64, arm64, s390x)
- `quay.io/containerdisks/centos-stream` (amd64, arm64, s390x)
- `quay.io/containerdisks/ubuntu` (amd64, arm64, s390x)
- `quay.io/containerdisks/debian` (amd64, arm64)
- `quay.io/containerdisks/opensuse-tumbleweed` (amd64, s390x)
- `quay.io/containerdisks/opensuse-leap` (amd64, arm64)

Use with tag for specific versions (e.g., `quay.io/containerdisks/fedora:41`).

### Cluster DataSources (golden images)

On OpenShift, golden images are pre-imported as DataSources and kept updated
via DataImportCron jobs. These live in the `openshift-virtualization-os-images`
namespace and can be used directly as boot sources without downloading anything.

```bash
# List available DataSources (golden images) in the cluster
kubectl get datasource -n openshift-virtualization-os-images

# Create a VM from a cluster DataSource (infer sizing from annotations)
kubectl virt create vm \
  --name=my-vm \
  --volume-import=type:ds,src:openshift-virtualization-os-images/fedora,size:30Gi \
  --infer-instancetype --infer-preference \
  | kubectl apply -f -
```

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

### Method 2: virtctl port-forward (SSH config integration)

Forwards a local port through the API server. Useful for integrating with your
`~/.ssh/config`. Same API server caveat as Method 1.

```bash
# Forward local port 2222 to VM port 22
kubectl virt port-forward vm/my-vm 2222:22

# Then in another terminal
ssh -p 2222 fedora@127.0.0.1
```

### Method 3: NodePort Service (external access without cloud LB)

Creates a Kubernetes Service that maps the VM's SSH port to a high port (30000-32767)
on every cluster node. You can then SSH from outside the cluster using any node's IP
and the assigned port. Works on any cluster (bare metal, cloud, etc.).

```bash
# Create a NodePort service for SSH
kubectl virt expose vm my-vm --name=my-vm-ssh --port=22 --type=NodePort

# Check which port was assigned
kubectl get svc my-vm-ssh
# Example output:
#   NAME        TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
#   my-vm-ssh   NodePort   10.96.0.123   <none>        22:31245/TCP   5s
#                                                          ^^^^^
#                                                    this is the node port

# SSH using any cluster node IP and the assigned NodePort
ssh -p 31245 fedora@<any-node-ip>

# To find node IPs:
kubectl get nodes -o wide
```

### Method 4: ClusterIP Service (access from within the cluster)

For access from other pods/VMs inside the same cluster only. Not reachable from outside.

```bash
kubectl virt expose vm my-vm --name=my-vm-ssh --port=22 --type=ClusterIP
# Connect from another pod in the cluster:
ssh fedora@my-vm-ssh.<namespace>.svc.cluster.local
```

### Method 5: Secondary network

Attach the VM to a secondary network (bridge, SR-IOV) where it gets a DHCP address.
Connect directly to that IP. This requires network infrastructure setup (not covered here).

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
