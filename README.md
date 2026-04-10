# dns-mgr

**Unified DNS · Traefik · Certificate Management**

A single Bash script that replaces Plesk by automatically connecting HestiaCP, PowerDNS, Traefik and Mailcow — no hooks, no manual configuration.

> **Language:** This tool is primarily documented in German as it targets German-speaking server administrators replacing Plesk. English documentation is planned.

---

## Was macht dns-mgr?

Wenn du eine Domain in HestiaCP anlegst passiert automatisch:

- PowerDNS-Zone mit korrekten SOA, NS, A, AAAA Records wird angelegt
- Traefik-Route wird erstellt (inkl. Let's Encrypt Zertifikat per DNS-01)
- Bei Mail-Domains: MX, SPF, DKIM, DMARC, MTA-STS, TLSRPT, SRV werden gesetzt
- Mailcow SAN-Zertifikat wird automatisch aktualisiert
- Zertifikate werden automatisch an Mailcow und andere Dienste deployt

Kein manueller Eingriff nötig.

---

## Architektur

```
Internet
    ↓
Traefik (Reverse Proxy + ACME DNS-01)
    ↓
HestiaCP (Web-Hosting)    Mailcow (Mail)
    ↓
PowerDNS (Autoritativer NS)
    ↓
Hetzner Secondary DNS + eigener NS2
```

### Vorausgesetzte Infrastruktur

| Dienst | Empfohlene IP |
|---|---|
| Traefik + dns-mgr | 192.168.1.120 |
| HestiaCP | 192.168.1.118 |
| PowerDNS + Admin | 192.168.1.119 |
| Mailcow | 192.168.1.106 |

---

## Voraussetzungen

- Debian/Ubuntu LXC oder VM
- PowerDNS Authoritative Server mit MySQL-Backend und API aktiviert
- Traefik v2+ mit DNS-01 ACME (pdns Provider)
- HestiaCP mit aktivierter REST-API
- Mailcow (optional, für Mail-Domains)

### Abhängigkeiten installieren

```bash
apt install curl jq openssl inotify-tools openssh-client
```

---

## Installation

```bash
# 1. Script installieren
install -m 750 -o root -g root dns-mgr /usr/local/bin/dns-mgr

# 2. Konfigurationsverzeichnis anlegen
mkdir -p /etc/dns-mgr
chmod 700 /etc/dns-mgr

# 3. Beispiel-Config generieren
dns-mgr init-config

# 4. Config befüllen
nano /etc/dns-mgr/dns-mgr.conf
chmod 600 /etc/dns-mgr/dns-mgr.conf

# 5. systemd-Service für Zertifikat-Deployment installieren
install -m 644 watch-cert.service /etc/systemd/system/
systemctl daemon-reload
# Erst starten wenn erste Zertifikate ausgestellt sind:
# systemctl enable --now watch-cert

# 6. Cron-Jobs einrichten
crontab -e
# * * * * * /usr/local/bin/dns-mgr hestia-sync  >> /var/log/dns-mgr-hestia.log 2>&1
# * * * * * /usr/local/bin/dns-mgr mailcow-sync >> /var/log/dns-mgr-mailcow.log 2>&1
```

---

## Konfiguration

Alle Einstellungen in `/etc/dns-mgr/dns-mgr.conf`:

```bash
# PowerDNS API
PDNS_API="http://192.168.1.119:8081/api/v1"
PDNS_KEY="DEIN-POWERDNS-API-KEY"

# Traefik
TRAEFIK_CONF_DIR="/etc/traefik/conf.d"
ACME_JSON="/etc/traefik/acme.json"
ACME_RESOLVER="letsencrypt"

# Mailcow
MAILCOW_IP="192.168.1.106"
MAILCOW_KEY="DEIN-MAILCOW-API-KEY"
MAILCOW_IPV6="2a01:XXXX::106"

# HestiaCP
HESTIA_IP="192.168.1.118"
HESTIA_USER="admin"
HESTIA_ACCESS_KEY="DEIN-ACCESS-KEY"
HESTIA_SECRET_KEY="DEIN-SECRET-KEY"

# Öffentliche Adressen
PUBLIC_IPV4="DEINE-IPv4"
IPV6_PREFIX="2a01:XXXX:XXXX:XXXX"

# Nameserver
MAIL_HOSTNAME="mail.DEINE-DOMAIN.tld"
SOA_HOSTMASTER="hostmaster.DEINE-DOMAIN.tld."
NS_RECORDS=(
  "ns1.DEINE-DOMAIN.tld."
  "ns2.DEINE-DOMAIN.tld."
  "helium.ns.hetzner.de."
)

# Zertifikat-Deployment
declare -A CERT_TARGETS
CERT_TARGETS=(
  [mailcow]="192.168.1.106:/opt/mailcow/.../cert.pem:/opt/mailcow/.../key.pem:docker compose restart ...:mail.DEINE-DOMAIN.tld"
)
```

---

## Verwendung

### Domains verwalten

```bash
# Web-Domain anlegen (DNS + Traefik → HestiaCP):
dns-mgr add-web example.com

# Web-Domain mit Mail:
dns-mgr add-web example.com --mail

# Mail-Records nachträglich setzen:
dns-mgr add-mail example.com

# Beliebigen Dienst anlegen:
dns-mgr add-service subdomain.example.com 192.168.1.100 8080 --ipv6=::100
dns-mgr add-service admin.example.com 192.168.1.1 443 --ipv6=::120 --insecure
dns-mgr add-service mumble.example.com 192.168.1.123 64738 --ipv6=::123 --no-traefik
dns-mgr add-service phpmyadmin.example.com 192.168.1.118 80 --prefix=/phpmyadmin
dns-mgr add-service intern.example.com 192.168.1.100 80 --basic-auth="admin:$(htpasswd -nb admin passwort | cut -d: -f2)"
dns-mgr add-service old.example.com 192.168.1.100 80 --redirect=https://new.example.com

# SRV-Record setzen:
dns-mgr add-srv example.com mumble tcp 0 10 64738 mumble.example.com

# Domain entfernen:
dns-mgr remove example.com
dns-mgr remove example.com --force
```

### Synchronisation

```bash
# HestiaCP-Domains prüfen (neue/entfernte erkennen):
dns-mgr hestia-sync

# Mailcow-Domains prüfen:
dns-mgr mailcow-sync

# MTA-STS IDs aus Mailcow-Datenbank synchronisieren:
dns-mgr update-mta-sts
```

### Zertifikate

```bash
# Alle Zertifikate anzeigen:
dns-mgr cert-list

# Zertifikate an alle Targets deployen:
dns-mgr deploy-certs

# Nur ein bestimmtes Target:
dns-mgr deploy-certs --target mailcow

# Zertifikat-Watcher starten (inotify auf acme.json):
dns-mgr watch-certs
```

### Übersicht

```bash
dns-mgr list
```

---

## add-service Optionen

| Option | Beschreibung |
|---|---|
| `--ipv6=::CTID` | IPv6-Suffix, CTID oder vollständige Adresse |
| `--no-dns` | Nur Traefik-Route, kein DNS |
| `--no-ipv6` | Nur A-Record, kein AAAA |
| `--no-traefik` | Nur DNS, keine Traefik-Route (für direkte NAT-Dienste) |
| `--insecure` | Backend-Zertifikat nicht prüfen (pfSense, Proxmox etc.) |
| `--type=tcp` | TCP statt HTTP |
| `--tcp-listen-port=N` | Pflicht bei TCP |
| `--subdomain-of=zone` | Zone explizit angeben |
| `--prefix=/pfad` | Pfad dem Request voranstellen (z.B. `/phpmyadmin`) |
| `--strip-prefix=/pfad` | Pfad aus dem Request entfernen bevor er ans Backend geht |
| `--basic-auth=user:hash` | HTTP Basic Authentication vorschalten (Hash via `htpasswd -nb user pw`) |
| `--redirect=https://ziel` | URL-Weiterleitung zur Ziel-URL |
| `--internal` | DNS zeigt auf interne Backend-IP — kein Traefik, nur über VPN erreichbar |
| `--extra-path=/pfad:ip:port[:insecure]` | Zusätzlichen Pfad mit separatem Backend zur Route hinzufügen |

---

## Zertifikat-Deployment (CERT_TARGETS)

Format: `[name]="host:cert-pfad:key-pfad:reload-cmd:main-domain[:owner:group:cert-mode:key-mode]"`

Die letzten vier Felder `owner`, `group`, `cert-mode` und `key-mode` sind optional.
Standard: `root:root:0644:0600`

```bash
CERT_TARGETS=(
  # Mailcow
  [mailcow]="192.168.1.106:/opt/mailcow/data/assets/ssl/cert.pem:/opt/mailcow/data/assets/ssl/key.pem:cd /opt/mailcow && docker compose restart nginx-mailcow:mail.example.com"

  # Mumble — mit korrektem Owner und Rechten damit mumble-server die Datei lesen kann
  [mumble]="192.168.1.123:/etc/mumble/certs/mumble.example.com/fullchain.pem:/etc/mumble/certs/mumble.example.com/privkey.pem:systemctl restart mumble-server:mumble.example.com:root:mumble-server:0640:0640"

  # Lokales Deployment (kein SSH):
  [lokal]=":/pfad/cert.pem:/pfad/key.pem::subdomain.example.com"
)
```

---

## PowerDNS Konfiguration

```ini
# /etc/powerdns/pdns.conf
primary=yes
secondary=no
api=yes
api-key=DEIN-KEY
webserver=yes
webserver-address=192.168.1.119
webserver-port=8081
webserver-allow-from=192.168.1.120,127.0.0.1

# Secondary NS per NOTIFY informieren:
also-notify=213.239.242.238,213.133.100.103,193.47.99.3
```

---

## HestiaCP API einrichten

```bash
# Auf HestiaCP-Server:
cat > /usr/local/hestia/data/api/dns-mgr << 'EOF'
ROLE=admin
COMMANDS=v-list-users,v-list-web-domains
EOF

v-add-access-key admin dns-mgr dns-mgr json
```

IP `192.168.1.120` in HestiaCP unter Einstellungen → Sicherheit → Erlaubte IPs whitelisten.

---

## Traefik Konfiguration

```yaml
# /etc/traefik/traefik.yml
certificatesResolvers:
  letsencrypt:
    acme:
      email: deine@email.de
      storage: /etc/traefik/acme.json
      dnsChallenge:
        provider: pdns
        resolvers:
          - "192.168.1.119:53"
```

Umgebungsvariablen für den pdns-Provider:

```bash
# /etc/traefik/traefik.env
PDNS_API_URL=http://192.168.1.119:8081
PDNS_API_KEY=DEIN-POWERDNS-API-KEY
```

---

## Lizenz

Copyright (C) 2026 Andreas P.

Dieses Programm ist freie Software: Sie können es unter den Bedingungen der
GNU Affero General Public License, wie von der Free Software Foundation
veröffentlicht, weitergeben und/oder modifizieren, entweder gemäß Version 3
der Lizenz oder (nach Ihrer Wahl) jeder neueren Version.

Siehe [LICENSE](LICENSE) für den vollständigen Lizenztext.
