---
name: Bitwarden Secrets
slug: bitwarden
version: "1.1.0"
description: Manage secrets with Bitwarden Secrets Manager (bws). Auto-sync secrets to environment variables and SSH key files on session start. Create and store SSH keypairs.
---

# Bitwarden Secrets Skill

Use `bws` CLI to interact with Bitwarden Secrets Manager.

## Bootstrap

`BWS_ACCESS_TOKEN` must be set before any `bws` command. It lives hardcoded in `~/.profile` — it is the only secret that cannot bootstrap itself.

All other secrets are synced automatically via `sync-bws` on login.

## Key Conventions

| BWS key (lowercase) | Result |
|---------------------|--------|
| `github_token` | `export GITHUB_TOKEN=...` |
| `anthropic_api_key` | `export ANTHROPIC_API_KEY=...` |
| `ssh_pumacc` | `~/.ssh/pumacc` (chmod 700) |
| `ssh_pumacc.pub` | `~/.ssh/pumacc.pub` (chmod 700) |

- Keys starting with `ssh_` are written as files to `~/.ssh/`, stripping the `ssh_` prefix.
- All other keys are exported as uppercase environment variables.
- A variable already set in the environment is **not overwritten** by BWS.

## sync-bws Script

Located at `~/.local/bin/sync-bws`. Called via `eval "$(sync-bws)"` in `~/.profile`.

```bash
# Force a fresh sync (re-export even already-set vars):
eval "$(sync-bws --force)"

# Dry-run (print what would be exported):
sync-bws --dry-run

# SSH keys only:
sync-bws --ssh-only
```

## Creating SSH Keys

Use `create-ssh-key` to generate a keypair, write it to `~/.ssh/`, and store both keys in Bitwarden in one step.

```bash
create-ssh-key <name> [ed25519|rsa]
```

Examples:

```bash
create-ssh-key deploy          # ed25519 by default → ~/.ssh/deploy
create-ssh-key github rsa      # RSA → ~/.ssh/github
```

What it does:
1. Runs `ssh-keygen -t <type> -f ~/.ssh/<name> -N ""` (no passphrase)
2. Sets `chmod 700` on both `~/.ssh/<name>` and `~/.ssh/<name>.pub`
3. Saves private key to BWS as `ssh_<name>`
4. Saves public key to BWS as `ssh_<name>.pub`

On next login (or after `eval "$(sync-bws)"`), the keys will be restored to `~/.ssh/` automatically on any machine.

## Adding a New Secret

```bash
# Create secret in BWS
bws secret create my_service_token "the-value" --project-id <project-id>

# It will be available as MY_SERVICE_TOKEN on next login,
# or immediately via: eval "$(sync-bws)"
```

## Listing Secrets

```bash
bws secret list
bws secret list --output json | jq '.[].key'
```

## Getting a Single Secret

```bash
bws secret get <secret-id>
# or by key (requires listing first):
bws secret list --output json | jq -r '.[] | select(.key=="github_token") | .value'
```
