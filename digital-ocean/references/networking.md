# Networking

Covers VPCs, firewalls, reserved IPs, load balancers, domains/DNS, and certificates. All commands live under `doctl compute` (with the exception of `doctl network` for newer features) and `doctl vpcs`.

## VPCs (Virtual Private Cloud)

Each region has a default VPC. Create custom VPCs to isolate environments.

```bash
doctl vpcs list
doctl vpcs get <id>
doctl vpcs create --name prod-nyc3 --region nyc3 --ip-range 10.10.0.0/20
doctl vpcs update <id> --name new-name --description "..."
doctl vpcs delete <id>
```

Place a droplet into a VPC at create time with `--vpc-uuid <uuid>`. A droplet's VPC cannot be changed after creation — destroy and recreate.

VPC peering (lets two VPCs talk privately, even across regions):

```bash
doctl vpcs peerings list
doctl vpcs peerings create my-peering --vpc-ids <vpc-a>,<vpc-b>
doctl vpcs peerings delete <peering-id>
```

## Firewalls (cloud firewalls)

Cloud firewalls are stateful packet filters applied to droplets by ID or tag. They are independent from any in-VM firewall (ufw/iptables) and recommended over them for DO.

```bash
doctl compute firewall list
doctl compute firewall get <id>
```

### Create

```bash
doctl compute firewall create \
  --name web-fw \
  --tag-names web \
  --inbound-rules "protocol:tcp,ports:22,sources:addresses:0.0.0.0/0,address:::/0 protocol:tcp,ports:80,sources:addresses:0.0.0.0/0 protocol:tcp,ports:443,sources:addresses:0.0.0.0/0" \
  --outbound-rules "protocol:tcp,ports:all,destinations:addresses:0.0.0.0/0,address:::/0 protocol:udp,ports:all,destinations:addresses:0.0.0.0/0,address:::/0 protocol:icmp,destinations:addresses:0.0.0.0/0,address:::/0"
```

Rule grammar (space-separated rules, comma-separated key:value pairs inside each rule):

- `protocol:tcp|udp|icmp`
- `ports:22` or `ports:8000-9000` or `ports:all`
- `sources:` / `destinations:` with one or more of:
  - `addresses:CIDR` (multiple addresses separated by another `address:` key)
  - `tags:tagname`
  - `droplet_ids:1234`
  - `load_balancer_uids:<uid>`
  - `kubernetes_ids:<uid>`

### Modify membership

```bash
doctl compute firewall add-droplets <fw-id> --droplet-ids 111,222
doctl compute firewall remove-droplets <fw-id> --droplet-ids 111
doctl compute firewall add-tags <fw-id> --tag-names web
doctl compute firewall remove-tags <fw-id> --tag-names web
doctl compute firewall add-rules <fw-id> --inbound-rules "protocol:tcp,ports:5432,sources:tags:db-clients"
doctl compute firewall remove-rules <fw-id> --inbound-rules "protocol:tcp,ports:5432,sources:tags:db-clients"
doctl compute firewall delete <fw-id>
```

Best practice: scope by **tag**, not droplet ID. Then any new droplet with that tag inherits the firewall automatically.

## Reserved IPs (formerly Floating IPs)

A static IPv4 you can move between droplets in the same region — useful for blue/green and failover.

```bash
doctl compute reserved-ip list
doctl compute reserved-ip create --region nyc3
doctl compute reserved-ip get <ip>
doctl compute reserved-ip delete <ip>

doctl compute reserved-ip-action assign <ip> <droplet-id>
doctl compute reserved-ip-action unassign <ip>
doctl compute reserved-ip-action get <ip> <action-id>
```

Legacy aliases `floating-ip` / `floating-ip-action` still work. Reassignment is near-instant; existing connections drop.

IPv6 equivalent for VPC-internal routing: `doctl compute reserved-ipv6 ...` (newer feature; check `doctl compute reserved-ipv6 --help`).

## Load Balancers

```bash
doctl compute load-balancer list
doctl compute load-balancer get <id>
```

### Create

```bash
doctl compute load-balancer create \
  --name web-lb \
  --region nyc3 \
  --vpc-uuid <vpc-uuid> \
  --tag-name web \
  --forwarding-rules "entry_protocol:https,entry_port:443,target_protocol:http,target_port:80,certificate_id:<cert-id> entry_protocol:http,entry_port:80,target_protocol:http,target_port:80" \
  --health-check "protocol:http,port:80,path:/health,check_interval_seconds:10,response_timeout_seconds:5,healthy_threshold:3,unhealthy_threshold:5" \
  --redirect-http-to-https \
  --enable-proxy-protocol
```

Targets can be specified by `--tag-name` (recommended — auto-includes new droplets) or by `--droplet-ids`.

### Modify

```bash
doctl compute load-balancer add-droplets <lb-id> --droplet-ids 111,222
doctl compute load-balancer remove-droplets <lb-id> --droplet-ids 111
doctl compute load-balancer add-forwarding-rules <lb-id> --forwarding-rules "..."
doctl compute load-balancer remove-forwarding-rules <lb-id> --forwarding-rules "..."
doctl compute load-balancer update <lb-id> --name new-name --health-check "..."
doctl compute load-balancer delete <lb-id>
```

Cache the load balancer's public IP from `doctl compute load-balancer get <id> --output json | jq -r '.[0].ip'` and point your DNS A record at it.

## Domains and DNS

DigitalOcean's free authoritative DNS.

```bash
doctl compute domain list
doctl compute domain create example.com --ip-address 203.0.113.10   # creates zone + default A record
doctl compute domain delete example.com
```

### Records

```bash
doctl compute domain records list example.com
doctl compute domain records create example.com \
  --record-type A --record-name www --record-data 203.0.113.10 --record-ttl 300
doctl compute domain records create example.com \
  --record-type CNAME --record-name blog --record-data ghs.googlehosted.com.
doctl compute domain records create example.com \
  --record-type MX --record-name @ --record-data smtp.example.com. --record-priority 10
doctl compute domain records create example.com \
  --record-type TXT --record-name @ --record-data "v=spf1 include:_spf.example.com ~all"
doctl compute domain records update example.com --record-id <id> --record-data <new-value>
doctl compute domain records delete example.com <record-id>
```

CNAME and MX values must end in a trailing dot.

## Certificates

Used by load balancers for HTTPS termination. Two flavors: `lets_encrypt` (DO manages renewal) and `custom` (you supply the cert).

```bash
doctl compute certificate list
doctl compute certificate get <id>

# Let's Encrypt — DigitalOcean must be authoritative for the domain
doctl compute certificate create \
  --name web-cert \
  --type lets_encrypt \
  --dns-names example.com,www.example.com

# Custom (BYOC)
doctl compute certificate create \
  --name web-cert \
  --type custom \
  --certificate-chain-path fullchain.pem \
  --leaf-certificate-path cert.pem \
  --private-key-path privkey.pem

doctl compute certificate delete <id>
```

Reference the returned certificate ID in load-balancer forwarding rules (`certificate_id:...`).

## CDN endpoints

```bash
doctl compute cdn list
doctl compute cdn create --origin <space-name>.<region>.digitaloceanspaces.com --ttl 3600
doctl compute cdn update <id> --custom-domain cdn.example.com --certificate-id <cert-id>
doctl compute cdn flush <id> --files "*"
doctl compute cdn delete <id>
```

## Quick recipes

**Set up a tag-driven pool behind a load balancer:**
```bash
doctl compute tag create web
# Create droplets with --tag-names web
doctl compute firewall create --name web-fw --tag-names web --inbound-rules "..." --outbound-rules "..."
doctl compute load-balancer create --name web-lb --tag-name web --forwarding-rules "..."
```

**Failover a reserved IP:**
```bash
doctl compute reserved-ip-action assign <ip> <new-droplet-id> --wait
```

**List everything in a VPC:**
```bash
doctl compute droplet list --output json | jq '.[] | select(.vpc_uuid=="<uuid>") | {id,name,networks}'
```
