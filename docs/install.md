# Installation

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Cursor](https://www.cursor.com/) installed and configured

## Clone the Repository

```bash
git clone https://github.com/yaacov/kubevirt-skills.git
cd kubevirt-skills
```

## Claude Code

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

for skill in /path/to/kubevirt-skills/skills/*/; do
  ln -sfn "$skill" .claude/skills/"$(basename "$skill")"
done
```

## Cursor

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
