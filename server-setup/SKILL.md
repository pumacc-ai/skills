---
name: Server Setup
slug: server-setup
version: "2.1.0"
description: Provision a new Ubuntu server over SSH as root. Reads host definitions from ./inventory.yaml. service_user is required per host — fails if missing. No Ansible required.
---

# Server Setup Skill

Provisions a fresh Ubuntu server over SSH as root using direct shell commands.
Requires `./inventory.yaml` to be present with host definitions.

## Inventory

Read host configuration from `./inventory.yaml`. **`service_user` is required for every host** — the skill must fail immediately if it is absent or empty for any host being provisioned.

Expected inventory structure:

```yaml
all:
  vars:
    project: PumaCoder
  children:
    agents:
      vars:
        ansible_user: root
      hosts:
        coder.pumacc.com:
          ansible_host: 129.212.150.103
          service_user: pumacc          # REQUIRED — fails if missing
          profile_vars:
            ANTHROPIC_API_KEY: "bw:9455622e-e759-4cec-ba11-b40b00df693b"
            CLAUDE_MODEL: "claude-sonnet-4-6"
```

Parse and validate all hosts:

```bash
python3 - << 'EOF'
import yaml, json, subprocess, sys

def bws_get_by_uuid(uuid):
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
errors   = []

for group_name, group in inv.get('all', {}).get('children', {}).items():
    group_vars = group.get('vars', {})
    for hostname, host_vars in (group.get('hosts') or {}).items():
        hv = host_vars or {}

        # service_user is required — fail hard if missing
        service_user = hv.get('service_user') or hv.get('bands_user')
        if not service_user:
            errors.append(f"ERROR: host '{hostname}' is missing required field 'service_user'")
            continue

        # Resolve profile_vars — bw:<uuid> values fetched from Bitwarden
        profile_vars = {}
        for pk, pv in hv.get('profile_vars', {}).items():
            if isinstance(pv, str) and pv.startswith('bw:'):
                profile_vars[pk] = bws_get_by_uuid(pv[3:])
            else:
                profile_vars[pk] = pv

        results.append({
            'hostname':     hostname,
            'ansible_host': hv.get('ansible_host', hostname),
            'service_user': service_user,
            'project':      all_vars.get('project', 'Agent'),
            'profile_vars': profile_vars,
        })

if errors:
    for e in errors:
        print(e, file=sys.stderr)
    sys.exit(1)

safe = json.loads(json.dumps(results))
for h in safe:
    for k in h.get('profile_vars', {}):
        h['profile_vars'][k] = '***'
print(json.dumps(safe, indent=2))
EOF
```

After parsing, set shell variables for the host being provisioned:

```bash
SERVER_HOST="129.212.150.103"   # ansible_host from inventory
SERVICE_USER="pumacc"           # service_user from inventory (required)
PROJECT="PumaCoder"             # all.vars.project from inventory
```

Run each section as root over SSH:

```bash
ssh root@$SERVER_HOST "bash -s" << 'ENDSSH'
  # commands here
ENDSSH
```

If `inventory.yaml` is not present, stop and ask the user to provide one before continuing.

---

## 1. Service User

```bash
# Create user with sudo group
id -u $SERVICE_USER &>/dev/null || useradd -m -s /bin/bash -G sudo $SERVICE_USER
usermod -c "$PROJECT CC Service Account" $SERVICE_USER

# Passwordless sudo
cat > /etc/sudoers.d/$SERVICE_USER << EOF
$SERVICE_USER ALL=(ALL) NOPASSWD:ALL
EOF
chmod 0440 /etc/sudoers.d/$SERVICE_USER
visudo -cf /etc/sudoers.d/$SERVICE_USER

# SSH directory
mkdir -p /home/$SERVICE_USER/.ssh
chmod 700 /home/$SERVICE_USER/.ssh
chown $SERVICE_USER:$SERVICE_USER /home/$SERVICE_USER/.ssh
```

## 2. Authorized Keys

Copy `authorized_keys` from the local machine to the remote server:

```bash
# From local machine
scp authorized_keys root@$SERVER_HOST:/home/$SERVICE_USER/.ssh/authorized_keys
ssh root@$SERVER_HOST "chmod 600 /home/$SERVICE_USER/.ssh/authorized_keys && \
  chown $SERVICE_USER:$SERVICE_USER /home/$SERVICE_USER/.ssh/authorized_keys"
```

## 3. Profile — Touch and Write Vars

```bash
touch /home/$SERVICE_USER/.profile
chmod 644 /home/$SERVICE_USER/.profile
chown $SERVICE_USER:$SERVICE_USER /home/$SERVICE_USER/.profile
```

Write each `profile_vars` entry (resolve `bw:<uuid>` values locally via `bws` before SSHing — never store secrets in inventory):

```bash
# For each KEY=VALUE from profile_vars:
grep -q "^export KEY=" /home/$SERVICE_USER/.profile \
  || echo "export KEY=VALUE" >> /home/$SERVICE_USER/.profile
```

## 4. Fix APT Sources for EOL Ubuntu Releases

Migrates EOL releases (oracular, mantic, lunar, kinetic) to `old-releases.ubuntu.com`:

```bash
EOL_RELEASES="oracular|mantic|lunar|kinetic"
CURRENT_RELEASES="plucky|noble"

# sources.list
if grep -qE "$EOL_RELEASES" /etc/apt/sources.list 2>/dev/null; then
  sed -i \
    -e 's|http://mirrors.digitalocean.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' \
    -e 's|http://security.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' \
    -e 's|http://archive.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' \
    /etc/apt/sources.list
fi
if grep -qE "$CURRENT_RELEASES" /etc/apt/sources.list 2>/dev/null; then
  sed -i 's|http://old-releases.ubuntu.com/ubuntu|http://archive.ubuntu.com/ubuntu|g' \
    /etc/apt/sources.list
fi

# ubuntu.sources (newer Ubuntu)
if [ -f /etc/apt/sources.list.d/ubuntu.sources ]; then
  if grep -qE "$EOL_RELEASES" /etc/apt/sources.list.d/ubuntu.sources 2>/dev/null; then
    sed -i \
      -e 's|URIs: http://mirrors.digitalocean.com/ubuntu|URIs: http://old-releases.ubuntu.com/ubuntu|g' \
      -e 's|URIs: http://security.ubuntu.com/ubuntu|URIs: http://old-releases.ubuntu.com/ubuntu|g' \
      -e 's|URIs: http://archive.ubuntu.com/ubuntu|URIs: http://old-releases.ubuntu.com/ubuntu|g' \
      /etc/apt/sources.list.d/ubuntu.sources
  fi
  if grep -qE "$CURRENT_RELEASES" /etc/apt/sources.list.d/ubuntu.sources 2>/dev/null; then
    sed -i 's|URIs: http://old-releases.ubuntu.com/ubuntu|URIs: http://archive.ubuntu.com/ubuntu|g' \
      /etc/apt/sources.list.d/ubuntu.sources
  fi
fi

# Remove stale backup files
rm -f /etc/apt/sources.list.d/ubuntu.sources.distUpgrade \
      /etc/apt/sources.list.d/ubuntu.sources.save
```

## 5. System Update and Packages

```bash
apt-get update
apt-get upgrade -y
apt-get dist-upgrade -y
apt-get install -y \
  curl git vim openvpn easy-rsa ufw \
  python3 python3-pip python3-dev \
  rsync yq jq
```

## 6. Swap

```bash
# Remove undersized swapfile (< 2 GB)
if swapon --show | grep -q /swapfile; then
  SIZE=$(stat -c%s /swapfile)
  if [ "$SIZE" -lt 2147483648 ]; then
    swapoff /swapfile
    rm -f /swapfile
  fi
fi

# Create 2 GB swapfile if none exists
if [ "$(swapon --show | wc -l)" -eq 0 ]; then
  dd if=/dev/zero of=/swapfile bs=1024 count=2097152
  chmod 600 /swapfile
  mkswap /swapfile
  swapon /swapfile
  grep -qxF '/swapfile none swap sw 0 0' /etc/fstab \
    || echo '/swapfile none swap sw 0 0' >> /etc/fstab
fi

# Tune kernel memory settings
sysctl -w vm.swappiness=60
sysctl -w vm.vfs_cache_pressure=50
grep -q "^vm.swappiness"        /etc/sysctl.conf || echo "vm.swappiness=60"        >> /etc/sysctl.conf
grep -q "^vm.vfs_cache_pressure" /etc/sysctl.conf || echo "vm.vfs_cache_pressure=50" >> /etc/sysctl.conf
```

## 7. Node.js 22

```bash
NODE_VERSION="22"

mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
  | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] \
  https://deb.nodesource.com/node_${NODE_VERSION}.x nodistro main" \
  > /etc/apt/sources.list.d/nodesource.list

apt-get update
apt-get install -y nodejs
```

## 8. Global npm Packages

```bash
npm install -g pnpm
npm install -g @bitwarden/cli
npm install -g @anthropic-ai/claude-code
```

## 9. SSH Hardening

```bash
sed -i 's|^#\?PermitRootLogin .*|PermitRootLogin prohibit-password|' /etc/ssh/sshd_config
sshd -t   # validate config
systemctl restart ssh
```

## 10. fail2ban

```bash
apt-get install -y fail2ban

# PostgreSQL filter
cat > /etc/fail2ban/filter.d/postgresql.conf << 'EOF'
[Definition]
failregex = ^.* FATAL:\s+no pg_hba\.conf entry for host "<HOST>", .+$
ignoreregex =
EOF

# Jail config
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5

[sshd]
enabled  = true
port     = ssh
logpath  = %(sshd_log)s
backend  = %(sshd_backend)s

[postgresql]
enabled  = true
filter   = postgresql
logpath  = /var/log/postgresql/postgresql-*.log
maxretry = 5
bantime  = 3600
EOF

systemctl enable --now fail2ban
systemctl restart fail2ban
```

## Verify

```bash
# Service user exists
id $SERVICE_USER

# Node.js and global tools
node --version && npm --version && pnpm --version
claude --version
bw --version

# Swap active
swapon --show

# fail2ban running
systemctl status fail2ban
fail2ban-client status
```
