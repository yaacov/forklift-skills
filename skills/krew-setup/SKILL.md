---
name: krew-setup
description: Ensure kubectl krew, mtv, and virt plugins are installed. Use this skill when you need to run kubectl mtv or kubectl virt commands and want to verify the tools are available first.
---

# Krew and Plugin Setup

Before using kubectl mtv or kubectl virt commands, ensure the prerequisites are installed.
Instead of installing automatically, present the user with the instructions below and let them handle the installation themselves.

## What to check

Run these checks silently to determine what is missing:

```bash
kubectl krew version 2>/dev/null && echo "KREW_OK" || echo "KREW_MISSING"
kubectl krew list 2>/dev/null | grep -q '^virt$' && echo "VIRT_OK" || echo "VIRT_MISSING"
kubectl krew list 2>/dev/null | grep -q '^mtv$' && echo "MTV_OK" || echo "MTV_MISSING"
```

## How to respond

Based on the results above, tell the user **only** what is missing and provide the relevant instructions. If everything is already installed, confirm that and proceed with the task.

### If krew is missing

Tell the user:

> **kubectl krew** is not installed. To install it, run:
>
> ```bash
> (
>   set -x; cd "$(mktemp -d)" &&
>   OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
>   ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')" &&
>   KREW="krew-${OS}_${ARCH}" &&
>   curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
>   tar zxvf "${KREW}.tar.gz" &&
>   ./"${KREW}" install krew
> )
> ```
>
> Then add krew to your PATH by adding this line to your shell profile (~/.zshrc or ~/.bashrc):
>
> ```bash
> export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
> ```
>
> After that, restart your shell or run `source ~/.zshrc`.

### If the virt plugin is missing

Tell the user:

> **kubectl virt** plugin is not installed. To install it, run:
>
> ```bash
> kubectl krew install virt
> ```

### If the mtv plugin is missing

Tell the user:

> **kubectl mtv** plugin is not installed. To install it, run:
>
> ```bash
> kubectl krew install mtv
> ```

### Verification

After the user confirms they have installed the missing components, suggest they verify with:

```bash
kubectl virt --help
kubectl mtv --help
```

Note: `oc virt` and `oc mtv` are aliases for `kubectl virt` and `kubectl mtv` on OpenShift clusters.
