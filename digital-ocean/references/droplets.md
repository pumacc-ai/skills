# Droplets

A "droplet" is DigitalOcean's term for a virtual machine. All commands live under `doctl compute droplet ...`.

## Discover what's available

```bash
doctl compute region list                  # regions (nyc3, sfo3, ams3, fra1, sgp1, blr1, syd1, tor1, lon1)
doctl compute size list                    # slugs like s-1vcpu-1gb, c-2, g-2vcpu-8gb
doctl compute image list --public          # public OS images
doctl compute image list-distribution      # filter to distros
doctl compute ssh-key list                 # SSH keys registered on the account
```

Use `--output json | jq` for any of these when scripting.

## Create

Minimum required: name, region, size, image.

```bash
doctl compute droplet create web-01 \
  --region nyc3 \
  --size s-1vcpu-1gb \
  --image ubuntu-24-04-x64 \
  --ssh-keys 12:34:56:...:fingerprint \
  --wait
```

Useful flags:

| Flag | Purpose |
|---|---|
| `--ssh-keys <fp,fp>` | Comma-separated fingerprints or numeric IDs. Required for password-less SSH. |
| `--user-data "..."` | Cloud-init script as a string. |
| `--user-data-file path` | Cloud-init script from a file. |
| `--enable-monitoring` | Install the DO monitoring agent. |
| `--enable-backups` | Enable weekly backups (+20% cost). |
| `--enable-ipv6` | Allocate an IPv6 address. |
| `--enable-private-networking` | Legacy; use `--vpc-uuid` instead. |
| `--vpc-uuid <uuid>` | Place in a specific VPC (defaults to region's default VPC). |
| `--tag-names a,b` | Apply tags at creation — enables tag-based operations later. |
| `--volumes <id,id>` | Attach existing block-storage volumes. |
| `--wait` | Block until the droplet is `active`. |

Bulk create: pass multiple names — `doctl compute droplet create web-01 web-02 web-03 ...`.

## Inspect

```bash
doctl compute droplet list
doctl compute droplet list --tag-name prod
doctl compute droplet get <id|name>
doctl compute droplet get <id> --template '{{.Networks.V4}}'
doctl compute droplet-action get <droplet-id> <action-id>
```

Get just the public IPv4 (handy for piping into ssh):

```bash
doctl compute droplet get web-01 --output json \
  | jq -r '.[0].networks.v4[] | select(.type=="public") | .ip_address'
```

## Lifecycle actions

```bash
doctl compute droplet-action power-off <id> --wait
doctl compute droplet-action power-on <id> --wait
doctl compute droplet-action reboot <id>
doctl compute droplet-action shutdown <id>           # graceful
doctl compute droplet-action power-cycle <id>        # hard
doctl compute droplet-action rebuild <id> --image ubuntu-24-04-x64
doctl compute droplet-action rename <id> --droplet-name new-name
doctl compute droplet-action resize <id> --size s-2vcpu-4gb --resize-disk --wait
doctl compute droplet-action enable-backups <id>
doctl compute droplet-action snapshot <id> --snapshot-name pre-upgrade --wait
```

`--resize-disk` makes a resize permanent (and irreversible to a smaller size). Without it, only CPU/RAM change and you can downgrade later.

## Delete

```bash
doctl compute droplet delete <id|name>           # prompts for confirmation
doctl compute droplet delete <id> --force         # no prompt — be careful
doctl compute droplet delete --tag-name dev --force
```

Always preview first: `doctl compute droplet list --tag-name dev`.

## SSH

doctl can shell directly into a droplet by name or ID, resolving the IP for you:

```bash
doctl compute ssh web-01                 # uses default user (root)
doctl compute ssh web-01 --ssh-user deploy
doctl compute ssh <id> --ssh-command "uptime"
```

## SSH key management

```bash
doctl compute ssh-key list
doctl compute ssh-key import my-laptop --public-key-file ~/.ssh/id_ed25519.pub
doctl compute ssh-key create my-laptop --public-key "$(cat ~/.ssh/id_ed25519.pub)"
doctl compute ssh-key delete <id|fingerprint>
```

## Snapshots & images

```bash
doctl compute snapshot list
doctl compute snapshot list --resource droplet
doctl compute snapshot get <id>
doctl compute snapshot delete <id>

doctl compute image list --user           # custom + snapshot images you own
doctl compute image create my-image --image-url https://example.com/image.qcow2 --region nyc3
doctl compute image delete <id>
```

Restoring from a snapshot is a rebuild action: `doctl compute droplet-action restore <droplet-id> --image <snapshot-id>`.

## Block storage volumes

Adjacent to droplets — managed under `doctl compute volume`.

```bash
doctl compute volume create data --region nyc3 --size 100GiB --fs-type ext4
doctl compute volume-action attach <volume-id> <droplet-id>
doctl compute volume-action detach <volume-id> <droplet-id>
doctl compute volume-action resize <volume-id> --size 200GiB --region nyc3
doctl compute volume list
doctl compute volume delete <id>
```

After attaching, the volume appears at `/dev/disk/by-id/scsi-0DO_Volume_<name>`. Mount and add to `/etc/fstab` from inside the droplet.

## Tagging

```bash
doctl compute tag create prod
doctl compute tag list
doctl compute droplet tag <droplet-id> --tag-name prod
doctl compute droplet untag <droplet-id> --tag-name prod
```

Tags are the cleanest way to scope bulk operations and firewall rules.
