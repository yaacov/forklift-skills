# Installation

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Cursor](https://www.cursor.com/) installed and configured

## Claude Code Plugin (recommended)

Install as a Claude Code plugin — no cloning or symlinks needed:

```bash
# Add the marketplace and install the plugin
claude plugin marketplace add yaacov/forklift-skills
claude plugin install forklift-skills@forklift-skills
```

Skills appear as `/forklift-skills:<skill-name>` in Claude Code.

To update later:

```bash
claude plugin marketplace update forklift-skills
```

To uninstall:

```bash
claude plugin uninstall forklift-skills@forklift-skills
```

## Claude Code Symlinks (alternative)

If you prefer managing skills as symlinks instead of a plugin:

```bash
git clone https://github.com/yaacov/forklift-skills.git
cd forklift-skills
```

**User-wide** (available in all your projects):

```bash
mkdir -p ~/.claude/skills

for skill in skills/*/; do
  ln -sfn "$(pwd)/$skill" ~/.claude/skills/"$(basename "$skill")"
done
```

**Per-project** (available only in a specific project):

```bash
# From inside the target project directory
mkdir -p .claude/skills

for skill in /path/to/forklift-skills/skills/*/; do
  ln -sfn "$skill" .claude/skills/"$(basename "$skill")"
done
```

## Cursor

```bash
git clone https://github.com/yaacov/forklift-skills.git
cd forklift-skills
```

**User-wide** (available in all your projects):

```bash
mkdir -p ~/.cursor/skills

for skill in skills/*/; do
  ln -sfn "$(pwd)/$skill" ~/.cursor/skills/"$(basename "$skill")"
done
```

**Per-project** (available only in a specific project):

```bash
# From inside the target project directory
mkdir -p .cursor/skills

for skill in /path/to/forklift-skills/skills/*/; do
  ln -sfn "$skill" .cursor/skills/"$(basename "$skill")"
done
```

> **Tip:** Symlinks keep skills up to date — just `git pull` inside the cloned repo to get the latest changes.

## MCP Server Prerequisites

Several skills require MCP tools provided by kubectl plugins. Install them
with the one-line installer (Linux / macOS):

```bash
# kubectl-mtv (MTV/Forklift migrations)
curl -sSL https://raw.githubusercontent.com/yaacov/kubectl-mtv/main/install.sh | bash

# kubectl-metrics (Prometheus/Thanos metrics)
curl -sSL https://raw.githubusercontent.com/yaacov/kubectl-metrics/main/install.sh | bash

# kubectl-debug-queries (Kubernetes resources, logs, events)
curl -sSL https://raw.githubusercontent.com/yaacov/kubectl-debug-queries/main/install.sh | bash
```

Or install with [krew](https://krew.sigs.k8s.io/):

```bash
kubectl krew install mtv
kubectl krew install debug-queries
```

All installers place binaries in `~/.local/bin` by default. If that directory
is not in your PATH:

```bash
# bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# zsh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

The **mcp-setup** skill can guide you through installation and MCP server
configuration interactively. Just ask your agent: *"Set up the MCP servers
so I can use these tools."*

## Removal

### Claude Code Plugin

```bash
claude plugin uninstall forklift-skills@forklift-skills
```

### Claude Code Symlinks

**User-wide:**

```bash
for skill in check-ceph-health check-ocp-health govc-vsphere kubectl-mtv kubectl-virt mcp-setup observe-metrics troubleshoot-virt; do
  rm -f ~/.claude/skills/"$skill"
done
```

**Per-project:**

```bash
# From inside the target project directory
for skill in check-ceph-health check-ocp-health govc-vsphere kubectl-mtv kubectl-virt mcp-setup observe-metrics troubleshoot-virt; do
  rm -f .claude/skills/"$skill"
done
```

### Cursor

**User-wide:**

```bash
for skill in check-ceph-health check-ocp-health govc-vsphere kubectl-mtv kubectl-virt mcp-setup observe-metrics troubleshoot-virt; do
  rm -f ~/.cursor/skills/"$skill"
done
```

**Per-project:**

```bash
# From inside the target project directory
for skill in check-ceph-health check-ocp-health govc-vsphere kubectl-mtv kubectl-virt mcp-setup observe-metrics troubleshoot-virt; do
  rm -f .cursor/skills/"$skill"
done
```
