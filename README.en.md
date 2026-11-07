# dns-mgr

🇬🇧 English · [🇩🇪 Deutsch](README.md)

**Unified DNS · Traefik · Certificate Management**

A single Bash script that replaces Plesk by automatically connecting HestiaCP, PowerDNS, Traefik and Mailcow — no hooks, no manual configuration.

> **Which parts do I need?** The core is **PowerDNS + Traefik**: `add-service`, `edit`, `add-srv`, `enable-dnssec`, HSTS and certificate deployment work with just those two. **HestiaCP** (`add-web`, `hestia-sync`), **Mailcow** (`add-mail`, `mailcow-sync`) and **pfSense** (split DNS) are optional integrations — if you don't use them, leave their config fields empty and simply don't run the related commands or cron jobs. Without `PFSENSE_HOST`, split DNS is skipped automatically.

---

## What does dns-mgr do?

When you create a domain in HestiaCP, the following happens automatically:

- A PowerDNS zone with correct SOA, NS, A and AAAA records is created
- A Traefik route is created (including a Let's Encrypt certificate via DNS-01)
- For mail domains: MX, SPF, DKIM, DMARC, MTA-STS, TLSRPT and SRV records are set
- The Mailcow SAN certificate is updated automatically
- Certificates are deployed automatically to Mailcow and other services
- *(optional)* a pfSense Unbound host override is set → split DNS

No manual steps required.

---

## Architecture

```
Internet
    ↓
Traefik (reverse proxy + ACME DNS-01)
    ↓
HestiaCP (web hosting)    Mailcow (mail)
    ↓
PowerDNS (authoritative NS)
    ↓
Hetzner secondary DNS + own NS2
```

### Expected infrastructure

| Service | Suggested IP |
|---|---|
| Traefik + dns-mgr | 192.168.1.120 |
| HestiaCP | 192.168.1.118 |
| PowerDNS + Admin | 192.168.1.119 |
| Mailcow | 192.168.1.106 |

---

## Requirements

- Debian/Ubuntu LXC or VM
- PowerDNS Authoritative Server with MySQL backend and API enabled
- Traefik v2+ with DNS-01 ACME (pdns provider)
- HestiaCP with REST API enabled (optional, for automatic web domains)
- Mailcow (optional, for mail domains)

### Install dependencies

```bash
apt install curl jq openssl inotify-tools openssh-client
```

---

## Installation

```bash
# 1. Install the script
install -m 750 -o root -g root dns-mgr /usr/local/bin/dns-mgr

# 2. Create the config directory
mkdir -p /etc/dns-mgr
chmod 700 /etc/dns-mgr

# 3. Generate an example config
dns-mgr init-config

# 4. Fill in the config
nano /etc/dns-mgr/dns-mgr.conf
chmod 600 /etc/dns-mgr/dns-mgr.conf

# 5. Install the systemd service for certificate deployment
install -m 644 watch-cert.service /etc/systemd/system/
systemctl daemon-reload
# Only start it once the first certificates have been issued:
# systemctl enable --now watch-cert

# 6. Set up cron jobs
crontab -e
# */5 * * * * /usr/local/bin/dns-mgr hestia-sync  >> /var/log/dns-mgr-hestia.log 2>&1
# */5 * * * * /usr/local/bin/dns-mgr mailcow-sync >> /var/log/dns-mgr-mailcow.log 2>&1
```

> **Updating:** after a `git pull`, run the `install` command from step 1 again — cron and your shell run the copy in `/usr/local/bin`, not the one in the repository.

---

## Configuration

All settings live in `/etc/dns-mgr/dns-mgr.conf`:

```bash
# PowerDNS API
PDNS_API="http://192.168.1.119:8081/api/v1"
PDNS_KEY="YOUR-POWERDNS-API-KEY"

# Traefik
TRAEFIK_CONF_DIR="/etc/traefik/conf.d"
ACME_JSON="/etc/traefik/acme.json"
ACME_RESOLVER="letsencrypt"

# Mailcow
MAILCOW_IP="192.168.1.106"
MAILCOW_KEY="YOUR-MAILCOW-API-KEY"
MAILCOW_IPV6="2a01:XXXX::106"

# HestiaCP
HESTIA_IP="192.168.1.118"
HESTIA_USER="admin"
HESTIA_ACCESS_KEY="YOUR-ACCESS-KEY"
HESTIA_SECRET_KEY="YOUR-SECRET-KEY"

# Public addresses
PUBLIC_IPV4="YOUR-IPv4"
IPV6_PREFIX="2a01:XXXX:XXXX:XXXX"

# Name servers
MAIL_HOSTNAME="mail.YOUR-DOMAIN.tld"
SOA_HOSTMASTER="hostmaster.YOUR-DOMAIN.tld."
NS_RECORDS=(
  "ns1.YOUR-DOMAIN.tld."
  "ns2.YOUR-DOMAIN.tld."
  "helium.ns.hetzner.de."
)

# Certificate deployment
declare -A CERT_TARGETS
CERT_TARGETS=(
  [mailcow]="192.168.1.106:/opt/mailcow/.../cert.pem:/opt/mailcow/.../key.pem:docker compose restart ...:mail.YOUR-DOMAIN.tld"
)

# pfSense split DNS (optional)
PFSENSE_HOST="192.168.1.1"
PFSENSE_USER="admin"
PFSENSE_PASS="your-password"

# HSTS (optional)
HSTS_MAX_AGE="31536000"   # start with 300 while testing
HSTS_SUBDOMAINS=0
HSTS_PRELOAD=0
```

---

## Usage

### Managing domains

```bash
# Create a web domain (DNS + Traefik → HestiaCP):
dns-mgr add-web example.com

# Web domain with mail:
dns-mgr add-web example.com --mail

# Add mail records later:
dns-mgr add-mail example.com

# Create any service:
dns-mgr add-service subdomain.example.com 192.168.1.100 8080 --ipv6=::100
dns-mgr add-service admin.example.com 192.168.1.1 443 --ipv6=::120 --insecure
dns-mgr add-service mumble.example.com 192.168.1.123 64738 --ipv6=::123 --no-traefik
dns-mgr add-service phpmyadmin.example.com 192.168.1.118 80 --prefix=/phpmyadmin
dns-mgr add-service intern.example.com 192.168.1.100 80 --basic-auth="admin:$(htpasswd -nb admin password | cut -d: -f2)"
dns-mgr add-service old.example.com 192.168.1.100 80 --redirect=https://new.example.com

# Change an existing add-service entry (only the given values, everything else is kept):
dns-mgr edit cloud.example.com                         # just show the current config
dns-mgr edit cloud.example.com --port=8081              # change only the port
dns-mgr edit phpmyadmin.example.com --no-prefix         # remove the prefix
dns-mgr edit cp.example.com --extra-path=/phpmyadmin:192.168.1.118:80   # add an extra path
dns-mgr edit cp.example.com --no-extra-path=/phpmyadmin                 # remove an extra path

# Set an SRV record:
dns-mgr add-srv example.com mumble tcp 0 10 64738 mumble.example.com
# e.g. TeamSpeak 3 (_ts3._udp.example.com → ts.example.com:9987):
dns-mgr add-srv example.com ts3 udp 0 5 9987 ts.example.com

# Enable DNSSEC (once; prints the DS record for your registrar):
dns-mgr enable-dnssec example.com

# Remove a domain:
dns-mgr remove example.com
dns-mgr remove example.com --force
```

### Synchronisation

```bash
# Check HestiaCP domains (detect new/removed ones):
dns-mgr hestia-sync

# Check Mailcow domains:
dns-mgr mailcow-sync

# Sync MTA-STS IDs from the Mailcow database:
dns-mgr update-mta-sts

# Split DNS: reconcile all Traefik configs with pfSense Unbound:
dns-mgr sync-split-dns

# Split DNS: show current Unbound host overrides:
dns-mgr list-split-dns
```

### Certificates

```bash
# List all certificates:
dns-mgr cert-list

# Deploy certificates to all targets:
dns-mgr deploy-certs

# Only one specific target:
dns-mgr deploy-certs --target mailcow

# Start the certificate watcher (inotify on acme.json):
dns-mgr watch-certs
```

### Overview

```bash
# All zones with their origin ([hestia]/[mailcow]/[manuell]), Traefik configs, cert targets:
dns-mgr list
```

The origin is derived from the sync state files (`hestia-domains.json`, `mailcow-domains.json` in `$STATE_DIR`): domains known through `hestia-sync`/`mailcow-sync` are marked accordingly — everything else counts as `manuell` (manual, e.g. created with `add-service`).

> The script's log output and messages are in German.

---

## Split DNS (pfSense Unbound)

Domains like `pve.example.com` point to the public IP from the outside — inside the LAN, clients should resolve directly to the backend IP, without NAT loopback. dns-mgr sets Unbound host overrides in pfSense automatically.

### Requirements

No additional package needed — it uses the built-in XML-RPC of pfSense CE and works with the standard admin credentials.

### Configuration

```bash
# Add to /etc/dns-mgr/dns-mgr.conf:
PFSENSE_HOST="192.168.1.1"
PFSENSE_USER="admin"
PFSENSE_PASS="your-password"
```

If `PFSENSE_HOST` is left empty, split DNS is silently skipped for all commands — the feature is entirely optional.

### Usage

```bash
# Automatically on add-service / add-web:
dns-mgr add-service pve.example.com 192.168.1.50 8006 --insecure
# → sets the PowerDNS A record to PUBLIC_IPV4
# → sets the Traefik route
# → sets the pfSense Unbound override: pve.example.com → 192.168.1.50

# Sync existing services afterwards:
dns-mgr sync-split-dns

# Show all current overrides:
dns-mgr list-split-dns

# The override is removed automatically on removal:
dns-mgr remove pve.example.com
```

### How it works

| Client | DNS resolution | Result |
|--------|--------------|---------|
| External (internet) | Public DNS | PUBLIC_IPV4 → Traefik → backend |
| Internal (LAN) | pfSense Unbound | 192.168.1.50 → backend directly |

---

## Notifications

`hestia-sync` and `sync-split-dns` run unattended via cron — errors or automatic cleanups would otherwise only end up in log files nobody reads regularly. Several channels can be enabled at the same time:

```bash
# In /etc/dns-mgr/dns-mgr.conf:
NTFY_URL="https://ntfy.sh/my-secret-topic"       # better: your own ntfy instance instead of ntfy.sh —
NTFY_TOKEN=""                                     # topics on ntfy.sh without auth can be read AND
                                                  # written by anyone who knows the name

NOTIFY_MAIL_TO="admin@example.com"
NOTIFY_MAIL_FROM="dns-mgr@example.com"
NOTIFY_SMTP_URL="smtps://mail.example.com:465"    # e.g. your own Mailcow
NOTIFY_SMTP_USER="dns-mgr@example.com"
NOTIFY_SMTP_PASS="..."

MATRIX_HOMESERVER="https://matrix.example.com"    # your homeserver
MATRIX_TOKEN="..."                                # access token of a bot account
MATRIX_ROOM_ID="!AbCdEfGhIjKlMnOp:example.com"
```

Leaving a field empty disables that channel — with no configuration at all, dns-mgr stays silent as before (logs only). `dns-mgr notify-test` sends a test message through all configured channels.

A notification is sent for:
- unreachable backends in `hestia-sync`/`sync-split-dns` (HestiaCP, pfSense),
- a PowerDNS zone deleted automatically by `hestia-sync` (see below),
- a Traefik `Host()` rule without a matching PowerDNS zone (orphaned config) — only once per domain, until it disappears and shows up again.

### Automatic cleanup of removed HestiaCP domains

When a domain is deleted in HestiaCP, `hestia-sync` cleans up automatically:
- The Traefik config and the pfSense split DNS override are always removed.
- If the domain was a subdomain in a shared zone, only its own A/AAAA records (+`www.`) are deleted — mail records (`mail.*`, MX, …) are never affected.
- If the domain was its own PowerDNS zone, the whole zone is deleted — but **only if it contains nothing but the records created by `add-web`** (SOA/NS, A/AAAA at the apex and `www`). If there are other records (MX, SRV, TXT, other subdomains, …), only the web records are removed and the zone is kept.
- If HestiaCP cannot return a user's domains (timeout, error response), the previous state is kept — a short outage no longer causes domains to be treated as deleted.

---

## HSTS

HTTP Strict Transport Security — browsers remember that a domain is only reachable via HTTPS and skip the HTTP step entirely.

### Requirements

All domains must be HTTPS-ready. Check with:

```bash
cat > /tmp/check-https.sh << 'EOF'
grep -rhoP "Host\(\`\K[^\`]+" /etc/traefik/conf.d/*.yml 2>/dev/null \
| sort -u \
| while read -r domain; do
    http_code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 \
      -H "Host: ${domain}" "http://127.0.0.1/" 2>/dev/null)
    redirect=$(curl -s -o /dev/null -w "%{redirect_url}" --max-time 5 \
      -H "Host: ${domain}" "http://127.0.0.1/" 2>/dev/null)
    if [[ "$redirect" == https://* ]]; then
      printf "OK   %-45s  HTTP %s -> HTTPS\n" "$domain" "$http_code"
    else
      printf "!!   %-45s  HTTP %s - no HTTPS redirect\n" "$domain" "$http_code"
    fi
  done
EOF
bash /tmp/check-https.sh
```

### Enabling

```bash
# In /etc/dns-mgr/dns-mgr.conf:
HSTS_MAX_AGE="300"      # 5 minutes for testing
HSTS_SUBDOMAINS=0
HSTS_PRELOAD=0          # do not enable — practically irreversible

# Add HSTS to all Traefik routes:
dns-mgr enable-hsts

# After a few days, raise it to 1 year:
# HSTS_MAX_AGE="31536000"
# dns-mgr enable-hsts

# Emergency exit:
dns-mgr disable-hsts
```

### About includeSubDomains

`HSTS_SUBDOMAINS=1` means: a browser that visits `example.com` will afterwards enforce HTTPS for all subdomains too, without having visited them. Only enable it if **all** subdomains reliably support HTTPS.

---

## Real IP (X-Forwarded-For)

One-time infrastructure configuration — not part of the dns-mgr script.

**Problem:** PHP and nginx logs see `REMOTE_ADDR = Traefik IP` instead of the real client IP. AWStats and other log analysis tools therefore show only a single IP.

### Traefik (`/etc/traefik/traefik.yml`)

Prevents clients from spoofing `X-Forwarded-For`:

```yaml
entryPoints:
  web:
    address: ":80"
    forwardedHeaders:
      trustedIPs: ["127.0.0.1/32", "::1/128"]
  websecure:
    address: ":443"
    forwardedHeaders:
      trustedIPs: ["127.0.0.1/32", "::1/128"]
```

### HestiaCP nginx (`/etc/nginx/conf.d/cloudflare.inc`)

Add Traefik as a trusted proxy and use `X-Forwarded-For` as the real IP header:

```nginx
# Traefik (local reverse proxy)
set_real_ip_from 192.168.1.120/32;

# ... further set_real_ip_from entries ...

real_ip_header    X-Forwarded-For;
real_ip_recursive on;
```

After `systemctl reload nginx`, PHP and the nginx access logs see the real client IP.

---

## add-service options

| Option | Description |
|---|---|
| `--ipv6=::CTID` | IPv6 suffix, CTID or full address |
| `--no-dns` | Traefik route only, no DNS |
| `--no-ipv6` | A record only, no AAAA |
| `--no-traefik` | DNS only, no Traefik route (for services exposed directly via NAT) |
| `--insecure` | Don't verify the backend certificate (pfSense, Proxmox etc.) |
| `--type=tcp` | TCP instead of HTTP |
| `--tcp-listen-port=N` | Required for TCP |
| `--subdomain-of=zone` | Specify the zone explicitly |
| `--prefix=/path` | Prepend a path to the request (e.g. `/phpmyadmin`) |
| `--strip-prefix=/path` | Strip a path from the request before it reaches the backend |
| `--basic-auth=user:hash` | Put HTTP basic authentication in front (hash via `htpasswd -nb user pw`) |
| `--redirect=https://target` | Redirect to the target URL |
| `--internal` | DNS points to the internal backend IP — no Traefik, reachable via VPN only |
| `--extra-path=/path:ip:port[:insecure]` | Add an extra path with a separate backend to the route (repeatable; inherits HSTS and basic auth of the main route) |

---

## Certificate deployment (CERT_TARGETS)

For the target `mailcow`, every successful deployment automatically updates a `TLSA` record (DANE, `_25._tcp.mail.<domain>`) for all known mail domains from the current SMTP certificate — no separate command needed, it runs as part of `deploy-certs`/`watch-certs`. The target must be named exactly `mailcow` for this.

Format: `[name]="host:cert-path:key-path:reload-cmd:main-domain[:owner:group:cert-mode:key-mode]"`

The last four fields `owner`, `group`, `cert-mode` and `key-mode` are optional.
Default: `root:root:0644:0600`

```bash
CERT_TARGETS=(
  # Mailcow
  [mailcow]="192.168.1.106:/opt/mailcow/data/assets/ssl/cert.pem:/opt/mailcow/data/assets/ssl/key.pem:cd /opt/mailcow && docker compose restart nginx-mailcow:mail.example.com"

  # Mumble — with correct owner and permissions so mumble-server can read the file
  [mumble]="192.168.1.123:/etc/mumble/certs/mumble.example.com/fullchain.pem:/etc/mumble/certs/mumble.example.com/privkey.pem:systemctl restart mumble-server:mumble.example.com:root:mumble-server:0640:0640"

  # Local deployment (no SSH):
  [local]=":/path/cert.pem:/path/key.pem::subdomain.example.com"
)
```

---

## PowerDNS configuration

```ini
# /etc/powerdns/pdns.conf
primary=yes
secondary=no
api=yes
api-key=YOUR-KEY
webserver=yes
webserver-address=192.168.1.119
webserver-port=8081
webserver-allow-from=192.168.1.120,127.0.0.1

# Notify secondary name servers:
also-notify=213.239.242.238,213.133.100.103,193.47.99.3
```

---

## Setting up the HestiaCP API

```bash
# On the HestiaCP server:
cat > /usr/local/hestia/data/api/dns-mgr << 'EOF'
ROLE=admin
COMMANDS=v-list-users,v-list-web-domains
EOF

v-add-access-key admin dns-mgr dns-mgr json
```

Whitelist the IP `192.168.1.120` in HestiaCP under Settings → Security → Allowed IP addresses.

---

## Traefik configuration

```yaml
# /etc/traefik/traefik.yml
certificatesResolvers:
  letsencrypt:
    acme:
      email: you@example.com
      storage: /etc/traefik/acme.json
      dnsChallenge:
        provider: pdns
        resolvers:
          - "192.168.1.119:53"
```

Environment variables for the pdns provider:

```bash
# /etc/traefik/traefik.env
PDNS_API_URL=http://192.168.1.119:8081
PDNS_API_KEY=YOUR-POWERDNS-API-KEY
```

---

## License

Copyright (C) 2026 Andreas P.

This program is free software: you can redistribute it and/or modify it under
the terms of the GNU Affero General Public License as published by the Free
Software Foundation, either version 3 of the License, or (at your option) any
later version.

See [LICENSE](LICENSE) for the full license text.
