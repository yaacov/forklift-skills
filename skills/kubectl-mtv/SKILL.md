---
name: kubectl-mtv
description: Use MTV/Forklift MCP tools to manage VM migrations. Use this skill when the user wants to migrate VMs from vSphere, oVirt, OpenStack, OVA, EC2, or HyperV to OpenShift/KubeVirt.
---

# MTV/Forklift - Migration Toolkit for Virtualization

Manage VM migrations from VMware vSphere, oVirt (RHV), OpenStack, OVA, EC2, and HyperV to OpenShift Virtualization (KubeVirt) using the MTV MCP tools.

## Required MCP Servers

This skill requires: `mtv_read`, `mtv_write`, and `mtv_help` (from the kubectl-mtv MCP server).

If these tools are not available in your environment, inform the user and refer them to the `mcp-setup` skill for installation instructions. Do not attempt bash fallback.

## Getting Help

Always call `mtv_help` before using an unfamiliar command to learn its flags and see examples:

```
mtv_help  command: "create provider"
mtv_help  command: "create plan"
mtv_help  command: "get inventory vm"
mtv_help  command: "tsl"
mtv_help  command: "karl"
```

## Typical Migration Workflow

### 1. Check system health

```
mtv_read  command: "health"
mtv_read  command: "health"  flags: {all_namespaces: true}
mtv_read  command: "health"  flags: {skip_logs: true}
```

### 2. Configure settings (e.g., VDDK image for vSphere)

```
mtv_write  command: "settings set"  flags: {setting: "vddk_image", value: "<registry-url>/vddk"}
mtv_read   command: "settings get"  flags: {setting: "vddk_image"}
mtv_write  command: "settings unset"  flags: {setting: "vddk_image"}
mtv_read   command: "settings"  flags: {all: true}
```

To build a VDDK image from the VMware SDK tar:

```
mtv_write  command: "create vddk-image"  flags: {tar: "VMware-vix-disklib-8.0.1.tar.gz", tag: "quay.io/myorg/vddk:8.0.1", push: true}
mtv_write  command: "create vddk-image"  flags: {tar: "VMware-vix-disklib-8.0.1.tar.gz", tag: "quay.io/myorg/vddk:8.0.1", push: true, set_controller_image: true}
```

### 3. Create providers

For vSphere providers, the VDDK init image is required. Check if it is already configured globally:

```
mtv_read  command: "settings get"  flags: {setting: "vddk_image"}
```

If a global VDDK image is set, you do NOT need `vddk_init_image` on the provider. If it is not set, prefer setting it globally (if you have permissions). Only use `vddk_init_image` on the provider as a fallback.

```
mtv_write  command: "create provider"  flags: {
  name: "host", type: "openshift", namespace: "<namespace>"
}

mtv_write  command: "create provider"  flags: {
  name: "my-vsphere", type: "vsphere",
  url: "https://vcenter.example.com/sdk",
  username: "admin@vsphere.local", password: "$PASSWORD",
  namespace: "<namespace>"
}

mtv_write  command: "create provider"  flags: {
  name: "my-ovirt", type: "ovirt",
  url: "https://rhv-manager.example.com/ovirt-engine/api",
  username: "admin@internal", password: "$PASSWORD",
  namespace: "<namespace>"
}

mtv_write  command: "create provider"  flags: {
  name: "my-ec2", type: "ec2",
  ec2_region: "us-east-1",
  username: "$EC2_KEY", password: "$EC2_SECRET",
  auto_target_credentials: true,
  namespace: "<namespace>"
}

mtv_write  command: "create provider"  flags: {
  name: "my-hyperv", type: "hyperv",
  url: "https://192.168.1.100",
  username: "Administrator", password: "$PASSWORD",
  smb_url: "//192.168.1.100/VMShare",
  namespace: "<namespace>"
}
```

### 4. List providers and verify

```
mtv_read  command: "get provider"  flags: {namespace: "<namespace>"}
mtv_read  command: "get provider"  flags: {all_namespaces: true}
```

### 5. Browse inventory VMs

```
mtv_read  command: "get inventory vm"  flags: {provider: "my-vsphere", namespace: "<namespace>"}

mtv_read  command: "get inventory vm"  flags: {
  provider: "my-vsphere", namespace: "<namespace>",
  query: "where name ~= 'prod-.*'"
}

mtv_read  command: "get inventory vm"  flags: {
  provider: "my-vsphere", namespace: "<namespace>",
  query: "where powerState = 'poweredOn' and memoryMB > 4096"
}

mtv_read  command: "get inventory vm"  flags: {
  provider: "my-vsphere", namespace: "<namespace>",
  query: "where cpuCount > 4 and len(disks) > 1"
}

mtv_read  command: "get inventory vm"  flags: {
  provider: "my-vsphere", namespace: "<namespace>",
  query: "where name ~= 'web-.*'", output: "planvms"
}
```

### 6. Browse other inventory resources

```
mtv_read  command: "get inventory network"   flags: {provider: "my-vsphere", namespace: "<namespace>"}
mtv_read  command: "get inventory storage"   flags: {provider: "my-vsphere", namespace: "<namespace>"}
mtv_read  command: "get inventory host"      flags: {provider: "my-vsphere", namespace: "<namespace>"}
mtv_read  command: "get inventory cluster"   flags: {provider: "my-vsphere", namespace: "<namespace>"}
mtv_read  command: "get inventory datacenter"  flags: {provider: "my-vsphere", namespace: "<namespace>"}
mtv_read  command: "get inventory datastore" flags: {provider: "my-vsphere", namespace: "<namespace>"}
```

### 7. Create a migration plan

#### Prerequisites

A plan requires an OpenShift host (target) provider in the same namespace. Verify one exists:

```
mtv_read  command: "get provider"  flags: {namespace: "<namespace>"}
```

If no OpenShift provider is listed, create one:

```
mtv_write  command: "create provider"  flags: {name: "host", type: "openshift", namespace: "<namespace>"}
```

#### Creating plans

Only `name`, `source`, and `vms` are required. The target provider, network/storage mappings, and other settings are auto-detected. Only add optional flags when you need to override defaults.

```
mtv_write  command: "create plan"  flags: {
  name: "my-migration", source: "my-vsphere",
  vms: "vm1,vm2,vm3",
  namespace: "<namespace>"
}

mtv_write  command: "create plan"  flags: {
  name: "my-migration", source: "my-vsphere",
  vms: "where name ~= 'prod-.*'",
  namespace: "<namespace>"
}

mtv_write  command: "create plan"  flags: {
  name: "my-warm", source: "my-vsphere",
  vms: "critical-vm", migration_type: "warm",
  namespace: "<namespace>"
}
```

Override defaults only when auto-detection doesn't suit your needs:

```
mtv_write  command: "create plan"  flags: {
  name: "my-migration", source: "my-vsphere", target: "host",
  vms: "vm1,vm2",
  default_target_network: "default",
  default_target_storage_class: "standard",
  namespace: "<namespace>"
}
```

#### Verify plan health

Plans referencing invalid storage classes or networks are accepted at creation time but fail at the controller level. Always verify the plan is ready after creating it:

```
mtv_read  command: "get plan"  flags: {namespace: "<namespace>"}
```

If READY shows `false`, check conditions:

```
debug_read  command: "get"  flags: {resource: "plans", name: "<plan-name>", namespace: "<namespace>", output: "json", query: "select name, status.conditions"}
```

### 8. Start migration

```
mtv_write  command: "start plan"  flags: {name: "my-migration", namespace: "<namespace>"}
mtv_write  command: "start plan"  flags: {name: "plan1,plan2", namespace: "<namespace>"}
```

### 9. Monitor migration

```
mtv_read  command: "get plan"  flags: {namespace: "<namespace>"}
mtv_read  command: "get plan"  flags: {name: "my-migration", namespace: "<namespace>"}
mtv_read  command: "get plan"  flags: {name: "my-migration", vms: true, namespace: "<namespace>"}
mtv_read  command: "get plan"  flags: {name: "my-migration", disk: true, namespace: "<namespace>"}
mtv_read  command: "get plan"  flags: {name: "my-migration", vms: true, disk: true, namespace: "<namespace>"}
mtv_read  command: "get plan"  flags: {vms_table: true, namespace: "<namespace>"}
mtv_read  command: "get plan"  flags: {vms_table: true, query: "where planStatus = 'Failed'", namespace: "<namespace>"}
```

### 10. View logs

The `health` command includes built-in log analysis. Use `skip_logs: false` (the default) and adjust `log_lines` to control how many lines per pod are analyzed:

```
mtv_read  command: "health"  flags: {namespace: "<namespace>"}
mtv_read  command: "health"  flags: {all_namespaces: true, log_lines: 200}
```

For targeted log inspection of specific Forklift pods, use `debug_read`. First discover the operator namespace via `mtv_read health` (the output includes "Namespace: <actual-namespace>"):

```
debug_read  command: "logs"  flags: {name: "deployment/forklift-controller", namespace: "<forklift-namespace>", container: "main", tail: 100}
debug_read  command: "logs"  flags: {name: "deployment/forklift-controller", namespace: "<forklift-namespace>", container: "main", tail: 100, query: "where level = 'ERROR'"}
```

Before writing log queries, discover the actual field names and values:

```
debug_read  command: "logs"  flags: {name: "deployment/forklift-controller", namespace: "<forklift-namespace>", container: "main", tail: 5, output: "json"}
```

Full-text search when you don't know which field contains the value:

```
debug_read  command: "logs"  flags: {name: "deployment/forklift-controller", namespace: "<forklift-namespace>", container: "main", tail: 200, query: "where raw_line ~= '.*<search-term>.*'"}
```

### 11. Plan lifecycle

```
mtv_write  command: "cancel plan"   flags: {name: "my-migration", vms: "vm1,vm2", namespace: "<namespace>"}
mtv_write  command: "cutover plan"  flags: {name: "my-warm", namespace: "<namespace>"}
mtv_write  command: "archive plan"  flags: {name: "my-migration", namespace: "<namespace>"}
mtv_write  command: "unarchive plan"  flags: {name: "my-migration", namespace: "<namespace>"}
```

### 12. Modify existing resources

```
mtv_write  command: "patch plan"     flags: {plan_name: "my-migration", migration_type: "warm", namespace: "<namespace>"}
mtv_write  command: "patch plan"     flags: {plan_name: "my-migration", target_labels: "env=prod,team=platform", namespace: "<namespace>"}
mtv_write  command: "patch planvm"   flags: {plan_name: "my-migration", vm: "vm1", target_name: "new-vm-name", namespace: "<namespace>"}
mtv_write  command: "patch provider" flags: {name: "my-vsphere", url: "https://new-vcenter.example.com/sdk", namespace: "<namespace>"}
```

### 13. Cleanup

```
mtv_write  command: "delete plan"  flags: {name: "my-migration", namespace: "<namespace>"}
mtv_write  command: "delete provider"  flags: {name: "my-vsphere", namespace: "<namespace>"}
```

## TSL Query Syntax (for vms and query flags)

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

```
mtv_read  command: "get mapping network"  flags: {namespace: "<namespace>"}
mtv_read  command: "get mapping storage"  flags: {namespace: "<namespace>"}

mtv_write  command: "create mapping network"  flags: {
  name: "my-net", source: "my-vsphere", target: "host",
  network_pairs: "VM Network:default", namespace: "<namespace>"
}

mtv_write  command: "create mapping storage"  flags: {
  name: "my-store", source: "my-vsphere", target: "host",
  storage_pairs: "datastore1:standard", namespace: "<namespace>"
}

mtv_read  command: "get hook"  flags: {namespace: "<namespace>"}
mtv_read  command: "get host"  flags: {namespace: "<namespace>"}

mtv_read  command: "describe plan"  flags: {name: "my-migration", namespace: "<namespace>"}
mtv_read  command: "describe provider"  flags: {name: "my-vsphere", namespace: "<namespace>"}
mtv_read  command: "describe mapping network"  flags: {name: "my-net", namespace: "<namespace>"}
```

## KARL Affinity Syntax

The `create plan` and `patch plan` commands support `target_affinity` and `convertor_affinity` flags using KARL syntax for pod placement rules:

```
RULE_TYPE pods(selector) on TOPOLOGY [weight=N]
```

Rule types: `REQUIRE` (hard affinity), `PREFER` (soft affinity), `AVOID` (hard anti-affinity), `REPEL` (soft anti-affinity). Topology: `node`, `zone`, `region`, `rack`.

```
mtv_write  command: "create plan"  flags: {
  name: "my-plan", source: "my-vsphere", vms: "db-vm",
  target_affinity: "REQUIRE pods(app=database) on node",
  namespace: "<namespace>"
}
```

For the full KARL reference, call `mtv_help command: "karl"`.

## Tips

### Limit JSON output with `fields`

`fields` is a **top-level** parameter on `mtv_read` (not inside `flags`). Use it to limit JSON response size:

```
mtv_read  command: "get plan"  flags: {output: "json", namespace: "<namespace>"}  fields: ["name", "status"]
mtv_read  command: "get inventory vm"  flags: {provider: "my-vsphere", output: "json", namespace: "<namespace>"}  fields: ["name", "cpuCount", "memoryMB"]
```

### Preview commands with `dry_run`

`dry_run` is a top-level parameter that shows the equivalent CLI command without executing:

```
mtv_write  command: "create plan"  flags: {name: "test", source: "my-vsphere", vms: "vm1", namespace: "<namespace>"}  dry_run: true
```

## Self-Learning Rule

When you encounter an unfamiliar MTV command or need to verify flags, always call:

```
mtv_help  command: "<command>"
mtv_help  command: "<command> <subcommand>"
```

This ensures you use the correct and current syntax.
