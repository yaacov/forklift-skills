# KubeVirt VM Metrics

Queries, labels, and metrics for KubeVirt virtual machines. Use `name` and `namespace` labels to target a specific VM. See [SKILL.md](SKILL.md) for general usage.

## Top VMs by resource usage (presets)

```
metrics_read  command: "preset"  flags: {name: "vm_cpu_usage"}
metrics_read  command: "preset"  flags: {name: "vm_memory_usage"}
metrics_read  command: "preset"  flags: {name: "vm_network_rx"}
metrics_read  command: "preset"  flags: {name: "vm_network_tx"}
metrics_read  command: "preset"  flags: {name: "vm_storage_read"}
metrics_read  command: "preset"  flags: {name: "vm_storage_write"}
metrics_read  command: "preset"  flags: {name: "vm_storage_iops"}
```

## Per-VM queries (ad-hoc)

Replace `VM_NAME` and `VM_NAMESPACE` with actual values:

```
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_cpu_usage_seconds_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
metrics_read  command: "query"  flags: {query: "kubevirt_vmi_memory_resident_bytes{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_network_receive_bytes_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_network_transmit_bytes_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_storage_read_traffic_bytes_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_storage_write_traffic_bytes_total{name=\"VM_NAME\",namespace=\"VM_NAMESPACE\"}[5m])"}
```

## VM disk IOPS and latency

```
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_storage_iops_read_total{name=\"VM_NAME\"}[5m])"}
metrics_read  command: "query"  flags: {query: "rate(kubevirt_vmi_storage_read_times_seconds_total{name=\"VM_NAME\"}[5m]) / rate(kubevirt_vmi_storage_iops_read_total{name=\"VM_NAME\"}[5m])"}
```

## Available labels on kubevirt_vmi_* metrics

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

## Metrics reference

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
