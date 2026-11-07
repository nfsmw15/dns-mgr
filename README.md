# dns-mgr

[🇬🇧 English](README.en.md) · 🇩🇪 Deutsch

**Unified DNS · Traefik · Certificate Management**

Ein einzelnes Bash-Script, das Plesk ersetzt, indem es HestiaCP, PowerDNS, Traefik und Mailcow automatisch miteinander verbindet — ohne Hooks, ohne manuelle Konfiguration.

> **Was brauche ich davon?** Der Kern ist **PowerDNS + Traefik**: `add-service`, `edit`, `add-srv`, `enable-dnssec`, HSTS und Zertifikat-Deployment funktionieren allein damit. **HestiaCP** (`add-web`, `hestia-sync`), **Mailcow** (`add-mail`, `mailcow-sync`) und **pfSense** (Split-DNS) sind optionale Integrationen — wer sie nicht nutzt, lässt die Felder in der Config leer und ruft die zugehörigen Befehle bzw. Cron-Jobs einfach nicht auf. Ohne `PFSENSE_HOST` wird Split-DNS automatisch übersprungen.

---

## Was macht dns-mgr?

Wenn du eine Domain in HestiaCP anlegst passiert automatisch:

- PowerDNS-Zone mit korrekten SOA, NS, A, AAAA Records wird angelegt
- Traefik-Route wird erstellt (inkl. Let's Encrypt Zertifikat per DNS-01)
- Bei Mail-Domains: MX, SPF, DKIM, DMARC, MTA-STS, TLSRPT, SRV werden gesetzt
- Mailcow SAN-Zertifikat wird automatisch aktualisiert
- Zertifikate werden automatisch an Mailcow und andere Dienste deployt
- *(optional)* pfSense Unbound Host Override wird gesetzt → Split-DNS

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
- HestiaCP mit aktivierter REST-API (optional, für automatische Web-Domains)
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
# */5 * * * * /usr/local/bin/dns-mgr hestia-sync  >> /var/log/dns-mgr-hestia.log 2>&1
# */5 * * * * /usr/local/bin/dns-mgr mailcow-sync >> /var/log/dns-mgr-mailcow.log 2>&1
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

# pfSense Split-DNS (optional)
PFSENSE_HOST="192.168.1.1"
PFSENSE_USER="admin"
PFSENSE_PASS="dein-passwort"

# HSTS (optional)
HSTS_MAX_AGE="31536000"   # nach Testphase mit 300 starten
HSTS_SUBDOMAINS=0
HSTS_PRELOAD=0
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

# Bestehenden add-service-Eintrag ändern (nur angegebene Werte, Rest bleibt erhalten):
dns-mgr edit cloud.example.com                         # nur Anzeige der aktuellen Config
dns-mgr edit cloud.example.com --port=8081              # nur Port ändern
dns-mgr edit phpmyadmin.example.com --no-prefix         # Prefix entfernen

# SRV-Record setzen:
dns-mgr add-srv example.com mumble tcp 0 10 64738 mumble.example.com
# z.B. TeamSpeak 3 (_ts3._udp.example.com → ts.example.com:9987):
dns-mgr add-srv example.com ts3 udp 0 5 9987 ts.example.com

# DNSSEC aktivieren (einmalig, gibt DS-Record für den Registrar aus):
dns-mgr enable-dnssec example.com

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

# Split-DNS: alle Traefik-Configs mit pfSense Unbound abgleichen:
dns-mgr sync-split-dns

# Split-DNS: aktuelle Unbound Host Overrides anzeigen:
dns-mgr list-split-dns
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
# Alle Zonen mit Herkunft ([hestia]/[mailcow]/[manuell]), Traefik-Configs, Cert-Targets:
dns-mgr list
```

Die Herkunft wird aus den Sync-State-Dateien (`hestia-domains.json`, `mailcow-domains.json` in `$STATE_DIR`) abgeleitet: Domains, die per `hestia-sync`/`mailcow-sync` bekannt sind, werden entsprechend markiert — alles andere gilt als `manuell` (z.B. per `add-service` angelegt).

---

## Split-DNS (pfSense Unbound)

Domains wie `pve.nfsmw15.de` zeigen extern auf die öffentliche IP — intern sollen LAN-Clients direkt zur Backend-IP auflösen, ohne NAT-Loopback. dns-mgr setzt automatisch Unbound Host Overrides in pfSense.

### Voraussetzung

Kein zusätzliches Paket nötig — nutzt das eingebaute XML-RPC von pfSense CE. Funktioniert mit Standard-Admin-Zugangsdaten.

### Konfiguration

```bash
# In /etc/dns-mgr/dns-mgr.conf ergänzen:
PFSENSE_HOST="192.168.1.1"
PFSENSE_USER="admin"
PFSENSE_PASS="dein-passwort"
```

Wenn `PFSENSE_HOST` leer bleibt, wird Split-DNS bei allen Befehlen stillschweigend übersprungen — das Feature ist vollständig optional.

### Verwendung

```bash
# Automatisch bei add-service / add-web:
dns-mgr add-service pve.nfsmw15.de 192.168.1.50 8006 --insecure
# → setzt PowerDNS A-Record auf PUBLIC_IPV4
# → setzt Traefik-Route
# → setzt pfSense Unbound Override: pve.nfsmw15.de → 192.168.1.50

# Bestehende Dienste nachträglich synchronisieren:
dns-mgr sync-split-dns

# Alle aktuellen Overrides anzeigen:
dns-mgr list-split-dns

# Beim Entfernen wird der Override automatisch gelöscht:
dns-mgr remove pve.nfsmw15.de
```

### Wie es funktioniert

| Client | DNS-Auflösung | Ergebnis |
|--------|--------------|---------|
| Extern (Internet) | Öffentlicher DNS | PUBLIC_IPV4 → Traefik → Backend |
| Intern (LAN) | pfSense Unbound | 192.168.1.50 → Backend direkt |

---

## Benachrichtigungen

`hestia-sync` und `sync-split-dns` laufen unbeaufsichtigt per Cron — Fehler oder automatische Aufräumaktionen darin bleiben sonst nur in Log-Dateien stehen, die niemand regelmäßig anschaut. Optional lassen sich mehrere Kanäle gleichzeitig aktivieren:

```bash
# In /etc/dns-mgr/dns-mgr.conf:
NTFY_URL="https://ntfy.sh/mein-geheimes-topic"   # besser: eigene ntfy-Instanz statt ntfy.sh —
NTFY_TOKEN=""                                     # Topics auf ntfy.sh sind ohne Auth für jeden
                                                   # lesbar UND beschreibbar, der den Namen kennt

NOTIFY_MAIL_TO="admin@nfsmw15.de"
NOTIFY_MAIL_FROM="dns-mgr@nfsmw15.de"
NOTIFY_SMTP_URL="smtps://mail.nfsmw15.de:465"     # z.B. die eigene Mailcow
NOTIFY_SMTP_USER="dns-mgr@nfsmw15.de"
NOTIFY_SMTP_PASS="..."

MATRIX_HOMESERVER="https://matrix.nfsmw15.de"     # eigener Homeserver
MATRIX_TOKEN="..."                                # Access-Token eines Bot-Accounts
MATRIX_ROOM_ID="!AbCdEfGhIjKlMnOp:nfsmw15.de"
```

Jedes Feld leer lassen deaktiviert den jeweiligen Kanal — ganz ohne Konfiguration bleibt dns-mgr wie bisher stumm (nur Logs).

Ausgelöst wird eine Benachrichtigung bei:
- nicht erreichbaren Backends in `hestia-sync`/`sync-split-dns` (HestiaCP, pfSense),
- einer per `hestia-sync` automatisch gelöschten PowerDNS-Zone (siehe unten),
- einer Traefik-`Host()`-Regel ohne zugehörige PowerDNS-Zone (verwaiste Config) — pro Domain nur einmalig, bis sie verschwindet und erneut auftaucht.

### Automatisches Aufräumen bei entfernten HestiaCP-Domains

Wird eine Domain in HestiaCP gelöscht, räumt `hestia-sync` automatisch auf:
- Traefik-Config und pfSense-Split-DNS-Override werden immer entfernt.
- War die Domain eine Subdomain einer geteilten Zone, werden nur ihre eigenen A/AAAA-Records (+`www.`) gelöscht — Mail-Records (`mail.*`, MX, …) sind davon nie betroffen.
- War die Domain selbst eine eigene PowerDNS-Zone, wird die komplette Zone gelöscht — aber **nur wenn sie ausschließlich die von `add-web` angelegten Records enthält** (SOA/NS, A/AAAA am Apex und `www`). Gibt es weitere Records (MX, SRV, TXT, andere Subdomains, …), werden nur die Web-Records entfernt und die Zone bleibt erhalten.
- Kann HestiaCP die Domains eines Users nicht liefern (Timeout, Fehlerantwort), gilt der bisherige Stand weiter — ein kurzer Aussetzer führt nicht mehr dazu, dass Domains als gelöscht behandelt werden.

---

## HSTS

HTTP Strict Transport Security — Browser merken sich dass eine Domain immer nur über HTTPS erreichbar ist und überspringen den HTTP-Schritt komplett.

### Voraussetzung

Alle Domains müssen HTTPS-ready sein. Prüfen mit:

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
      printf "!!   %-45s  HTTP %s - kein HTTPS-Redirect\n" "$domain" "$http_code"
    fi
  done
EOF
bash /tmp/check-https.sh
```

### Aktivieren

```bash
# In /etc/dns-mgr/dns-mgr.conf:
HSTS_MAX_AGE="300"      # 5 Minuten zum Testen
HSTS_SUBDOMAINS=0
HSTS_PRELOAD=0          # Nicht aktivieren — quasi unwiderruflich

# HSTS in alle Traefik-Routen eintragen:
dns-mgr enable-hsts

# Nach ein paar Tagen auf 1 Jahr erhöhen:
# HSTS_MAX_AGE="31536000"
# dns-mgr enable-hsts

# Notausstieg:
dns-mgr disable-hsts
```

### Hinweis zu includeSubDomains

`HSTS_SUBDOMAINS=1` bedeutet: ein Browser der `example.com` besucht, erzwingt HTTPS danach auch für alle Subdomains ohne sie besucht zu haben. Nur aktivieren wenn **alle** Subdomains zuverlässig HTTPS haben.

---

## Real-IP (X-Forwarded-For)

Einmalige Infrastruktur-Konfiguration — nicht Teil des dns-mgr Scripts.

**Problem:** PHP und nginx-Logs sehen `REMOTE_ADDR = Traefik-IP` statt der echten Client-IP. AWStats und andere Log-Auswertungen zeigen dadurch nur eine einzige IP.

### Traefik (`/etc/traefik/traefik.yml`)

Verhindert dass Clients `X-Forwarded-For` fälschen können:

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

Traefik als vertrauenswürdigen Proxy eintragen und `X-Forwarded-For` als Real-IP Header nutzen:

```nginx
# Traefik (lokaler Reverse Proxy)
set_real_ip_from 192.168.1.120/32;

# ... weitere set_real_ip_from Einträge ...

real_ip_header    X-Forwarded-For;
real_ip_recursive on;
```

Nach `systemctl reload nginx` sehen PHP und die nginx Access Logs die echte Client-IP.

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

Für das Target `mailcow` wird bei jedem erfolgreichen Deployment automatisch ein `TLSA`-Record (DANE, `_25._tcp.mail.<domain>`) für alle bekannten Mail-Domains aus dem aktuellen SMTP-Zertifikat aktualisiert — kein separater Befehl nötig, läuft im `deploy-certs`/`watch-certs`-Automatismus mit. Der Target-Name muss dafür exakt `mailcow` heißen.

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
