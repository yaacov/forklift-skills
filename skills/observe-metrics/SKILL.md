---
name: observe-metrics
description: Observe cluster metrics via Prometheus/Thanos. Use when the user wants to check cluster metrics, monitor network traffic, storage I/O, pod resource usage, VM migration throughput, or discover available Prometheus metrics. Covers metric discovery, storage (Ceph/ODF), network traffic by namespace/pod, pod statistics, and Forklift/MTV migration monitoring.
---

# Observe Cluster Metrics

Use this guide to discover and query Prometheus/Thanos metrics on an OpenShift cluster.

For detailed per-domain queries, labels, and metrics tables see the `observe-metrics-reference` skill.

## Step 1: Set Up Access

Before querying, find the metrics route and get a token:

```bash
# Find Thanos Querier route (preferred -- aggregates all Prometheus data)
THANOS_URL=$(kubectl get route thanos-querier -n openshift-monitoring -o jsonpath='{.status.ingress[0].host}' 2>/dev/null)

# Fallback: try Prometheus route directly
if [ -z "$THANOS_URL" ]; then
  THANOS_URL=$(kubectl get route prometheus-k8s -n openshift-monitoring -o jsonpath='{.status.ingress[0].host}' 2>/dev/null)
fi

echo "Metrics endpoint: https://$THANOS_URL"
```

```bash
TOKEN=$(oc create token prometheus-k8s -n openshift-monitoring)
```

Verify connectivity:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/status/runtimeinfo"
```

## Step 2: Discover Available Metrics

### List all metric names

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/label/__name__/values"
```

### List all available labels

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/labels"
```

## How to Query

All queries use `--data-urlencode` so PromQL special characters are handled correctly.

### Instant query

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query" \
  --data-urlencode 'query=YOUR_PROMQL_HERE'
```

### Range query (last 1 hour, 1-minute steps)

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/query_range" \
  --data-urlencode 'query=YOUR_PROMQL_HERE' \
  --data-urlencode "start=$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --data-urlencode "end=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --data-urlencode 'step=60s'
```

### Prometheus API response format

All queries return JSON in this structure:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": { "label_name": "label_value", ... },
        "value": [ 1234567890.123, "42.5" ]
      }
    ]
  }
}
```

Each result entry has a `metric` map (labels) and a `value` array where `[0]` is the Unix timestamp and `[1]` is the value as a string.

---

## PromQL Quick Reference

### Selecting metrics

```
metric_name                          # all time series for this metric
metric_name{label="value"}           # filter by exact label match
metric_name{label=~"pattern.*"}      # filter by regex match
metric_name{label!="value"}          # exclude a label value
metric_name{l1="a", l2="b"}         # combine multiple filters
```

### Rate and increase (for counters)

Counters only go up. Use `rate` or `increase` to get meaningful values:

```
rate(metric[5m])                     # per-second rate over 5 minutes
increase(metric[1h])                 # total increase over 1 hour
```

### Aggregation

```
sum(metric)                          # total across all series
sum by (label)(metric)               # total grouped by label
avg by (label)(metric)               # average grouped by label
count by (label)(metric)             # count of series grouped by label
min by (label)(metric)               # minimum grouped by label
max by (label)(metric)               # maximum grouped by label
```

### Sorting and limiting

```
topk(10, metric)                     # top 10 series by value
bottomk(5, metric)                   # bottom 5 series by value
sort_desc(metric)                    # sort descending
```

### Arithmetic

```
metric_a / metric_b                  # ratio of two metrics
metric * 100                         # scale a metric
1 - (available / total)              # compute used percentage
```

### Combining techniques

These patterns appear throughout this guide:

```
# Per-second receive rate grouped by namespace, top 10
topk(10, sort_desc(sum by (namespace)(rate(container_network_receive_bytes_total[5m]))))

# Average OSD latency (rate of sum / rate of count)
rate(ceph_osd_op_latency_sum[5m]) / rate(ceph_osd_op_latency_count[5m])

# CPU usage as percentage per node
100 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
```

---

## Formatting Output

Raw Prometheus JSON is verbose. Pipe curl output through formatting tools for readability.

### Pretty-print with python3

```bash
curl -sk ... | python3 -m json.tool
```

### Pretty-print with jq

```bash
curl -sk ... | jq .
```

### Extract just the metric values with jq

```bash
# Show label=value pairs and the metric value
curl -sk ... | jq -r '.data.result[] | "\(.metric) \(.value[1])"'

# Extract a specific label and the value
curl -sk ... | jq -r '.data.result[] | "\(.metric.namespace) \(.value[1])"'

# Filter results where value exceeds a threshold
curl -sk ... | jq -r '.data.result[] | select(.value[1] | tonumber > 0) | "\(.metric.namespace) \(.value[1])"'
```

### Extract metric values with python3

```bash
# Print labels and values as a table
curl -sk ... | python3 -c "
import json,sys
d=json.load(sys.stdin)
if d['status']=='success':
  for r in d['data']['result']:
    labels=', '.join(f'{k}={v}' for k,v in r['metric'].items() if k!='__name__')
    print(f'{labels:60s}  {r[\"value\"][1]}')
"
```

### Search metric names by keyword with jq

```bash
# Replace KEYWORD (e.g., kubevirt, ceph, network, migration)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/label/__name__/values" \
  | jq -r '.data[] | select(test("KEYWORD"; "i"))'
```

### Group metric names by prefix with python3

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS_URL/api/v1/label/__name__/values" | python3 -c "
import json,sys
from collections import Counter
d=json.load(sys.stdin)
prefixes=Counter()
for n in d['data']:
    p=n.split('_')
    prefix=p[0]+'_'+p[1] if len(p)>=2 else p[0]
    prefixes[prefix]+=1
for prefix,count in sorted(prefixes.items(), key=lambda x:-x[1])[:30]:
    print(f'{prefix:45s} {count:4d} metrics')
"
```
