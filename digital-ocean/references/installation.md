# Installing and authenticating doctl

## Detect

```bash
doctl version
```

If the binary is missing, pick an install path below based on the platform.

## Install

### Linux (recommended: official tarball)

```bash
cd /tmp
LATEST=$(curl -s https://api.github.com/repos/digitalocean/doctl/releases/latest | jq -r '.tag_name' | sed 's/^v//')
curl -fsSL "https://github.com/digitalocean/doctl/releases/download/v${LATEST}/doctl-${LATEST}-linux-amd64.tar.gz" -o doctl.tar.gz
tar xf doctl.tar.gz
sudo mv doctl /usr/local/bin
doctl version
```

For arm64, replace `linux-amd64` with `linux-arm64`.

### Linux (snap)

```bash
sudo snap install doctl
# Snap confinement: grant access to required interfaces if you'll use SSH/k8s/dotfiles
sudo snap connect doctl:ssh-keys
sudo snap connect doctl:dot-kube :home
```

### macOS

```bash
brew install doctl
```

### Windows

```powershell
# Scoop
scoop install doctl
# or Chocolatey
choco install doctl
```

Or download the `.zip` from the GitHub releases page and place `doctl.exe` on PATH.

## Authenticate

1. Generate a Personal Access Token: https://cloud.digitalocean.com/account/api/tokens. Give it Read + Write scope unless the user wants read-only. Tokens look like `dop_v1_...`.

2. Initialize:

   ```bash
   doctl auth init
   # paste token when prompted
   ```

   For named contexts (multi-account):

   ```bash
   doctl auth init --context personal
   doctl auth init --context work
   doctl auth switch --context work
   doctl auth list   # shows all contexts, the active one starred
   ```

3. Verify:

   ```bash
   doctl account get
   ```

## Non-interactive auth (CI / scripts)

Set the token via environment variable — `doctl` reads `DIGITALOCEAN_ACCESS_TOKEN` automatically:

```bash
export DIGITALOCEAN_ACCESS_TOKEN=dop_v1_xxx
doctl account get
```

This bypasses the config file and is the right pattern for ephemeral environments.

## Config file

`doctl` stores contexts at `~/.config/doctl/config.yaml`. Don't print this file — it contains tokens. To revoke a token, delete it from the DigitalOcean control panel; removing it from the config file alone does not revoke it server-side.

## Shell completion

```bash
# bash
doctl completion bash | sudo tee /etc/bash_completion.d/doctl >/dev/null
# zsh
doctl completion zsh > "${fpath[1]}/_doctl"
# fish
doctl completion fish | source
```

## Upgrade

Tarball install: re-run the install snippet above. Snap: `sudo snap refresh doctl`. Brew: `brew upgrade doctl`.
