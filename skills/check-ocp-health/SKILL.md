---
name: check-ocp-health
description: General OpenShift (OCP) cluster health check. Use when the cluster is unhealthy, nodes are NotReady, operators are degraded, pods are crashing, etcd is slow, networking issues occur, or a general cluster diagnosis is needed.
---

# OpenShift Cluster Health Check

Use this guide for general OCP cluster health diagnosis and remediation.

## Quick Triage

Check these in order for a fast overview:

```bash
# 1. Node status
kubectl get nodes

# 2. Cluster operators (any Degraded or not Available?)
kubectl get clusteroperators

# 3. Pods not running across the cluster
kubectl get pods --all-namespaces --field-selector 'status.phase!=Running,status.phase!=Succeeded' | head -30

# 4. Recent warning events cluster-wide
kubectl get events --all-namespaces --field-selector type=Warning --sort-by='.lastTimestamp' | tail -20
```

## 1. Nodes

```bash
# Node status and roles
kubectl get nodes -o wide

# Resource capacity vs allocatable
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.conditions[-1].type,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory,PODS:.status.allocatable.pods'

# Actual resource usage (requires metrics-server)
kubectl top nodes

# Check conditions on a specific node (pressure, taints, readiness)
kubectl describe node <node-name> | grep -A10 'Conditions:'

# Check taints preventing scheduling
kubectl get nodes -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints[*].effect'

# Pods on a specific node
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name>
```

### Node NotReady

**Diagnosis**:
```bash
kubectl describe node <node-name> | grep -A5 -E 'Conditions:|Taints:'
kubectl get events --field-selector involvedObject.name=<node-name> --sort-by='.lastTimestamp'
```

**Common causes**:
- Kubelet not running -- SSH to node, check `systemctl status kubelet`
- Network partition -- node can't reach API server
- Disk pressure -- node disk full, check `df -h` on the node
- Memory pressure -- OOM conditions on the node

**Remediation**:
- For disk pressure: clean up logs, images, or unused containers on the node
- For kubelet issues: `systemctl restart kubelet` on the node
- For unrecoverable nodes: cordon, drain, and replace

## 2. Cluster Operators

```bash
# Overview (look for AVAILABLE=False or DEGRADED=True)
kubectl get clusteroperators

# Details on a degraded operator
kubectl describe clusteroperator <operator-name>

# Operator conditions in structured form
kubectl get clusteroperator <operator-name> -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

Key operators to watch:
- `etcd` -- cluster database
- `kube-apiserver` -- API server
- `openshift-controller-manager` -- controllers
- `ingress` -- routes and external access
- `monitoring` -- Prometheus/alerting
- `storage` -- CSI drivers and storage
- `machine-config` -- node configuration

### Degraded Operator

**Diagnosis**: Check the operator's namespace for unhealthy pods:
```bash
# Find the operator's namespace (usually matches the operator name)
kubectl get pods -n openshift-<operator-name> | grep -v Running
kubectl logs -n openshift-<operator-name> <pod-name> --tail=50
```

**Remediation**:
- Restart the operator pod if it's stuck
- Check if a dependent service (etcd, API server) is down
- Review MachineConfigPool if `machine-config` operator is degraded

## 3. etcd Health

etcd is critical -- if it's unhealthy, the entire cluster is at risk.

```bash
# etcd operator status
kubectl get clusteroperator etcd

# etcd pods
kubectl get pods -n openshift-etcd -l app=etcd

# etcd member health (run from an etcd pod)
kubectl -n openshift-etcd exec $(kubectl -n openshift-etcd get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}') -c etcd -- \
  etcdctl member list -w table

# etcd endpoint health
kubectl -n openshift-etcd exec $(kubectl -n openshift-etcd get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}') -c etcd -- \
  etcdctl endpoint health --cluster -w table

# etcd database size (should be < 8GB, warn at 6GB)
kubectl -n openshift-etcd exec $(kubectl -n openshift-etcd get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}') -c etcd -- \
  etcdctl endpoint status --cluster -w table
```

### etcd Slow or Degraded

**Common causes**:
- Slow disk I/O -- etcd needs fast storage (SSD recommended)
- Network latency between control plane nodes
- Database too large (fragmentation)

**Remediation**:
- Check disk performance on control plane nodes
- Defragment etcd if DB size is large: done automatically by the operator, but can be triggered
- Ensure control plane nodes have low-latency network between them

## 4. API Server

```bash
# API server pods
kubectl get pods -n openshift-kube-apiserver -l app=openshift-kube-apiserver

# API server responsiveness
kubectl get --raw /healthz

# API server audit events (if API calls are slow)
kubectl get events -n openshift-kube-apiserver --sort-by='.lastTimestamp' | tail -10
```

## 5. Pods and Workloads

```bash
# Pods in bad states across the cluster
kubectl get pods --all-namespaces --field-selector 'status.phase=Failed' | head -20
kubectl get pods --all-namespaces | grep -E 'CrashLoopBackOff|Error|ImagePullBackOff|Pending|OOMKilled' | head -20

# Pods with high restart counts (sign of instability)
kubectl get pods --all-namespaces -o json | \
  python3 -c "
import json,sys
data=json.load(sys.stdin)
restarts=[]
for p in data['items']:
  for cs in p.get('status',{}).get('containerStatuses',[]):
    if cs['restartCount']>5:
      restarts.append((p['metadata']['namespace'],p['metadata']['name'],cs['restartCount']))
restarts.sort(key=lambda x:-x[2])
for ns,name,r in restarts[:15]:
  print(f'{r:>5} restarts  {ns}/{name}')
"

# Pending pods (scheduling issues)
kubectl get pods --all-namespaces --field-selector status.phase=Pending
```

### CrashLoopBackOff

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
```

**Common causes**: missing config/secrets, OOM, application errors, image issues.

### ImagePullBackOff

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A5 Events
```

**Common causes**: wrong image name, registry auth missing, network issues to registry.

## 6. Networking

```bash
# Ingress controller pods
kubectl get pods -n openshift-ingress

# Routes with issues
kubectl get routes --all-namespaces | head -20

# DNS resolution test (from a debug pod)
kubectl run dns-test --rm -i --restart=Never --image=busybox -- nslookup kubernetes.default.svc.cluster.local

# Network operator status
kubectl get clusteroperator network
kubectl get pods -n openshift-network-operator

# SDN/OVN pods (depends on network plugin)
kubectl get pods -n openshift-ovn-kubernetes 2>/dev/null || kubectl get pods -n openshift-sdn 2>/dev/null
```

### Service/Route Not Reachable

**Diagnosis**:
```bash
# Check if endpoints exist for the service
kubectl get endpoints <service-name> -n <namespace>

# Check if ingress controller is healthy
kubectl get pods -n openshift-ingress
kubectl logs -n openshift-ingress <router-pod> --tail=20
```

## 7. Certificates

```bash
# Check for expiring certificates
kubectl get certificates --all-namespaces 2>/dev/null

# API server serving cert expiry
kubectl get secret -n openshift-kube-apiserver -o json | \
  python3 -c "import json,sys; [print(i['metadata']['name']) for i in json.load(sys.stdin)['items'] if 'cert' in i['metadata']['name'].lower()]" 2>/dev/null

# Check cluster operator certificate conditions
kubectl get clusteroperator kube-apiserver -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

## 8. MachineConfigPool (Node Updates)

```bash
# MCP status (are nodes updating or degraded?)
kubectl get machineconfigpool

# Detailed MCP status
kubectl describe machineconfigpool worker
kubectl describe machineconfigpool master

# Nodes being updated or stuck
kubectl get nodes -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,MCP:.metadata.annotations.machineconfiguration\.openshift\.io/state'
```

### Nodes Stuck Updating

**Diagnosis**:
```bash
kubectl get machineconfigpool -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Degraded")].status}{"\t"}{.status.conditions[?(@.type=="Degraded")].message}{"\n"}{end}'
```

**Remediation**:
- Check the machine-config-daemon pod on the stuck node
- Review `kubectl logs -n openshift-machine-config-operator machine-config-daemon-<id>` on that node
- A degraded MCP often means a config failed to apply -- fix the MachineConfig or remove it

## 9. Cluster Version and Updates

```bash
# Cluster version and update status
kubectl get clusterversion

# Detailed update progress
kubectl describe clusterversion version

# Check if an update is stuck
kubectl get clusterversion -o jsonpath='{.items[0].status.conditions}' | python3 -m json.tool
```

## 10. Resource Quotas and Limits

```bash
# Resource quotas across namespaces
kubectl get resourcequota --all-namespaces

# LimitRanges
kubectl get limitrange --all-namespaces

# Namespaces near quota limits
kubectl describe resourcequota -n <namespace>
```

## 11. Full Health Report

When the user asks for a cluster health report, run these commands **in parallel** and present the results as a formatted summary with tables:

### Commands to run in parallel

**1. Cluster & nodes:**
```bash
echo "=== Cluster Version ===" && kubectl get clusterversion && echo "" && echo "=== Nodes ===" && kubectl get nodes -o wide && echo "" && echo "=== Degraded Operators ===" && kubectl get clusteroperators | awk 'NR==1 || $3=="False" || $4=="True" || $5=="True"' && echo "" && echo "=== Problem Pods (top 15) ===" && kubectl get pods --all-namespaces | grep -E 'CrashLoopBackOff|Error|ImagePullBackOff|Pending|OOMKilled' | head -15
```

**2. Ceph storage health (if OCS/ODF is installed):**
```bash
kubectl -n openshift-storage get cephcluster -o jsonpath='{.items[*].status.ceph}' 2>/dev/null | python3 -c "
import json,sys
d=json.load(sys.stdin)
cap=d['capacity']
pct=cap['bytesUsed']/cap['bytesTotal']*100
print(f'Health:    {d[\"health\"]}')
print(f'Used:      {cap[\"bytesUsed\"]//2**30} GiB / {cap[\"bytesTotal\"]//2**30} GiB ({pct:.1f}%)')
print(f'Available: {cap[\"bytesAvailable\"]//2**30} GiB')
if d.get('details'):
    for k,v in d['details'].items():
        print(f'  {k}: {v[\"message\"]} ({v[\"severity\"]})')
"
```

**3. Node CPU & memory usage:**
```bash
kubectl top nodes
```

### How to present the report

Format the results as a concise summary with:

- **Cluster Overview** section: version, node count/status, operator health, problem pods
- **Storage** section: Ceph health, capacity used/available/percentage as a table
- **Memory & CPU** section: per-node usage as a table, highlight nodes above 70% memory or 80% CPU

Flag any issues found (degraded operators, NotReady nodes, Ceph warnings/errors, nodes under memory/CPU pressure, pending PVCs, problem pods) with brief remediation hints.

If everything is healthy, say so clearly.
