---
name: Agent Setup
slug: agent-setup
version: "1.1.0"
description: Provision a remote OpenClaw agent node from scratch. Covers SSH keys, git identity, home repo, openclaw install, systemd service, extensions, and cron sync. Auto-discovers hosts from inventory.yaml. No Ansible required.
---

# Agent Setup Skill

Fully provisions a remote OpenClaw agent node over SSH.
All commands run as `$AGENT_USER` on the remote server unless marked `(sudo)`.

## Inventory Detection

If `inventory.yaml` exists in the current directory, use it to discover and configure target hosts automatically.

### Inventory Format

Two `bw_secret_*` patterns are supported for resolving secrets from Bitwarden:

| Pattern | Example | Resolved via |
|---------|---------|--------------|
| `bw_secret_<var>: <key-name>` | `bw_secret_anthropic_api_key: dan_anthropic_api_key` | BWS key looked up by **name** |
| `profile_vars` value `bw:<uuid>` | `ANTHROPIC_API_KEY: "bw:9455622e-..."` | BWS secret looked up by **UUID** |

Example inventory (`farafon/inventory.yaml` style):

```yaml
all:
  vars:
    project: farafon
  children:
    agents:
      hosts:
        dan.farafonov.info:
          ansible_host: 178.128.132.23
          ansible_user: dan
          service_user: dan
          web_title: "Daniel Farafonov"
          ui_port: 6767
          openclaw_host: true
          openclaw_model: "anthropic/claude-sonnet-4-6"
          bw_secret_anthropic_api_key: dan_anthropic_api_key   # resolved from BWS by key name
          anthropic_api_key: ""                                 # filled in at provision time
          git_user_name: "Dan Farafon"
          git_user_email: "dan@farafon.com"
          git_memory: "git@github.com:pumacc-ai/farafon-dan.git"
          git_repos: []
          openclaw_extensions: []
          openclaw_skills: []
          discord:
            enabled: true
            bw_secret_token: dan_discord_bot_token
            dm_policy: pairing
            group_policy: open
```

### Parse All Hosts

```bash
python3 - << 'EOF'
import yaml, json, subprocess, sys

def bws_get_by_name(key_name):
    """Resolve a BWS secret value by key name."""
    try:
        out = subprocess.check_output(
            ['bws', 'secret', 'list', '--output', 'json'],
            stderr=subprocess.DEVNULL
        )
        secrets = json.loads(out)
        for s in secrets:
            if s['key'] == key_name:
                return s['value']
    except Exception:
        pass
    return ''

def bws_get_by_uuid(uuid):
    """Resolve a BWS secret value by UUID."""
    try:
        out = subprocess.check_output(
            ['bws', 'secret', 'get', uuid, '--output', 'json'],
            stderr=subprocess.DEVNULL
        )
        return json.loads(out)['value']
    except Exception:
        return ''

with open('inventory.yaml') as f:
    inv = yaml.safe_load(f)

all_vars = inv.get('all', {}).get('vars', {})
results  = []

for group_name, group in inv.get('all', {}).get('children', {}).items():
    group_vars = group.get('vars', {})
    for hostname, host_vars in (group.get('hosts') or {}).items():
        hv = host_vars or {}

        # Resolve bw_secret_<field>: <key-name> entries
        resolved = dict(hv)
        for k, v in hv.items():
            if k.startswith('bw_secret_'):
                field = k[len('bw_secret_'):]   # e.g. "anthropic_api_key"
                resolved[field] = bws_get_by_name(v)

        # Resolve profile_vars bw:<uuid> values
        profile_vars = {}
        for pk, pv in hv.get('profile_vars', {}).items():
            if isinstance(pv, str) and pv.startswith('bw:'):
                profile_vars[pk] = bws_get_by_uuid(pv[3:])
            else:
                profile_vars[pk] = pv

        results.append({
            'hostname':           hostname,
            'ansible_host':       resolved.get('ansible_host', hostname),
            'agent_user':         resolved.get('ansible_user', group_vars.get('ansible_user', 'root')),
            'service_user':       resolved.get('service_user', resolved.get('ansible_user', 'root')),
            'project':            all_vars.get('project', 'Agent'),
            'web_title':          resolved.get('web_title', hostname),
            'ui_port':            resolved.get('ui_port', 18789),
            'openclaw_host':      resolved.get('openclaw_host', False),
            'openclaw_model':     resolved.get('openclaw_model', 'anthropic/claude-sonnet-4-6'),
            'anthropic_api_key':  resolved.get('anthropic_api_key', ''),
            'git_user_name':      resolved.get('git_user_name', 'PumaCC Agent'),
            'git_user_email':     resolved.get('git_user_email', 'dev@pumacc.com'),
            'git_memory':         resolved.get('git_memory', ''),
            'git_repos':          resolved.get('git_repos', []),
            'profile_vars':       profile_vars,
            'openclaw_extensions':resolved.get('openclaw_extensions', ['whatsapp']),
            'openclaw_skills':    resolved.get('openclaw_skills', []),
            'discord':            resolved.get('discord', {}),
        })

# Mask secrets in output
safe = json.loads(json.dumps(results))
for h in safe:
    if h.get('anthropic_api_key'):
        h['anthropic_api_key'] = '***'
    for k in h.get('profile_vars', {}):
        h['profile_vars'][k] = '***'

print(json.dumps(safe, indent=2))
EOF
```

### Set Variables Per Host

After parsing, set shell variables for the host you are provisioning:

```bash
HOSTNAME="dan.farafonov.info"
AGENT_HOST="178.128.132.23"
AGENT_USER="dan"
SERVICE_USER="dan"
PROJECT="farafon"                  # matches ~/.ssh/<PROJECT> key name
GIT_USER_NAME="Dan Farafon"
GIT_USER_EMAIL="dan@farafon.com"
GIT_MEMORY="git@github.com:pumacc-ai/farafon-dan.git"
OPENCLAW_MODEL="anthropic/claude-sonnet-4-6"
ANTHROPIC_API_KEY="<resolved from bw_secret_anthropic_api_key>"
UI_PORT=6767
NODE_HEAP_MB=1536
```

Run each section below on the remote:

```bash
ssh $AGENT_USER@$AGENT_HOST "bash -s" << 'ENDSSH'
  # commands here
ENDSSH
```

When multiple hosts share the same `ansible_host` IP (e.g. multiple users on one server), provision them sequentially, one per `AGENT_USER`.

Run each section remotely:
```bash
ssh $AGENT_USER@$AGENT_HOST "bash -s" << 'ENDSSH'
  # commands here
ENDSSH
```

---

## 1. Install Ansible (sudo)

```bash
sudo apt-get update && sudo apt-get install -y ansible
```

## 2. SSH Keys

Copy local SSH keys to the remote agent. Run these **from the local machine**:

```bash
# Project key (used for GitLab)
scp ~/.ssh/$PROJECT       $AGENT_USER@$AGENT_HOST:~/.ssh/id_rsa
scp ~/.ssh/$PROJECT.pub   $AGENT_USER@$AGENT_HOST:~/.ssh/id_rsa.pub

# GitHub key
scp ~/.ssh/github         $AGENT_USER@$AGENT_HOST:~/.ssh/github
scp ~/.ssh/github.pub     $AGENT_USER@$AGENT_HOST:~/.ssh/github.pub
```

On the remote, set permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa ~/.ssh/github
chmod 644 ~/.ssh/id_rsa.pub ~/.ssh/github.pub
```

Write `~/.ssh/config`:

```bash
cat > ~/.ssh/config << 'EOF'
Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_rsa
    StrictHostKeyChecking no

Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github
    StrictHostKeyChecking no
EOF
chmod 600 ~/.ssh/config
```

## 3. Git Identity

```bash
git config --global user.name  "$GIT_USER_NAME"
git config --global user.email "$GIT_USER_EMAIL"
git config --global init.defaultBranch main
```

## 4. Profile Setup

Append the shared team profile block to `~/.profile` (idempotent via markers):

```bash
PROFILE_BLOCK="$(cat local/profile)"   # content from local/profile in this repo

if grep -q "# BEGIN PUMACC PROFILE" ~/.profile 2>/dev/null; then
  # replace existing block between markers
  python3 - << PYEOF
import re, pathlib
p = pathlib.Path("$HOME/.profile")
text = p.read_text()
block = open("local/profile").read()
text = re.sub(
    r"# BEGIN PUMACC PROFILE.*?# END PUMACC PROFILE\n",
    "# BEGIN PUMACC PROFILE\n" + block + "\n# END PUMACC PROFILE\n",
    text, flags=re.DOTALL
)
p.write_text(text)
PYEOF
else
  printf "\n# BEGIN PUMACC PROFILE\n%s\n# END PUMACC PROFILE\n" "$PROFILE_BLOCK" >> ~/.profile
fi
```

Write `profile_vars` to `~/.profile`. For values prefixed `bw:<uuid>`, resolve from Bitwarden first:

```bash
# Resolve a bw: value
bws secret get <uuid> --output json | python3 -c "import json,sys; print(json.load(sys.stdin)['value'])"

# Write each var (skip if already present)
for VAR in KEY1=value1 KEY2=value2; do
  KEY="${VAR%%=*}"
  VAL="${VAR#*=}"
  grep -q "^export $KEY=" ~/.profile \
    || echo "export $KEY=$VAL" >> ~/.profile
done
```

## 5. Home Git Repo (git_memory)

Initialize `$HOME` as a git working tree from the `git_memory` remote:

```bash
export GIT_SSH_COMMAND="ssh -i ~/.ssh/github -o StrictHostKeyChecking=no"

if [ ! -d "$HOME/.git" ]; then
  git clone --separate-git-dir="$HOME/.git" "$GIT_MEMORY" /tmp/git-memory-tmp
  rm -rf /tmp/git-memory-tmp
  git -C "$HOME" checkout --force HEAD
fi

# Pull latest
git -C "$HOME" remote set-url origin "$GIT_MEMORY" 2>/dev/null \
  || git -C "$HOME" remote add origin "$GIT_MEMORY"
git -C "$HOME" fetch origin
git -C "$HOME" reset --hard FETCH_HEAD
```

## 6. Deploy .gitignore

Write `~/.gitignore` after the git reset so it is never wiped by a pull:

```bash
cat > ~/.gitignore << 'EOF'
# ── Root dotfiles (local state — not for sharing) ────────────────
/.*
!/.gitignore
!/.gitmodules

# ── Infrastructure dirs ───────────────────────────────────────────
/playbooks/
/local/
/ssh/
/inventory.yaml
/run

# ── Ansible-managed scripts ───────────────────────────────────────
/sync-memory.sh

# ── SSH & credentials (NEVER commit) ─────────────────────────────
.ssh/
.gnupg/
*.pem
*.key
*.p12
*.pfx
*_rsa
*_ed25519
*_ecdsa
.netrc
.pgpass

# ── Secrets / environment ─────────────────────────────────────────
.env
.env.*
.envrc
*credentials*
*secret*
auth-profiles.json

# ── Claude temp & cache ───────────────────────────────────────────
.claude/projects/
.claude/todos/
.claude/cache/
.claude/statsig/
.claude/ide/
*.claude-backup

# ── Vim / Neovim ──────────────────────────────────────────────────
*.swp
*.swo
*.swn
*~
.viminfo
.netrwhist
.vim/undo/
.vim/backup/
.vim/swap/

# ── Temp & scratch ────────────────────────────────────────────────
tmp/
temp/
*.tmp
*.temp
*.bak
*.orig

# ── Logs ──────────────────────────────────────────────────────────
*.log
.sync-memory.log

# ── OS noise ──────────────────────────────────────────────────────
.DS_Store
Thumbs.db
desktop.ini

# ── Node.js ───────────────────────────────────────────────────────
node_modules/
npm-debug.log*
yarn-error.log*
.pnpm-store/

# ── Python ────────────────────────────────────────────────────────
__pycache__/
*.pyc
*.pyo
.venv/
venv/
*.egg-info/
dist/
build/

# ── Compiled / binary artefacts ───────────────────────────────────
*.o
*.so
*.a
*.class
*.jar
EOF
```

## 7. Deploy BOOTSTRAP.md

Copy `local/BOOTSTRAP.md.template` from this repo to the remote, substituting `agent_name`:

```bash
AGENT_NAME="$GIT_USER_NAME"
sed "s/{{ agent_name }}/$AGENT_NAME/g" local/BOOTSTRAP.md.template \
  | ssh $AGENT_USER@$AGENT_HOST "cat > ~/BOOTSTRAP.md"
```

## 8. Register Git Submodules

For each repo in `git_repos` (`{url, dest}` pairs from inventory):

```bash
export GIT_SSH_COMMAND="ssh -i ~/.ssh/github -o StrictHostKeyChecking=no"
cd "$HOME"

REPO_URL="git@github.com:pumacc-ai/farafon.git"
REPO_DEST="farafon"

ALREADY=$(git config --file .gitmodules --get "submodule.${REPO_DEST}.url" 2>/dev/null || echo "")

if [ -n "$ALREADY" ]; then
  git submodule update --init --remote "$REPO_DEST"
else
  rm -rf "$REPO_DEST"
  git submodule add --force "$REPO_URL" "$REPO_DEST"
fi
```

Use the correct SSH key based on the remote host:
- `github.com` → `~/.ssh/github`
- `gitlab.com` → `~/.ssh/id_rsa`

## 9. Deploy sync-memory.sh

```bash
cat > ~/sync-memory.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail

SSH_KEY="$HOME/.ssh/github"
LOG="$HOME/.sync-memory.log"
TS() { date -u +"%Y-%m-%dT%H:%M:%SZ"; }
export GIT_SSH_COMMAND="ssh -i $SSH_KEY -o StrictHostKeyChecking=no"

[ -d "$HOME/.git" ] || { echo "$(TS) ERROR: $HOME is not a git repo" >> "$LOG"; exit 1; }

cd "$HOME"

if [ -f .gitmodules ]; then
  git submodule update --remote --merge 2>&1 \
    | sed "s/^/$(TS) submodule: /" >> "$LOG" || true
fi

git add -A

if git diff --cached --quiet; then
  echo "$(TS) No changes" >> "$LOG"
  exit 0
fi

git commit -m "sync: $(TS)"
git push origin HEAD
echo "$(TS) Pushed changes" >> "$LOG"
EOF
chmod +x ~/sync-memory.sh
```

Schedule via cron (every 15 minutes):

```bash
(crontab -l 2>/dev/null | grep -v "sync-memory.sh"; \
 echo "*/15 * * * * /usr/bin/bash $HOME/sync-memory.sh >> $HOME/.sync-memory.log 2>&1") \
 | crontab -
```

## 10. Install OpenClaw

```bash
export PNPM_HOME="$HOME/.local/share/pnpm"
export PATH="$PNPM_HOME:$PATH"

pnpm setup
pnpm add -g openclaw@latest
```

## 11. Create OpenClaw Directories

```bash
mkdir -p ~/.openclaw/agents/main/sessions \
         ~/.openclaw/agents/main/agent \
         ~/.openclaw/credentials \
         ~/.openclaw/extensions
chmod 700 ~/.openclaw ~/.openclaw/credentials
```

## 12. Write openclaw.json Config

Resolve `OPENCLAW_MODEL` and `ANTHROPIC_API_KEY` from `~/.profile` first:

```bash
source ~/.profile 2>/dev/null
OPENCLAW_MODEL="${OPENCLAW_MODEL:-anthropic/claude-sonnet-4-6}"
```

Write config:

```bash
cat > ~/.openclaw/openclaw.json << EOF
{
  "agents": {
    "defaults": {
      "workspace": "$HOME",
      "model": {
        "primary": "$OPENCLAW_MODEL"
      }
    }
  },
  "gateway": {
    "mode": "local",
    "port": $UI_PORT,
    "trustedProxies": ["127.0.0.1", "::1"],
    "controlUi": {
      "allowedOrigins": ["https://$HOSTNAME"],
      "dangerouslyDisableDeviceAuth": false
    }
  }
}
EOF
chmod 600 ~/.openclaw/openclaw.json
```

Write Anthropic credentials:

```bash
source ~/.profile 2>/dev/null
cat > ~/.openclaw/credentials/anthropic.json << EOF
{
  "apiKey": "$ANTHROPIC_API_KEY"
}
EOF
chmod 600 ~/.openclaw/credentials/anthropic.json
```

## 13. Run OpenClaw Doctor

```bash
export PNPM_HOME="$HOME/.local/share/pnpm"
export PATH="$PNPM_HOME:$PATH"
export NODE_OPTIONS="--max-old-space-size=$NODE_HEAP_MB"

openclaw doctor --fix || true
```

## 14. Install Gateway Systemd Service

```bash
export PNPM_HOME="$HOME/.local/share/pnpm"
export PATH="$PNPM_HOME:$PATH"
SERVICE="$HOME/.config/systemd/user/openclaw-gateway.service"

[ -f "$SERVICE" ] || openclaw gateway install

# Add --allow-unconfigured to ExecStart
sed -i -E 's|^(ExecStart=.+gateway --port [0-9]+)$|\1 --allow-unconfigured|' "$SERVICE"

# Set Node.js heap limit
sed -i "/^Environment=OPENCLAW_SERVICE_KIND=gateway/a Environment=NODE_OPTIONS=--max-old-space-size=$NODE_HEAP_MB" "$SERVICE"

# Inject ANTHROPIC_API_KEY
source ~/.profile 2>/dev/null
if [ -n "${ANTHROPIC_API_KEY:-}" ]; then
  sed -i '/^Environment=ANTHROPIC_API_KEY=/d' "$SERVICE"
  sed -i "/^Environment=NODE_OPTIONS=/a Environment=ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY" "$SERVICE"
fi

# Write profile env drop-in (CLAUDE_MODEL, OPENCLAW_MODEL, BWS_ACCESS_TOKEN, etc.)
DROPIN_DIR="$HOME/.config/systemd/user/openclaw-gateway.service.d"
mkdir -p "$DROPIN_DIR"
{
  echo "[Service]"
  for var in CLAUDE_MODEL OPENCLAW_MODEL BWS_ACCESS_TOKEN BW_ACCESS_TOKEN; do
    val=$(bash -c "source ~/.profile 2>/dev/null; printf '%s' \"\${${var}:-}\"" 2>/dev/null || true)
    [ -n "$val" ] && echo "Environment=\"${var}=${val}\""
  done
} > "$DROPIN_DIR/profile-env.conf"
```

## 15. Start Gateway Service

```bash
XDG_RUNTIME_DIR="/run/user/$(id -u)"
export XDG_RUNTIME_DIR

systemctl --user daemon-reload
systemctl --user enable --now openclaw-gateway 2>/dev/null \
  || systemctl --user enable --now openclaw 2>/dev/null || true
```

## 16. Install Bundled Extensions

Copies extensions (e.g. `whatsapp`) from the pnpm content-addressed store into
`~/.openclaw/extensions/`. A version-marker file tracks which store path was used,
so extensions are re-copied automatically when openclaw upgrades.

```bash
export PNPM_HOME="$HOME/.local/share/pnpm"
export PATH="$PNPM_HOME:$PATH"

EXT_BASE=$(find ~/.local/share/pnpm/global/5/.pnpm/openclaw@*/node_modules/openclaw/extensions \
  -maxdepth 0 -type d 2>/dev/null | sort -V | tail -1)

[ -z "$EXT_BASE" ] && { echo "openclaw extensions dir not found"; exit 0; }

for plugin in whatsapp; do
  DEST=~/.openclaw/extensions/$plugin
  VERSION_FILE=~/.openclaw/extensions/.$plugin.version
  CURRENT=$(cat "$VERSION_FILE" 2>/dev/null || echo "")

  if [ "$EXT_BASE" = "$CURRENT" ] && [ -d "$DEST" ]; then
    echo "SKIP: $plugin already current"
    continue
  fi

  SRC="$EXT_BASE/$plugin"
  [ -d "$SRC" ] || { echo "WARN: $plugin not in $EXT_BASE"; continue; }

  rm -rf "$DEST"
  cp -r "$SRC" "$DEST"
  echo "$EXT_BASE" > "$VERSION_FILE"
  echo "OK: installed $plugin"
done
```

Restart the gateway after installing extensions:

```bash
XDG_RUNTIME_DIR="/run/user/$(id -u)" \
  systemctl --user daemon-reload
XDG_RUNTIME_DIR="/run/user/$(id -u)" \
  systemctl --user restart openclaw-gateway 2>/dev/null \
  || systemctl --user restart openclaw 2>/dev/null || true

# Wait up to 30s for gateway to bind on $UI_PORT
for i in $(seq 1 30); do
  ss -tlnp | grep -q ":$UI_PORT" && break
  sleep 1
done
```

## 17. Auto-Approve Pending Device Pairings

```bash
export PNPM_HOME="$HOME/.local/share/pnpm"
export PATH="$PNPM_HOME:$PATH"
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export NODE_OPTIONS="--max-old-space-size=$NODE_HEAP_MB"

max=10; count=0
while [ $count -lt $max ]; do
  pending=$(timeout 15 openclaw devices list --json 2>/dev/null \
    | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d.get('pending',[])))" 2>/dev/null || echo 0)
  [ "${pending:-0}" -eq 0 ] && break
  timeout 15 openclaw devices approve --latest 2>&1
  count=$((count + 1))
done
```

## 18. Show Dashboard URL

```bash
TOKEN=$(grep -oP '(?<=OPENCLAW_GATEWAY_TOKEN=)\S+' \
  ~/.config/systemd/user/openclaw-gateway.service 2>/dev/null || echo "")

echo ""
echo "=========================================="
echo " OpenClaw Dashboard"
echo "=========================================="
echo " URL: https://$HOSTNAME/#token=$TOKEN"
echo "=========================================="
```

## Debugging

```bash
# Service status
systemctl --user status openclaw-gateway

# Live logs
journalctl --user -u openclaw-gateway -f

# Gateway port check
ss -tlnp | grep $UI_PORT

# OpenClaw CLI check
openclaw gateway status
openclaw devices list
```
