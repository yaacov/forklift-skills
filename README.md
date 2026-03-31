# forklift-skills

AI agent skills for MTV/Forklift migrations on OpenShift and Kubernetes. Works with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [Cursor](https://www.cursor.com/).

## Included Skills

| Skill | Description |
|-------|-------------|
| **check-ceph-health** | Check Ceph storage health on OpenShift OCS/ODF clusters |
| **check-ocp-health** | General OpenShift (OCP) cluster health check |
| **kubectl-mtv** | Manage MTV/Forklift VM migrations from vSphere, oVirt, OpenStack, OVA, EC2, or HyperV |
| **kubectl-virt** | Create, start, stop, and manage KubeVirt virtual machines |
| **mcp-setup** | Install and configure MCP servers (kubectl-mtv, kubectl-metrics, kubectl-debug-queries) |
| **observe-metrics** | Observe cluster metrics via Prometheus/Thanos (discovery, presets, PromQL) |
| **troubleshoot-virt** | Troubleshoot stuck VMs, DataVolumes, and migrations |

## Docs and Scripts

| Path | Description |
|------|-------------|
| **docs/setup-mtv-agent.md** | Setting up the [mtv-agent](https://github.com/yaacov/mtv-agent) AI assistant (install, configure, run) |
| **docs/create-providers-cli.md** | Step-by-step guide for creating MTV source providers (vSphere, oVirt, OpenStack, OVA) using `oc mtv` |
| **scripts/create-providers.sh** | Script that creates providers automatically from environment variables |

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Cursor](https://www.cursor.com/) installed and configured

## Installation

First, clone the repository:

```bash
git clone https://github.com/yaacov/kubevirt-skills.git
cd kubevirt-skills
```

### Claude Code

**User-wide** (available in all your projects):

```bash
# Create the skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# Symlink each skill
for skill in skills/*/; do
  ln -sfn "$(pwd)/$skill" ~/.claude/skills/"$(basename "$skill")"
done
```

**Per-project** (available only in a specific project):

```bash
# From inside the target project directory
mkdir -p .claude/skills

for skill in /path/to/kubevirt-skills/skills/*/; do
  ln -sfn "$skill" .claude/skills/"$(basename "$skill")"
done
```

### Cursor

**User-wide** (available in all your projects):

```bash
# Create the skills directory if it doesn't exist
mkdir -p ~/.cursor/skills

# Symlink each skill
for skill in skills/*/; do
  ln -sfn "$(pwd)/$skill" ~/.cursor/skills/"$(basename "$skill")"
done
```

**Per-project** (available only in a specific project):

```bash
# From inside the target project directory
mkdir -p .cursor/skills

for skill in /path/to/kubevirt-skills/skills/*/; do
  ln -sfn "$skill" .cursor/skills/"$(basename "$skill")"
done
```

> **Tip:** Symlinks keep skills up to date — just `git pull` inside the cloned repo to get the latest changes.

## Removal

### Claude Code

**User-wide:**

```bash
for skill in check-ceph-health check-ocp-health kubectl-mtv kubectl-virt mcp-setup observe-metrics troubleshoot-virt; do
  rm -f ~/.claude/skills/"$skill"
done
```

**Per-project:**

```bash
# From inside the target project directory
for skill in check-ceph-health check-ocp-health kubectl-mtv kubectl-virt mcp-setup observe-metrics troubleshoot-virt; do
  rm -f .claude/skills/"$skill"
done
```

### Cursor

**User-wide:**

```bash
for skill in check-ceph-health check-ocp-health kubectl-mtv kubectl-virt mcp-setup observe-metrics troubleshoot-virt; do
  rm -f ~/.cursor/skills/"$skill"
done
```

**Per-project:**

```bash
# From inside the target project directory
for skill in check-ceph-health check-ocp-health kubectl-mtv kubectl-virt mcp-setup observe-metrics troubleshoot-virt; do
  rm -f .cursor/skills/"$skill"
done
```

## License

[Apache-2.0](LICENSE)
