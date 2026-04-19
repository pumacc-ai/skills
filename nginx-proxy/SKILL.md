---
name: Nginx Proxy
slug: nginx-proxy
version: "1.0.0"
description: Set up Nginx as a TLS-terminating reverse proxy with Let's Encrypt certificates and WebSocket support. Run via the proxy.yaml Ansible playbook.
---

# Nginx Proxy Skill

Provisions Nginx as a TLS reverse proxy using `proxy.yaml`. Handles HTTPS, WebSocket upgrades, Let's Encrypt certificates, and an optional OpenClaw landing page.

## Quick Start

```bash
# Provision nginx proxy for a specific host
./run proxy coder.pumacc.com

# Provision all hosts in the agents group
./run proxy
```

The host must already exist in `inventory.yaml` with `ansible_host`, `ui_port`, and `web_title` set.

## Required inventory.yaml Fields

```yaml
hosts:
  myserver.example.com:
    ansible_host: 1.2.3.4
    ui_port: 18789          # backend port nginx will proxy to
    web_title: "My Agent"   # used in HTTP header and landing page title
```

Optional flag to enable the OpenClaw landing page:

```yaml
    openclaw_host: true
```

## What the Playbook Does

### 1. Packages

Installs: `nginx`, `certbot`, `python3-certbot-nginx`

### 2. Firewall

Opens ports 80 (HTTP) and 443 (HTTPS) via UFW before running certbot so the ACME HTTP-01 challenge can reach the server.

### 3. Let's Encrypt Certificate

- Checks for an existing cert at `/etc/letsencrypt/live/<hostname>/fullchain.pem`
- If absent: clears any stale ACME state, starts nginx, then runs:

```bash
certbot --nginx --non-interactive --agree-tos \
  --register-unsafely-without-email -d <hostname>
```

- Skips the certbot step if the cert already exists (idempotent)

### 4. WebSocket Upgrade Map

Writes `/etc/nginx/conf.d/websocket_upgrade.conf`:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

This makes the `$connection_upgrade` variable available globally to all server blocks.

### 5. Nginx Site Config

Writes `/etc/nginx/sites-available/default` with two server blocks:

**HTTP block** — redirects all traffic to HTTPS:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name <hostname>;
    location / { return 301 https://$host$request_uri; }
}
```

**HTTPS block** — TLS termination + reverse proxy to `localhost:<ui_port>`:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name <hostname>;

    ssl_certificate     /etc/letsencrypt/live/<hostname>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<hostname>/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    add_header X-Web-Title "<web_title>" always;

    location / {
        proxy_pass         http://localhost:<ui_port>;
        proxy_http_version 1.1;

        # WebSocket
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Forwarding
        proxy_set_header Host             $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Streaming / long-lived connections
        proxy_buffering    off;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

### 6. OpenClaw Landing Page (optional)

When `openclaw_host: true` is set in inventory, the playbook also:

- Writes a styled landing page to `/var/www/html/landing.html`
- Adds an `error_page 404 = @landing` intercept to the proxy block so the OpenClaw gateway's 404 at root serves the landing page instead (WebSocket upgrades bypass this)

### 7. Cleanup

Removes `/etc/nginx/sites-enabled/default.bak` — a stale backup certbot sometimes leaves that causes duplicate `server_name` conflicts.

Validates config with `nginx -t` before reloading.

## Manual Certificate Renewal

Certbot auto-renewal is configured by the certbot package. To force a renewal manually:

```bash
certbot renew --force-renewal
```

## Debugging

```bash
# Check nginx config syntax
nginx -t

# View nginx error log
tail -f /var/log/nginx/error.log

# Check cert expiry
certbot certificates

# Test WebSocket connectivity
curl -i -N -H "Connection: Upgrade" -H "Upgrade: websocket" \
     -H "Host: myserver.example.com" \
     https://myserver.example.com/
```

## Re-issuing a Certificate

If a cert is corrupted or needs to be reissued, delete the letsencrypt state and re-run the playbook:

```bash
# On the server
rm -rf /etc/letsencrypt/live/<hostname> \
       /etc/letsencrypt/archive/<hostname> \
       /etc/letsencrypt/renewal/<hostname>.conf

# Then from the coder agent
./run proxy <hostname>
```
