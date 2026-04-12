# forklift-skills

AI agent skills for MTV/Forklift migrations on OpenShift and Kubernetes. Works with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [Cursor](https://www.cursor.com/).

## What Can I Do with These Skills?

Just open a chat and ask. Here's one high-impact example per skill:

| Ask the agent to… | Skill used |
|--------------------|------------|
| *"Migrate my 20 VMs from vSphere to OpenShift"* | **kubectl-mtv** |
| *"Check why my cluster nodes are NotReady"* | **check-ocp-health** |
| *"My VM won't start — figure out what's wrong"* | **troubleshoot-virt** |
| *"Show me network traffic by namespace for the last hour"* | **observe-metrics** |
| *"Plot the forklift namespace RX/TX for the last 24h in a chart"* | **observe-metrics** |
| *"Create a Fedora VM with 4 GiB RAM and start it"* | **kubectl-virt** |
| *"Is Ceph healthy? Any OSDs near full?"* | **check-ceph-health** |
| *"Set up the MCP servers so I can use these tools"* | **mcp-setup** |
| *"Write a verification script for MTV-4911"* | **mtv-verify-script** |

## Quick Start

### Claude Code (plugin install)

```bash
# Add the marketplace and install the plugin
claude plugin marketplace add yaacov/forklift-skills
claude plugin install forklift-skills@forklift-skills
```

Or test locally from a cloned repo:

```bash
claude --plugin-dir ./forklift-skills
```

### Cursor (symlink install)

```bash
git clone https://github.com/yaacov/forklift-skills.git
cd forklift-skills

mkdir -p ~/.cursor/skills
for skill in skills/*/; do
  ln -sfn "$(pwd)/$skill" ~/.cursor/skills/"$(basename "$skill")"
done
```

For Claude Code per-project installs, Cursor per-project installs, and removal see [docs/install.md](docs/install.md).

## Prerequisites

Several skills use MCP tools (`mtv_read`, `debug_read`, `metrics_read`) that require
kubectl plugins installed on your machine. The **mcp-setup** skill can guide you through
installation, or see [docs/install.md](docs/install.md) for manual steps.

| kubectl plugin | MCP tools provided |
|---|---|
| [kubectl-mtv](https://github.com/yaacov/kubectl-mtv) | `mtv_read`, `mtv_write`, `mtv_help` |
| [kubectl-debug-queries](https://github.com/yaacov/kubectl-debug-queries) | `debug_read`, `debug_help` |
| [kubectl-metrics](https://github.com/yaacov/kubectl-metrics) | `metrics_read`, `metrics_help` |

Skills that do not require MCP tools (**govc-vsphere**, **kubectl-virt**) work without any prerequisites.

## Included Skills

| Skill | Description |
|-------|-------------|
| **check-ceph-health** | Check Ceph storage health on OpenShift OCS/ODF clusters |
| **check-ocp-health** | General OpenShift (OCP) cluster health check |
| **govc-vsphere** | Manage VMware vSphere VMs using the govc CLI |
| **kubectl-mtv** | Manage MTV/Forklift VM migrations from vSphere, oVirt, OpenStack, OVA, EC2, or HyperV |
| **kubectl-virt** | Create, start, stop, and manage KubeVirt virtual machines |
| **mcp-setup** | Install and configure MCP servers (kubectl-mtv, kubectl-metrics, kubectl-debug-queries) |
| **mtv-verify-script** | Generate and run a self-contained bash e2e verification script for an MTV/Forklift Jira ticket |
| **observe-metrics** | Observe cluster metrics via Prometheus/Thanos (discovery, instant and range queries, PromQL) |
| **troubleshoot-virt** | Troubleshoot stuck VMs, DataVolumes, and migrations |

## Docs

| Path | Description |
|------|-------------|
| **[docs/install.md](docs/install.md)** | Full installation and removal instructions (Claude Code & Cursor) |
| **[docs/setup-mtv-agent.md](docs/setup-mtv-agent.md)** | Setting up the [mtv-agent](https://github.com/yaacov/mtv-agent) AI assistant |
| **[docs/create-providers-cli.md](docs/create-providers-cli.md)** | Creating MTV source providers using `oc mtv` |
| **[scripts/create-providers.sh](scripts/create-providers.sh)** | Script that creates providers from environment variables |

## License

[Apache-2.0](LICENSE)
