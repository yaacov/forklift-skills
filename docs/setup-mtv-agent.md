# Setting Up the MTV Agent

The [mtv-agent](https://github.com/yaacov/mtv-agent) is a Python-based AI assistant
for MTV/Forklift VM migrations. It provides a chat interface with an LLM tool loop,
connects to MCP servers over SSE, and includes a built-in web UI.

## Prerequisites

- Python 3.11+
- Docker or Podman (for running MCP server containers)
- OpenShift cluster with MTV/Forklift installed
- An OpenAI-compatible LLM backend (e.g. [LM Studio](https://lmstudio.ai/) or Claude via proxy)

## Install

Using pip:

```bash
pip install mtv-agent
```

Or with [uv](https://docs.astral.sh/uv/) (recommended):

```bash
uv tool install mtv-agent
```

Verify the installation:

```bash
mtv-agent --version
```

## Initialize

Create the default configuration directory (`~/.mtv-agent/`) with config files,
skills, and playbooks:

```bash
mtv-agent init
```

This generates:

| File / Directory | Purpose |
|------------------|---------|
| `config.json` | LLM endpoint, server bind address, agent limits, cache |
| `mcp.json` | MCP server definitions (URLs, container images, auth) |
| `skills/` | Markdown skill files the agent can select at runtime |
| `playbooks/` | Multi-step playbooks for common workflows |

To reinitialize or overwrite existing files:

```bash
mtv-agent init --force
```

---

## Configure the LLM Backend

Edit `~/.mtv-agent/config.json` or set environment variables:

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `LLM_BASE_URL` | `http://localhost:1234/v1` | OpenAI-compatible API base URL |
| `LLM_API_KEY` | `lm-studio` | API key for the LLM service |
| `LLM_MODEL` | _(from config)_ | Model name to use |

### Using LM Studio (default)

Start LM Studio, load a model, and enable the local server on port 1234.
No config changes are needed — the defaults point to LM Studio.

### Using Claude via Proxy

Start the agent with the built-in `claude-openai-proxy`:

```bash
mtv-agent start --with-cop
```

This launches a proxy on port 1234 that translates OpenAI-format requests to the
Claude API.

---

## Configure Cluster Access

The agent needs access to your OpenShift cluster. Credentials are resolved in order:

1. CLI flags (`--kube-api-url`, `--kube-token`)
2. Environment variables (`KUBE_API_URL`, `KUBE_TOKEN`)
3. Kubeconfig file (`--kubeconfig`, `--kube-context`, or default `~/.kube/config`)

For token-based access:

```bash
export KUBE_API_URL="https://api.mycluster.example.com:6443"
export KUBE_TOKEN="sha256~..."
```

---

## Start the Agent

The `start` command launches MCP server containers, then starts the API server:

```bash
mtv-agent start
```

This will:

1. Pull and run three MCP server containers (MTV, metrics, debug-queries)
2. Start the FastAPI server with the web UI on `http://localhost:8000`

### Default MCP Containers

| Container | Host Port | Image |
|-----------|-----------|-------|
| kubectl-mtv | 8080 | `quay.io/yaacov/kubectl-mtv-mcp-server:latest` |
| kubectl-metrics | 8081 | `quay.io/yaacov/kubectl-metrics-mcp-server:latest` |
| kubectl-debug-queries | 8082 | `quay.io/yaacov/kubectl-debug-queries-mcp-server:latest` |

### Common Start Options

```bash
# Use Podman instead of Docker
mtv-agent start --runtime podman

# Use Claude via the built-in proxy
mtv-agent start --with-cop

# Bind to a specific host and port
mtv-agent start --host 0.0.0.0 --port 9000

# Skip TLS verification for container image pulls (Podman only)
mtv-agent start --runtime podman --skip-tls

# Use a custom config directory
mtv-agent start --config /path/to/config.json --mcp-config /path/to/mcp.json
```

---

## API-Only Mode

If MCP servers are already running (e.g. started separately or on a remote host),
use `serve` to start only the API server:

```bash
mtv-agent serve
```

Accepts the same `--host`, `--port`, `--config`, and `--kube-*` flags as `start`.

---

## Stop the Agent

Stop all MCP containers and the claude-openai-proxy (if running):

```bash
mtv-agent stop
```

---

## Upgrade

```bash
pip install --upgrade mtv-agent
# or
uv tool upgrade mtv-agent
```

---

## Key Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `LLM_BASE_URL` | `http://localhost:1234/v1` | LLM API endpoint |
| `LLM_API_KEY` | `lm-studio` | LLM API key |
| `LLM_MODEL` | _(config)_ | Model name |
| `SERVER_HOST` | `0.0.0.0` | API server bind address |
| `SERVER_PORT` | `8000` | API server port |
| `KUBE_API_URL` | — | Kubernetes API server URL |
| `KUBE_TOKEN` | — | Kubernetes bearer token |
| `SKILLS_DIR` | `~/.mtv-agent/skills` | Path to skill files |
| `PLAYBOOKS_DIR` | `~/.mtv-agent/playbooks` | Path to playbook files |
| `MEMORY_MAX_TURNS` | _(config)_ | Max conversation turns to retain |
| `MEMORY_TTL_SECONDS` | _(config)_ | Conversation memory TTL |
| `MAX_ITERATIONS` | _(config)_ | Max tool-loop iterations per request |
| `MCP_TOOL_TIMEOUT` | _(config)_ | Timeout for MCP tool calls (seconds) |
| `BASH_TIMEOUT` | _(config)_ | Timeout for bash tool calls (seconds) |

---

## Quick Start Summary

```bash
pip install mtv-agent
mtv-agent init
mtv-agent start
# Open http://localhost:8000
```
