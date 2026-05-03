---
name: digital-ocean
description: Manage DigitalOcean infrastructure with the `doctl` CLI — droplets, networking (VPCs, firewalls, floating/reserved IPs, load balancers, domains), SSH keys, snapshots, and regions. Use this skill whenever the user mentions DigitalOcean, doctl, droplets, DO VPCs, DO load balancers, reserved/floating IPs, or asks to spin up/destroy/list/resize cloud servers on DigitalOcean — even casually ("create a DO box", "ssh into my droplet"). Also handles installing and authenticating doctl when the tool is missing or the user has never used it on this machine.
---

# DigitalOcean (doctl)

`doctl` is DigitalOcean's official command-line client. It is the primary way to drive DigitalOcean from a terminal or script and is what this skill covers.

## When this skill applies

Trigger on anything DigitalOcean-flavored: provisioning droplets, attaching block storage, configuring networking (VPCs, firewalls, reserved IPs, load balancers), managing DNS, SSH keys, snapshots, and listing/inspecting account resources. If the user is talking about another cloud (AWS, GCP, Azure, Hetzner, Linode), this skill does not apply.

## Workflow

1. **Check the tool is available.** Run `doctl version`. If it errors with "command not found", follow `references/installation.md` to install it. Don't silently install — tell the user what you're about to do and confirm if the install path is non-trivial (e.g., requires sudo or a package manager change).

2. **Check authentication.** Run `doctl account get`. If it errors with `Unable to initialize DigitalOcean API client: access token is required`, the user needs to authenticate. Ask them for an API token (https://cloud.digitalocean.com/account/api/tokens) and run `doctl auth init` (or `doctl auth init --context <name>` for a named context). Never paste tokens into shell history — pipe via stdin or let the user type into the prompt.

3. **Pick the right command.** Use the references:
   - `references/droplets.md` — create/list/delete/resize droplets, snapshots, SSH access, user-data, images, sizes, regions.
   - `references/networking.md` — VPCs, firewalls, reserved IPs (formerly floating IPs), load balancers, DNS/domains, certificates.
   - `references/installation.md` — install paths and auth setup.

4. **Prefer JSON output for scripts and parsing.** Add `--output json` and pipe to `jq`. Default tabular output is for humans; JSON is for Claude.

5. **Confirm before destructive actions.** `doctl compute droplet delete`, `doctl compute load-balancer delete`, `doctl compute floating-ip delete`, etc. all destroy real, billable resources. Print what will be deleted, then ask the user to confirm. Don't pass `--force` without explicit user authorization.

6. **Use `--dry-run` where supported, and `--wait` for long operations.** Many commands (droplet create, snapshot, resize) accept `--wait` to block until the action completes — useful when subsequent steps depend on the resource being ready.

## Common patterns

**Create a droplet and SSH in:**
```bash
doctl compute droplet create my-box \
  --region nyc3 --size s-1vcpu-1gb --image ubuntu-24-04-x64 \
  --ssh-keys $(doctl compute ssh-key list --output json | jq -r '.[0].fingerprint') \
  --wait --output json | jq -r '.[0].networks.v4[] | select(.type=="public") | .ip_address'
```

**Tag-based operations** are powerful: tag droplets at creation (`--tag-names web,prod`) and operate on the tag (`doctl compute droplet delete --tag-name dev`).

**List anything:** `doctl compute <resource> list --output json`. This is the safest read-only way to discover state before changing it.

## Token & secret hygiene

- Never echo `$DIGITALOCEAN_ACCESS_TOKEN` or write it into a file the user didn't ask for.
- `doctl` stores credentials in `~/.config/doctl/config.yaml`. Don't `cat` that file into chat output.
- If the user is on a shared machine, suggest `doctl auth init --context <name>` and `doctl auth switch` over a single global token.

## Output style

When listing resources, show the user the JSON or a clean table — not a wall of prose. When provisioning, report the resource ID, public IP, and the command they'd use to SSH/access it. When deleting, summarize what was removed.

## Reference index

- `references/installation.md` — Install doctl on Linux, macOS, Windows; authenticate and switch contexts.
- `references/droplets.md` — Droplet lifecycle, sizing, images, snapshots, SSH keys, volumes.
- `references/networking.md` — VPCs, firewalls, reserved IPs, load balancers, domains/DNS, certificates.

Official docs: https://docs.digitalocean.com/reference/doctl/
