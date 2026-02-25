# kubevirt-skills

AI agent skills for MTV/Forklift migrations on OpenShift and Kubernetes. Works with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [Cursor](https://www.cursor.com/).

## Included Skills

| Skill | Description |
|-------|-------------|
| **krew-setup** | Ensure kubectl krew, mtv, and virt plugins are installed |
| **kubectl-mtv** | Manage MTV/Forklift VM migrations from vSphere, oVirt, OpenStack, OVA, EC2, or HyperV |
| **kubectl-virt** | Create, start, stop, and manage KubeVirt virtual machines |
| **troubleshoot-virt** | Troubleshoot stuck VMs, DataVolumes, and migrations |

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
for skill in krew-setup kubectl-mtv kubectl-virt troubleshoot-virt; do
  rm -f ~/.claude/skills/"$skill"
done
```

**Per-project:**

```bash
# From inside the target project directory
for skill in krew-setup kubectl-mtv kubectl-virt troubleshoot-virt; do
  rm -f .claude/skills/"$skill"
done
```

### Cursor

**User-wide:**

```bash
for skill in krew-setup kubectl-mtv kubectl-virt troubleshoot-virt; do
  rm -f ~/.cursor/skills/"$skill"
done
```

**Per-project:**

```bash
# From inside the target project directory
for skill in krew-setup kubectl-mtv kubectl-virt troubleshoot-virt; do
  rm -f .cursor/skills/"$skill"
done
```

## License

[Apache-2.0](LICENSE)
