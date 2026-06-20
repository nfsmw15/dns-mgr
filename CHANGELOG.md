# Changelog

Alle wichtigen Änderungen an dns-mgr werden in dieser Datei dokumentiert.

Das Format basiert auf [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
das Projekt folgt [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.4.3] - 2026-06-20

### Behoben
- `cert_deploy_target`: Mailcow legt für SNI-Verbindungen pro Domain einen eigenen Zertifikatspfad an (`data/assets/ssl/<domain>/cert.pem`), der unabhängig vom global per `CERT_TARGETS` deployten `cert.pem`/`key.pem` ist und von Dovecot/Postfix bei einer SNI-Verbindung bevorzugt wird. Dieser Pfad wurde bisher nie aktualisiert, wodurch Mailcow live ein nie erneuertes, irgendwann abgelaufenes Zertifikat auslieferte — unabhängig davon, wie aktuell der global deployte Cert war. Nach jedem Deploy wird jetzt automatisch ein Symlink von `<main_domain>/cert.pem` bzw. `key.pem` auf den globalen Cert/Key gesetzt, falls dieses Verzeichnis auf dem Zielhost existiert.

## [0.4.2] - 2026-06-20

### Behoben
- `cert_deploy_target`: Fehlschläge bei `scp`/`ssh` zum Zielhost (z.B. nicht erreichbar) wurden bisher fälschlich als `Unverändert — kein Neustart` gemeldet und in der Gesamtsumme als erfolgreich gezählt. Ursache: `set -e` greift nicht innerhalb von `if`-Bedingungen, daher lief die Funktion nach einem fehlgeschlagenen Transfer einfach weiter und landete im "kein Update nötig"-Zweig. Exit-Codes von `scp`/`ssh` werden jetzt explizit geprüft; bei Fehlschlag wird gewarnt und das Target als übersprungen gezählt.

## [0.4.1] - 2026-06-20

### Behoben
- `watch-certs`: `moved_to`-Event ergänzt — Traefik/lego schreibt `acme.json` atomar (Temp-Datei schreiben, dann `rename()` auf den finalen Namen). Der Watcher horchte bisher nur auf `close_write`, das ausschließlich auf den Temp-Dateinamen feuert und vom Filter verworfen wird. Dadurch wurde das eigentlich relevante Event nie empfangen und neue Zertifikate nie automatisch deployt.

## [0.4.0] - 2026-05-28

### Hinzugefügt
- **Split-DNS über pfSense Unbound** (optional): Beim Anlegen eines Dienstes mit `add-web` oder `add-service` wird automatisch ein Unbound Host Override in pfSense gesetzt, sodass LAN-Clients die Domain direkt zur internen Backend-IP auflösen — ohne NAT-Loopback.
- Neuer Befehl `sync-split-dns`: Liest alle bestehenden Traefik-Configs aus `TRAEFIK_CONF_DIR` und gleicht die Unbound Host Overrides in pfSense vollständig ab (hinzufügen, aktualisieren, veraltete entfernen). Ideal um bestehende Dienste nachzurüsten.
- Neuer Befehl `list-split-dns`: Zeigt alle aktuellen Unbound Host Overrides in pfSense tabellarisch an.
- Drei neue Config-Variablen: `PFSENSE_HOST`, `PFSENSE_USER`, `PFSENSE_PASS` — das Feature wird stillschweigend übersprungen wenn `PFSENSE_HOST` leer ist, kein Pflichtfeld.
- `remove` löscht den zugehörigen pfSense Host Override automatisch mit.
- **HSTS (HTTP Strict Transport Security)**: Neuer Befehl `enable-hsts` schreibt eine gemeinsame Traefik-Middleware und trägt sie in alle bestehenden Routen ein. Neuer Befehl `disable-hsts` als Notausstieg.
- Drei neue Config-Variablen: `HSTS_MAX_AGE`, `HSTS_SUBDOMAINS`, `HSTS_PRELOAD` — leer = deaktiviert.
- Neuer Befehl `check-https` (als Skript): Prüft alle Traefik-Domains ob HTTP korrekt auf HTTPS weiterleitet — Voraussetzung vor HSTS-Aktivierung.

### Technisch
- Split-DNS nutzt das eingebaute XML-RPC von pfSense CE (`/xmlrpc.php`) — kein zusätzliches Paket nötig. Authentifizierung über Admin-User und Passwort.
- HSTS unterstützt beide Traefik YAML-Formate (inline `[middleware]` und Listenformat `- middleware`).

### Infrastruktur-Setup (einmalig, nicht im Script)
- **Traefik** (`traefik.yml`): `forwardedHeaders.trustedIPs` auf Localhost gesetzt — verhindert dass Clients `X-Forwarded-For` Headers fälschen können.
- **HestiaCP nginx** (`/etc/nginx/conf.d/cloudflare.inc`): Traefik-IP als vertrauenswürdiger Proxy eingetragen, `real_ip_header` auf `X-Forwarded-For` umgestellt. PHP und nginx-Logs sehen ab sofort die echte Client-IP statt der Traefik-IP — AWStats und andere Log-Auswertungen funktionieren wieder korrekt.

## [0.3.0] - 2026-04-10

### Hinzugefügt
- `add-service` unterstützt jetzt zusätzliche Traefik-Middleware-Optionen:
  - `--prefix=/pfad`: Pfad dem Request voranstellen (z.B. `/phpmyadmin`)
  - `--strip-prefix=/pfad`: Pfad aus dem Request entfernen bevor er ans Backend geht
  - `--basic-auth=user:hash`: HTTP Basic Authentication vorschalten
  - `--redirect=https://ziel`: URL-Weiterleitung
  - `--internal`: DNS zeigt auf interne Backend-IP, kein Traefik — nur über VPN erreichbar (z.B. Drucker, interne Dienste)
  - `--extra-path=/pfad:ip:port[:insecure]`: Zusätzlichen Pfad mit separatem Backend zur bestehenden Route hinzufügen (z.B. phpMyAdmin unter cp.domain.tld/phpmyadmin)
- `CERT_TARGETS` unterstützt jetzt optionale Felder für Eigentümer und Dateiberechtigungen: `owner:group:cert-mode:key-mode` — damit können Zertifikate direkt mit den korrekten Rechten deployt werden (z.B. `root:mumble-server:0640:0640` für Mumble)

### Behoben
- `remove`: URL wird automatisch bereinigt — `https://`, `http://` und trailing `/` werden entfernt
- `deploy-certs`: Dateiberechtigungen und Eigentümer werden nach dem Kopieren korrekt gesetzt — vorher wurden Zertifikate immer als `root:root` mit `0644`/`0600` deployt

## [0.2.0] - 2026-04-10

### Hinzugefügt
- Neuer Befehl `update-mta-sts`: Liest MTA-STS IDs direkt aus der Mailcow-Datenbank per SSH und aktualisiert die PowerDNS TXT-Records automatisch

### Behoben
- Zonen werden jetzt als `Master` statt `Native` angelegt — PowerDNS sendet dadurch automatisch NOTIFY an alle Secondary Nameserver bei Änderungen (z.B. ACME-Challenge Records). Ohne NOTIFY konnten Let's Encrypt Challenges fehlschlagen, weil Secondary NS die Records nicht rechtzeitig synchronisierten.

## [0.1.0] - 2026-04-08

### Hinzugefügt
- Initiale Veröffentlichung
- Befehle: `add-web`, `add-mail`, `add-service`, `add-srv`, `remove`
- Befehle: `mailcow-sync`, `hestia-sync`
- Befehle: `deploy-certs`, `watch-certs`, `cert-list`, `list`, `init-config`
- PowerDNS API Integration
- Traefik Konfigurationsgenerierung
- Automatisches SAN-Zertifikat für Mailcow
- Let's Encrypt DNS-01 Challenge über PowerDNS
