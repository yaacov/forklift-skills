# govc Installation

Installation methods, PATH setup, and verification for `govc`.

See the main [SKILL.md](SKILL.md) for connection setup, VM workflows, and common operations.

---

## Via Homebrew (macOS / Linux)

```bash
brew install govc
```

## Manual binary download

Download from the [govmomi releases](https://github.com/vmware/govmomi/releases) page:

```bash
# Detect OS and architecture
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')

# Fetch the latest release tag
VERSION=$(curl -s https://api.github.com/repos/vmware/govmomi/releases/latest | grep '"tag_name"' | cut -d'"' -f4)

# Download and install
curl -fSL -o govc.gz \
  "https://github.com/vmware/govmomi/releases/download/${VERSION}/govc_${OS}_${ARCH}.gz"
gunzip govc.gz
mkdir -p ~/.local/bin
install -m 0755 govc ~/.local/bin/govc
rm -f govc
```

Ensure `~/.local/bin` is in your PATH:

```bash
# bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# zsh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

## Via Go install

If you have Go installed:

```bash
go install github.com/vmware/govmomi/govc@latest
```

The binary is placed in `$GOPATH/bin` (or `~/go/bin` by default).

## Verify

```bash
govc version
```
