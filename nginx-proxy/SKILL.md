---
name: Nginx Proxy
slug: nginx-proxy
version: "2.2.0"
description: Set up Nginx as a TLS-terminating reverse proxy with Let's Encrypt certificates and WebSocket support using direct shell commands. Auto-discovers target servers from inventory.yaml when present.
---

# Nginx Proxy Skill

Set up Nginx as a TLS reverse proxy with Let's Encrypt and WebSocket support.
All steps run directly on the target server as root (or via sudo).

## Inventory Detection

If `inventory.yaml` exists in the current directory, use it to discover target servers automatically. Extract all hosts and their variables with:

```bash
python3 - << 'EOF'
import yaml, json

with open('inventory.yaml') as f:
    inv = yaml.safe_load(f)

results = []
all_vars = inv.get('all', {}).get('vars', {})

for group_name, group in inv.get('all', {}).get('children', {}).items():
    group_vars = group.get('vars', {})
    for hostname, host_vars in (group.get('hosts') or {}).items():
        host_vars = host_vars or {}
        results.append({
            'hostname':     hostname,
            'ansible_host': host_vars.get('ansible_host', hostname),
            'ansible_user': host_vars.get('ansible_user', group_vars.get('ansible_user', 'root')),
            'ui_port':      host_vars.get('ui_port'),
            'web_title':    host_vars.get('web_title', hostname),
            'openclaw':     host_vars.get('openclaw_host', False),
        })

print(json.dumps(results, indent=2))
EOF
```

Use the output to set variables for each host before running the setup steps via SSH:

```bash
HOSTNAME="coder.pumacc.com"
ANSIBLE_HOST="129.212.150.103"   # IP to SSH to
ANSIBLE_USER="pumacc"
UI_PORT="18789"
WEB_TITLE="PumaCoder"
OPENCLAW=true
```

Run each setup step on the remote server:

```bash
ssh $ANSIBLE_USER@$ANSIBLE_HOST "sudo bash -s" << 'ENDSSH'
  # paste the commands from steps 1–7 below, with variables substituted
ENDSSH
```

When multiple hosts are present, loop over them and repeat for each.

---

Variables used throughout:
- `HOSTNAME` — the server's public domain name (e.g. `coder.pumacc.com`)
- `UI_PORT` — the backend port nginx proxies to (e.g. `18789`)
- `WEB_TITLE` — display name used in the HTTP header and landing page

## 1. Install Packages

```bash
apt-get update
apt-get install -y nginx certbot python3-certbot-nginx
```

## 2. Open Firewall Ports

Open ports before running certbot — the ACME HTTP-01 challenge requires port 80 to be reachable.

```bash
ufw allow 80/tcp
ufw allow 443/tcp
```

## 3. Obtain Let's Encrypt Certificate

Check whether a valid cert already exists and covers `$HOSTNAME`. If the cert is missing,
expired, or its SANs do not include `$HOSTNAME`, provision a new one via certbot.

```bash
CERT="/etc/letsencrypt/live/$HOSTNAME/fullchain.pem"

cert_valid_for_hostname() {
  local cert="$1" host="$2"
  [ -f "$cert" ] || return 1

  # Check expiry — treat certs expiring within 7 days as invalid
  openssl x509 -noout -checkend 604800 -in "$cert" 2>/dev/null || return 1

  # Check that $host appears in CN or SANs
  openssl x509 -noout -text -in "$cert" 2>/dev/null \
    | grep -qE "(Subject:.*CN\s*=\s*$host|DNS:$host)" && return 0

  return 1
}

if cert_valid_for_hostname "$CERT" "$HOSTNAME"; then
  echo "Certificate already valid for $HOSTNAME — skipping certbot"
else
  echo "No valid certificate for $HOSTNAME — provisioning via Let's Encrypt"

  # Remove stale state from any previous attempt
  rm -rf /etc/letsencrypt/renewal/$HOSTNAME.conf \
         /etc/letsencrypt/archive/$HOSTNAME \
         /etc/letsencrypt/live/$HOSTNAME

  systemctl start nginx

  certbot --nginx \
    --non-interactive \
    --agree-tos \
    --register-unsafely-without-email \
    -d $HOSTNAME
fi
```

## 4. WebSocket Upgrade Map

Write `/etc/nginx/conf.d/websocket_upgrade.conf` — makes `$connection_upgrade` available to all server blocks:

```bash
cat > /etc/nginx/conf.d/websocket_upgrade.conf << 'EOF'
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
EOF
```

## 5. Nginx Site Config

Write `/etc/nginx/sites-available/default`:

```bash
cat > /etc/nginx/sites-available/default << EOF
# HTTP — redirect all traffic to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name $HOSTNAME;

    location / {
        return 301 https://\$host\$request_uri;
    }
}

# HTTPS — TLS termination + reverse proxy
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name $HOSTNAME;

    ssl_certificate     /etc/letsencrypt/live/$HOSTNAME/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$HOSTNAME/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    add_header X-Web-Title "$WEB_TITLE" always;

    location / {
        proxy_pass          http://localhost:$UI_PORT;
        proxy_http_version  1.1;

        proxy_set_header    Upgrade    \$http_upgrade;
        proxy_set_header    Connection \$connection_upgrade;

        proxy_set_header    Host              \$host;
        proxy_set_header    X-Real-IP         \$remote_addr;
        proxy_set_header    X-Forwarded-For   \$proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto \$scheme;

        proxy_buffering     off;
        proxy_cache_bypass  \$http_upgrade;
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }
}
EOF
```

Enable the site:

```bash
ln -sf /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
rm -f /etc/nginx/sites-enabled/default.bak   # remove stale certbot backup
```

## 6. OpenClaw Landing Page (optional)

When the server runs an OpenClaw gateway, the backend returns 404 for plain HTTP at root.
This serves a landing page for browser visits while letting WebSocket upgrades pass through.

Write `/var/www/html/landing.html`:

```bash
cat > /var/www/html/landing.html << EOF
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>$WEB_TITLE Coder Agent</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      min-height: 100vh; display: flex; align-items: center;
      justify-content: center; background: #0d1117; color: #b0bec5;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    }
    .card { width: min(520px, 90vw); border: 1px solid rgba(255,255,255,0.08);
      border-radius: 16px; padding: 40px; background: rgba(255,255,255,0.03); }
    .badge { display: inline-flex; align-items: center; gap: 7px; font-size: 12px;
      font-weight: 600; letter-spacing: 0.06em; text-transform: uppercase;
      color: #24e08a; margin-bottom: 24px; }
    .dot { width: 8px; height: 8px; border-radius: 50%; background: #24e08a;
      box-shadow: 0 0 8px #24e08a; animation: pulse 2s ease-in-out infinite; }
    @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.4; } }
    h1 { font-size: 28px; font-weight: 700; color: #e8ecf0;
      letter-spacing: -0.3px; margin-bottom: 10px; }
    .sub { font-size: 14px; line-height: 1.6; color: #78909c; margin-bottom: 32px; }
    .info-block { background: rgba(0,0,0,0.3); border: 1px solid rgba(255,255,255,0.06);
      border-radius: 10px; padding: 18px 20px; font-size: 13px; line-height: 1.7; }
    .info-block strong { color: #cfd8dc; }
    .info-block code { background: rgba(255,255,255,0.07); border-radius: 4px;
      padding: 1px 6px; font-size: 12px; color: #90caf9; }
    .info-block a { color: #2563eb; text-decoration: none; }
    .info-block a:hover { text-decoration: underline; }
    .footer { margin-top: 28px; font-size: 11px; color: #37474f; text-align: center; }
  </style>
</head>
<body>
  <div class="card">
    <div class="badge"><span class="dot"></span> Online</div>
    <h1>$WEB_TITLE Coder Agent</h1>
    <p class="sub">AI coding assistant powered by Claude — ready to help with code, architecture, and engineering tasks.</p>
    <div class="info-block">
      <strong>Connect via OpenClaw</strong><br>
      Open the <a href="https://openclaw.ai" target="_blank">OpenClaw</a> app and add a remote gateway:<br>
      <code>wss://$HOSTNAME</code>
      <br><br>
      <strong>WhatsApp</strong><br>
      Send a message to the configured WhatsApp group to interact with the agent directly.
    </div>
    <div class="footer">$HOSTNAME &mdash; $WEB_TITLE</div>
  </div>
</body>
</html>
EOF
```

Add the landing page intercept to the `location /` block inside the HTTPS server block:

```nginx
proxy_intercept_errors on;
error_page 404 = @landing;
```

Add the named location after the `location /` block:

```nginx
location @landing {
    root /var/www/html;
    try_files /landing.html =503;
    add_header Content-Type "text/html; charset=utf-8" always;
}
```

## 7. Validate and Reload

```bash
nginx -t && systemctl reload nginx
```

## Certificate Renewal

Certbot installs a systemd timer for automatic renewal. To force renewal manually:

```bash
certbot renew --force-renewal
systemctl reload nginx
```

## Re-issuing a Certificate

```bash
rm -rf /etc/letsencrypt/live/$HOSTNAME \
       /etc/letsencrypt/archive/$HOSTNAME \
       /etc/letsencrypt/renewal/$HOSTNAME.conf

certbot --nginx --non-interactive --agree-tos \
  --register-unsafely-without-email -d $HOSTNAME

systemctl reload nginx
```

## Debugging

```bash
nginx -t                             # validate config
tail -f /var/log/nginx/error.log     # error log
certbot certificates                 # check cert expiry
systemctl status nginx               # service status

# Test WebSocket upgrade
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  https://$HOSTNAME/
```
