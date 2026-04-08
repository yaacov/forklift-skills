---
name: mcp-setup
description: Install and configure the MCP servers for Forklift/MTV, Prometheus metrics, and Kubernetes debug queries. Use when MCP tools (metrics_read, mtv_read, debug_read) are not available, or when the user wants to set up AI agent integration.
---

# MCP Server Setup

This plugin provides skills only — MCP servers are **not** bundled and must be
installed and configured separately. This skill walks the user through both steps.

## What to Check

First, check if the kubectl plugins are already installed:

```bash
kubectl metrics --help 2>/dev/null && echo "METRICS_OK" || echo "METRICS_MISSING"
kubectl mtv --help 2>/dev/null && echo "MTV_OK" || echo "MTV_MISSING"
kubectl debug-queries --help 2>/dev/null && echo "DEBUG_OK" || echo "DEBUG_MISSING"
```

Then check if MCP servers are already configured by trying a simple call to
each MCP tool. If the tools respond, skip to "Tool Summary."

## How to Respond

Based on the results above, tell the user **only** what is missing and provide
the relevant install and configuration instructions. If everything is already
installed and configured, confirm it and move on.

### Install kubectl-mtv (MTV/Forklift migrations)

Quick install (Linux / macOS):

```bash
curl -sSL https://raw.githubusercontent.com/yaacov/kubectl-mtv/main/install.sh | bash
```

Or with [krew](https://krew.sigs.k8s.io/):

```bash
kubectl krew install mtv
```

Verify: `kubectl mtv --help`

### Install kubectl-metrics (Prometheus/Thanos metrics)

Quick install (Linux / macOS):

```bash
curl -sSL https://raw.githubusercontent.com/yaacov/kubectl-metrics/main/install.sh | bash
```

Verify: `kubectl metrics --help`

### Install kubectl-debug-queries (Kubernetes resources, logs, events)

Quick install (Linux / macOS):

```bash
curl -sSL https://raw.githubusercontent.com/yaacov/kubectl-debug-queries/main/install.sh | bash
```

Or with [krew](https://krew.sigs.k8s.io/):

```bash
kubectl krew install debug-queries
```

Verify: `kubectl debug-queries --help`

### PATH setup

All three install to `~/.local/bin` by default. If it is not in the user's PATH:

```bash
# bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# zsh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

## MCP Server Configuration

After installing the binaries, configure each as an MCP server. The plugin
does not include MCP server configs, so this step is required for the skills
that use MCP tools (`mtv_read`, `debug_read`, `metrics_read`).

### Claude Code (CLI)

If the user installed this plugin via Claude Code, run:

```bash
claude mcp add kubectl-metrics -- kubectl metrics mcp-server
claude mcp add kubectl-mtv -- kubectl mtv mcp-server
claude mcp add kubectl-debug-queries -- kubectl debug-queries mcp-server
```

After adding, restart the session or run `/mcp` to verify the servers appear.

### Cursor IDE

Settings -> MCP -> Add Server for each:

| Name | Command | Args |
|------|---------|------|
| kubectl-metrics | `kubectl` | `metrics mcp-server` |
| kubectl-mtv | `kubectl` | `mtv mcp-server` |
| kubectl-debug-queries | `kubectl` | `debug-queries mcp-server` |

### Claude Desktop

Edit `claude_desktop_config.json` and add to the `mcpServers` section:

```json
{
  "mcpServers": {
    "kubectl-metrics": {
      "command": "kubectl",
      "args": ["metrics", "mcp-server"]
    },
    "kubectl-mtv": {
      "command": "kubectl",
      "args": ["mtv", "mcp-server"]
    },
    "kubectl-debug-queries": {
      "command": "kubectl",
      "args": ["debug-queries", "mcp-server"]
    }
  }
}
```

### SSE Mode (OpenShift Lightspeed or Remote Agents)

For remote or server-based agents, run each MCP server in SSE mode:

```bash
kubectl metrics mcp-server --sse --port 8080
kubectl mtv mcp-server --sse --port 8081
kubectl debug-queries mcp-server --sse --port 8082
```

### Container Images (no local binary needed)

Run MCP servers as containers instead of installing kubectl plugins locally.
Requires Docker or Podman and a valid cluster token:

```bash
# kubectl-mtv
docker run --rm -p 8080:8080 \
  -e MCP_KUBE_SERVER=https://api.cluster.example.com:6443 \
  -e MCP_KUBE_TOKEN=sha256~xxxx \
  quay.io/yaacov/kubectl-mtv-mcp-server:latest

# kubectl-metrics
docker run --rm -p 8081:8080 \
  -e MCP_KUBE_SERVER=https://api.cluster.example.com:6443 \
  -e MCP_KUBE_TOKEN=sha256~xxxx \
  quay.io/yaacov/kubectl-metrics-mcp-server:latest

# kubectl-debug-queries
docker run --rm -p 8082:8080 \
  -e MCP_KUBE_SERVER=https://api.cluster.example.com:6443 \
  -e MCP_KUBE_TOKEN=sha256~xxxx \
  quay.io/yaacov/kubectl-debug-queries-mcp-server:latest
```

Then configure your agent to connect via SSE at `http://localhost:8080/sse`,
`http://localhost:8081/sse`, and `http://localhost:8082/sse`.

### Deploy on OpenShift

Deploy the MCP servers directly on the cluster:

```bash
# kubectl-mtv
oc apply -f https://raw.githubusercontent.com/yaacov/kubectl-mtv/main/deploy/mcp-server.yaml

# kubectl-debug-queries
oc apply -f https://raw.githubusercontent.com/yaacov/kubectl-debug-queries/main/deploy/mcp-server.yaml
```

## Tool Summary

| MCP Server | MCP Tools | What It Does |
|------------|-----------|--------------|
| kubectl-metrics | `metrics_read`, `metrics_help` | Query Prometheus/Thanos metrics, discover metrics, instant and range queries |
| kubectl-mtv | `mtv_read`, `mtv_write`, `mtv_help` | Manage MTV/Forklift migrations: providers, plans, inventory, health |
| kubectl-debug-queries | `debug_read`, `debug_help` | List/get Kubernetes resources, pod logs, events with TSL filtering |

Note: `oc metrics`, `oc mtv`, and `oc debug-queries` are aliases for the kubectl versions on OpenShift clusters.
