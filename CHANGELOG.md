# Changelog

Alle wichtigen Änderungen an dns-mgr werden in dieser Datei dokumentiert.

Das Format basiert auf [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
das Projekt folgt [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
