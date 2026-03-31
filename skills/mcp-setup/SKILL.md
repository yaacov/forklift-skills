---
name: mcp-setup
description: Install and configure the MCP servers for Forklift/MTV, Prometheus metrics, and Kubernetes debug queries. Use when MCP tools (metrics_read, mtv_read, debug_read) are not available, or when the user wants to set up AI agent integration.
---

# MCP Server Setup

Install the kubectl plugins that provide MCP (Model Context Protocol) servers for AI assistants, and configure them for your agent.

## What to Check

Run these checks silently to determine what is missing:

```bash
kubectl metrics --help 2>/dev/null && echo "METRICS_OK" || echo "METRICS_MISSING"
kubectl mtv --help 2>/dev/null && echo "MTV_OK" || echo "MTV_MISSING"
kubectl debug-queries --help 2>/dev/null && echo "DEBUG_OK" || echo "DEBUG_MISSING"
```

## How to Respond

Based on the results above, tell the user **only** what is missing and provide the relevant install instructions. If everything is already installed, move on to MCP configuration for their agent.

### Install kubectl-metrics (Prometheus/Thanos metrics)

```bash
curl -sSL https://raw.githubusercontent.com/yaacov/kubectl-metrics/main/install.sh | bash
```

Verify: `kubectl metrics --help`

### Install kubectl-mtv (MTV/Forklift migrations)

```bash
curl -sSL https://raw.githubusercontent.com/yaacov/kubectl-mtv/main/install.sh | bash
```

Verify: `kubectl mtv --help`

### Install kubectl-debug-queries (Kubernetes resources, logs, events)

```bash
curl -sSL https://raw.githubusercontent.com/yaacov/kubectl-debug-queries/main/install.sh | bash
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

After installing the binaries, configure each as an MCP server for the user's AI agent. Ask the user which agent they use and provide the matching instructions.

### Cursor IDE

Settings -> MCP -> Add Server for each:

| Name | Command | Args |
|------|---------|------|
| kubectl-metrics | `kubectl` | `metrics mcp-server` |
| kubectl-mtv | `kubectl` | `mtv mcp-server` |
| kubectl-debug-queries | `kubectl` | `debug-queries mcp-server` |

### Claude Code (CLI)

```bash
claude mcp add kubectl-metrics kubectl metrics mcp-server
claude mcp add kubectl-mtv kubectl mtv mcp-server
claude mcp add kubectl-debug-queries kubectl debug-queries mcp-server
```

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

## Tool Summary

| MCP Server | MCP Tools | What It Does |
|------------|-----------|--------------|
| kubectl-metrics | `metrics_read`, `metrics_help` | Query Prometheus/Thanos metrics, discover metrics, instant and range queries |
| kubectl-mtv | `mtv_read`, `mtv_write`, `mtv_help` | Manage MTV/Forklift migrations: providers, plans, inventory, health |
| kubectl-debug-queries | `debug_read`, `debug_help` | List/get Kubernetes resources, pod logs, events with TSL filtering |

Note: `oc metrics`, `oc mtv`, and `oc debug-queries` are aliases for the kubectl versions on OpenShift clusters.
