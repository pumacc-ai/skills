---
name: Server Setup
slug: server-setup
version: "1.0.0"
description: Provision a new server using the setup.yaml Ansible playbook. Handles user creation, packages, Node.js, Claude Code, swap, fail2ban, and profile vars from Bitwarden.
---

# Server Setup Skill

Use `./run setup <host>` from the repo root to provision a fresh server.

## Quick Start

```bash
# Provision a specific host (must exist in inventory.yaml)
./run setup coder.pumacc.com

# Provision all hosts in the default group (agents)
./run setup
```

## What setup.yaml Does

1. **Service user** — creates `bands` user (or `bands_user` override) with passwordless sudo
2. **Authorized keys** — deploys `../authorized_keys` to the service user's `~/.ssh/`
3. **Profile vars** — writes env exports to `~/.profile`; values prefixed `bw:<uuid>` are resolved from Bitwarden at run time
4. **APT** — migrates EOL Ubuntu apt sources, full upgrade, installs packages
5. **Packages** — `curl git vim openvpn easy-rsa ufw python3 rsync yq jq`
6. **Swap** — creates a 2 GB swapfile if none exists; sets swappiness=60, vfs_cache_pressure=50
7. **Node.js 22** — via NodeSource repo
8. **pnpm** — installed globally via npm
9. **Bitwarden CLI** (`bw`) — installed globally via npm
10. **Claude Code CLI** (`claude`) — installed globally via npm, available to all users
11. **SSH hardening** — disables root password login
12. **fail2ban** — protects SSH and PostgreSQL; bans 1h after 5 failures in 10 min

## Adding a New Host

Edit `inventory.yaml` and add the host under the appropriate group:

```yaml
all:
  children:
    agents:
      hosts:
        myserver.example.com:
          ansible_host: 1.2.3.4
          profile_vars:
            ANTHROPIC_API_KEY: "bw:9455622e-e759-4cec-ba11-b40b00df693b"
            CLAUDE_MODEL: "claude-sonnet-4-6"
```

Then run:

```bash
./run setup myserver.example.com
```

## profile_vars and Bitwarden

Values in `profile_vars` are written as `export KEY=VALUE` in the remote `~/.profile`.

Prefix a value with `bw:<secret-uuid>` to pull it from Bitwarden Secrets Manager at provision time — the value is resolved server-side during the playbook run and never stored in the inventory file.

```yaml
profile_vars:
  ANTHROPIC_API_KEY: "bw:9455622e-e759-4cec-ba11-b40b00df693b"   # resolved from BWS
  CLAUDE_MODEL: "claude-sonnet-4-6"                                # literal value
```

To find a secret UUID:
```bash
bws secret list --output json | jq -r '.[] | "\(.id)  \(.key)"'
```

## Prerequisites

- `BWS_ACCESS_TOKEN` must be set (it is in `~/.profile`)
- Ansible collections installed: `ansible-galaxy collection install -r playbooks/requirements.yaml`
- SSH access to the target host as `root` (initial provision) or `ansible_user`

## Overriding the Service User

By default the service user is `bands`. Override per-host in inventory:

```yaml
hosts:
  myserver.example.com:
    bands_user: pumacc
```

## Idempotency

The playbook is fully idempotent — safe to re-run on an already-provisioned server to apply updates or correct drift.
