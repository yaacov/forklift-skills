---
name: check-ocp-health
description: General OpenShift (OCP) cluster health check. Use when the cluster is unhealthy, nodes are NotReady, operators are degraded, pods are crashing, etcd is slow, networking issues occur, or a general cluster diagnosis is needed.
---

# OpenShift Cluster Health Check

Use this guide for general OCP cluster health diagnosis and remediation.

## Required MCP Servers

This skill requires:
- `debug_read` (from the kubectl-debug-queries MCP server) -- for listing resources, logs, events
- `metrics_read` (from the kubectl-metrics MCP server) -- for CPU/memory/node metrics

If any of these tools are not available in your environment, inform the user and refer them to the `mcp-setup` skill for installation instructions. Do not attempt bash fallback.

## Quick Triage

Check these in order for a fast overview:

```
debug_read   command: "list"    flags: {resource: "nodes"}
debug_read   command: "list"    flags: {resource: "clusteroperators"}
debug_read   command: "list"    flags: {resource: "pods", all_namespaces: true, query: "where status.phase != 'Running' and status.phase != 'Succeeded'", limit: 30}
debug_read   command: "events"  flags: {all_namespaces: true, query: "where type = 'Warning'", sort_by: "time_desc", limit: 20}
```

## 1. Nodes

```
debug_read  command: "list"  flags: {resource: "nodes"}
debug_read  command: "get"   flags: {resource: "node", name: "<node-name>"}
```

Resource usage:

```
metrics_read  command: "preset"  flags: {name: "cluster_cpu_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_memory_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_node_readiness"}
```

Pods on a specific node:

```
debug_read  command: "list"  flags: {resource: "pods", all_namespaces: true, query: "where spec.nodeName = '<node-name>'"}
```

### Node NotReady

**Diagnosis**:

```
debug_read  command: "get"     flags: {resource: "node", name: "<node-name>"}
debug_read  command: "events"  flags: {all_namespaces: true, name: "<node-name>", sort_by: "time_desc"}
```

**Common causes**:
- Kubelet not running
- Network partition -- node can't reach API server
- Disk pressure -- node disk full
- Memory pressure -- OOM conditions

**Remediation**:
- For disk pressure: clean up logs, images, or unused containers on the node
- For kubelet issues: restart kubelet on the node (requires shell)
- For unrecoverable nodes: cordon, drain, and replace

## 2. Cluster Operators

```
debug_read  command: "list"  flags: {resource: "clusteroperators"}
debug_read  command: "get"   flags: {resource: "clusteroperator", name: "<operator-name>"}
```

Key operators to watch: `etcd`, `kube-apiserver`, `openshift-controller-manager`, `ingress`, `monitoring`, `storage`, `machine-config`.

### Degraded Operator

**Diagnosis**: Check the operator's namespace for unhealthy pods:

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-<operator-name>", query: "where status.phase != 'Running'"}
debug_read  command: "logs"  flags: {name: "<pod-name>", namespace: "openshift-<operator-name>", tail: 50}
```

**Remediation**:
- Restart the operator pod if it's stuck
- Check if a dependent service (etcd, API server) is down
- Review MachineConfigPool if `machine-config` operator is degraded

## 3. etcd Health

```
debug_read  command: "get"   flags: {resource: "clusteroperator", name: "etcd"}
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-etcd", selector: "app=etcd"}
debug_read  command: "logs"  flags: {name: "deployment/etcd-operator", namespace: "openshift-etcd-operator", tail: 50, query: "where level = 'ERROR' or level = 'WARN'"}
```

### etcd Slow or Degraded

**Common causes**:
- Slow disk I/O -- etcd needs fast storage (SSD recommended)
- Network latency between control plane nodes
- Database too large (fragmentation)

**Remediation**:
- Check disk performance on control plane nodes
- Defragment etcd if DB size is large (done automatically by the operator)
- Ensure control plane nodes have low-latency network

## 4. API Server

```
debug_read  command: "list"    flags: {resource: "pods", namespace: "openshift-kube-apiserver", selector: "app=openshift-kube-apiserver"}
debug_read  command: "events"  flags: {namespace: "openshift-kube-apiserver", sort_by: "time_desc", limit: 10}
```

## 5. Pods and Workloads

```
debug_read  command: "list"  flags: {resource: "pods", all_namespaces: true, query: "where status.phase = 'Failed'", limit: 20}
debug_read  command: "list"  flags: {resource: "pods", all_namespaces: true, query: "where status.phase = 'Pending'"}
```

Pods with high restart counts:

```
metrics_read  command: "preset"  flags: {name: "pod_restarts_top10"}
```

### CrashLoopBackOff

**Diagnosis**:

```
debug_read  command: "get"   flags: {resource: "pod", name: "<pod-name>", namespace: "<namespace>"}
debug_read  command: "logs"  flags: {name: "<pod-name>", namespace: "<namespace>", previous: true}
```

**Common causes**: missing config/secrets, OOM, application errors, image issues.

### ImagePullBackOff

**Diagnosis**:

```
debug_read  command: "events"  flags: {namespace: "<namespace>", name: "<pod-name>", resource: "Pod"}
```

**Common causes**: wrong image name, registry auth missing, network issues to registry.

## 6. Networking

```
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-ingress"}
debug_read  command: "get"   flags: {resource: "clusteroperator", name: "network"}
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-network-operator"}
```

### Service/Route Not Reachable

**Diagnosis**:

```
debug_read  command: "list"  flags: {resource: "endpoints", namespace: "<namespace>"}
debug_read  command: "list"  flags: {resource: "pods", namespace: "openshift-ingress"}
debug_read  command: "logs"  flags: {name: "<router-pod>", namespace: "openshift-ingress", tail: 20}
```

## 7. Certificates

```
debug_read  command: "list"  flags: {resource: "certificates", all_namespaces: true}
debug_read  command: "get"   flags: {resource: "clusteroperator", name: "kube-apiserver"}
```

## 8. MachineConfigPool (Node Updates)

```
debug_read  command: "list"  flags: {resource: "machineconfigpool"}
debug_read  command: "get"   flags: {resource: "machineconfigpool", name: "worker"}
debug_read  command: "get"   flags: {resource: "machineconfigpool", name: "master"}
```

### Nodes Stuck Updating

**Diagnosis**:

```
debug_read  command: "list"  flags: {resource: "machineconfigpool", output: "json"}
```

**Remediation**:
- Check the machine-config-daemon pod on the stuck node
- Review logs: `debug_read` logs for the machine-config-daemon pod on that node
- A degraded MCP often means a config failed to apply -- fix the MachineConfig or remove it

## 9. Cluster Version and Updates

```
debug_read  command: "list"  flags: {resource: "clusterversion"}
debug_read  command: "get"   flags: {resource: "clusterversion", name: "version"}
```

## 10. Resource Quotas and Limits

```
debug_read  command: "list"  flags: {resource: "resourcequota", all_namespaces: true}
debug_read  command: "list"  flags: {resource: "limitrange", all_namespaces: true}
debug_read  command: "get"   flags: {resource: "resourcequota", name: "<quota-name>", namespace: "<namespace>"}
```

## 11. Full Health Report

When the user asks for a cluster health report, run these commands **in parallel** and present the results as a formatted summary with tables:

**Cluster & nodes:**

```
debug_read   command: "list"    flags: {resource: "clusterversion"}
debug_read   command: "list"    flags: {resource: "nodes"}
debug_read   command: "list"    flags: {resource: "clusteroperators"}
debug_read   command: "list"    flags: {resource: "pods", all_namespaces: true, query: "where status.phase != 'Running' and status.phase != 'Succeeded'", limit: 15}
```

**Resource usage:**

```
metrics_read  command: "preset"  flags: {name: "cluster_cpu_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_memory_utilization"}
metrics_read  command: "preset"  flags: {name: "cluster_node_readiness"}
```

**Storage health:**

```
metrics_read  command: "query"  flags: {query: "ceph_health_status"}
metrics_read  command: "query"  flags: {query: "ceph_cluster_total_used_bytes / ceph_cluster_total_bytes * 100"}
```

### How to present the report

Format the results as a concise summary with:

- **Cluster Overview** section: version, node count/status, operator health, problem pods
- **Storage** section: Ceph health, capacity used/available/percentage as a table
- **Memory & CPU** section: per-node usage as a table, highlight nodes above 70% memory or 80% CPU

Flag any issues found with brief remediation hints. If everything is healthy, say so clearly.

---

## Requires Shell

These operations cannot be performed via MCP and require shell access:

### API server raw health check

```bash
kubectl get --raw /healthz
```

### etcd member health (exec into etcd pod)

```bash
kubectl -n openshift-etcd exec $(kubectl -n openshift-etcd get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}') -c etcd -- \
  etcdctl member list -w table

kubectl -n openshift-etcd exec $(kubectl -n openshift-etcd get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}') -c etcd -- \
  etcdctl endpoint health --cluster -w table

kubectl -n openshift-etcd exec $(kubectl -n openshift-etcd get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}') -c etcd -- \
  etcdctl endpoint status --cluster -w table
```

### DNS resolution test

```bash
kubectl run dns-test --rm -i --restart=Never --image=busybox -- nslookup kubernetes.default.svc.cluster.local
```

### Certificate expiry inspection

```bash
kubectl get secret -n openshift-kube-apiserver -o json | \
  python3 -c "import json,sys; [print(i['metadata']['name']) for i in json.load(sys.stdin)['items'] if 'cert' in i['metadata']['name'].lower()]" 2>/dev/null
```

## Self-Learning Rule

When you need to discover available flags or verify syntax, call the MCP help tools:

```
debug_help  command: "list"
debug_help  command: "logs"
metrics_help  command: "preset"
metrics_help  command: "promql"
```
