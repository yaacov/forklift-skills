# Network Traffic Metrics

Queries, labels, and metrics for container and node network traffic. See [SKILL.md](SKILL.md) for general usage.

## Network traffic by namespace (presets)

```
metrics_read  command: "preset"  flags: {name: "namespace_network_rx"}
metrics_read  command: "preset"  flags: {name: "namespace_network_tx"}
metrics_read  command: "preset"  flags: {name: "namespace_network_errors"}
```

## Network traffic by namespace (ad-hoc)

```
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(sum by (namespace)(rate(container_network_receive_bytes_total[5m]))))"}
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(sum by (namespace)(rate(container_network_transmit_bytes_total[5m]))))"}
```

## Network traffic by pod in a namespace

Replace `TARGET_NAMESPACE` with the namespace to inspect:

```
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(sum by (pod)(rate(container_network_receive_bytes_total{namespace=\"TARGET_NAMESPACE\"}[5m]))))"}
metrics_read  command: "query"  flags: {query: "topk(10, sort_desc(sum by (pod)(rate(container_network_transmit_bytes_total{namespace=\"TARGET_NAMESPACE\"}[5m]))))"}
```

## Network errors and drops by namespace

```
metrics_read  command: "query"  flags: {query: "topk(10, sum by (namespace)(rate(container_network_receive_errors_total[5m])) + sum by (namespace)(rate(container_network_transmit_errors_total[5m])))"}
```

## Node-level network throughput

```
metrics_read  command: "query"  flags: {query: "instance:node_network_receive_bytes_excluding_lo:rate1m + instance:node_network_transmit_bytes_excluding_lo:rate1m"}
```

## Available labels on network metrics

| Label | Description | Example values |
|-------|-------------|----------------|
| `namespace` | Pod namespace | `openshift-storage`, `konveyor-forklift` |
| `pod` | Pod name | `forklift-controller-6df77f6bf5-jtt7q` |
| `interface` | Network interface (per-pod metrics) | `eth0` |
| `instance` | Node instance (node-level metrics) | `10.0.0.5:9100` |
| `node` | Node name (node-level metrics) | `worker-0` |

## Metrics reference

| Metric | Description |
|--------|-------------|
| `container_network_receive_bytes_total` | Bytes received per pod/namespace |
| `container_network_transmit_bytes_total` | Bytes transmitted per pod/namespace |
| `container_network_receive_errors_total` | Receive errors per pod/namespace |
| `container_network_transmit_errors_total` | Transmit errors per pod/namespace |
| `container_network_receive_packets_dropped_total` | Dropped receive packets |
| `container_network_transmit_packets_dropped_total` | Dropped transmit packets |
| `node_network_receive_bytes_total` | Bytes received per node/interface |
| `node_network_transmit_bytes_total` | Bytes transmitted per node/interface |
| `instance:node_network_receive_bytes_excluding_lo:rate1m` | Pre-computed node receive rate |
