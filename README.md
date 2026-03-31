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

## Quick Start

```bash
git clone https://github.com/yaacov/kubevirt-skills.git
cd kubevirt-skills

# Symlink skills (Cursor — user-wide)
mkdir -p ~/.cursor/skills
for skill in skills/*/; do
  ln -sfn "$(pwd)/$skill" ~/.cursor/skills/"$(basename "$skill")"
done
```

For Claude Code setup, per-project installs, and removal see [docs/install.md](docs/install.md).

## Included Skills

| Skill | Description |
|-------|-------------|
| **check-ceph-health** | Check Ceph storage health on OpenShift OCS/ODF clusters |
| **check-ocp-health** | General OpenShift (OCP) cluster health check |
| **kubectl-mtv** | Manage MTV/Forklift VM migrations from vSphere, oVirt, OpenStack, OVA, EC2, or HyperV |
| **kubectl-virt** | Create, start, stop, and manage KubeVirt virtual machines |
| **mcp-setup** | Install and configure MCP servers (kubectl-mtv, kubectl-metrics, kubectl-debug-queries) |
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
