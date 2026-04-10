# Changelog

Alle wichtigen Änderungen an dns-mgr werden in dieser Datei dokumentiert.

Das Format basiert auf [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
das Projekt folgt [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
