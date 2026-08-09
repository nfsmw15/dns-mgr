# Changelog

Alle wichtigen Änderungen an dns-mgr werden in dieser Datei dokumentiert.

Das Format basiert auf [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
das Projekt folgt [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.6.3] - 2026-08-09

### Behoben
- `remove_traefik_config`: Die Schleife nutzte `[[ -f "$f" ]] && rm -f "$f" && log ...` als letzte Anweisung. Existierte die zuletzt geprüfte Datei (typischerweise `<name>-tcp.yml`, die fast nie existiert) nicht, lieferte `[[ -f "$f" ]]` Exit-Code 1 — und da das der letzte ausgeführte Befehl der Funktion war, gab auch die Funktion selbst 1 zurück. Wegen `set -euo pipefail` brach `hestia-sync` dadurch bei **jedem** entfernten HestiaCP-Domain sofort ab, bevor `$HESTIA_STATE` aktualisiert wurde — mit der Folge, dass die State-Datei dauerhaft veraltet blieb (nachweislich seit 2026-05-01 unverändert), bei jedem Sync-Lauf dieselben längst existierenden Domains erneut als "neu" erkannt und komplett neu provisioniert wurden, und dadurch unnötig SOA-Serials hochgezählt sowie NOTIFY/AXFR an alle Secondaries ausgelöst wurden. `remove_traefik_config` gibt jetzt zuverlässig `0` zurück, unabhängig davon ob die geprüften Dateien existieren.

## [0.6.2] - 2026-08-09

### Geändert
- `mailcow-sync`: Verbindungstest und Domainliste riefen bisher zweimal identisch `GET /get/domain/all` auf (einmal nur für den HTTP-Status, einmal für den Body). Zusammengefasst zu einem einzigen Request — Fehlerbehandlung (nicht erreichbar / falscher Key / ungültige Antwort) unverändert, jetzt explizit statt sich auf den `curl -f`-Seiteneffekt des entfernten zweiten Requests zu verlassen.
- Empfohlenes und tatsächlich einzurichtendes Cron-Intervall für `mailcow-sync` von `* * * * *` auf `*/5 * * * *` reduziert (README) — analog zu `hestia-sync` in 0.6.1.
- Lastabschätzung: vorher 1.440 Läufe/Tag × 2 Requests = 2.880 Requests/Tag, nachher 288 Läufe/Tag × 1 Request = 288 Requests/Tag — Reduktion um 90%. Die State-Datei `mailcow-domains.json` wird dadurch ebenfalls nur noch 288 statt 1.440 Mal täglich geschrieben.

## [0.6.1] - 2026-08-09

### Geändert
- `hestia-sync`: Verbindungstest und User-Abfrage riefen bisher zweimal identisch `v-list-users` auf (einmal nur für den HTTP-Status, einmal für den Body). Zusammengefasst zu einem einzigen Request — halbiert die Hestia-API-Last pro Sync-Lauf, Fehlerbehandlung (nicht erreichbar / HTTP-Fehler / ungültiges JSON) unverändert.
- Empfohlenes Cron-Intervall für `hestia-sync` von `* * * * *` auf `*/5 * * * *` reduziert (README + eingebauter Hilfetext) — auf einem produktiven HestiaCP-Server führte die minütliche Abfrage (inkl. Hestia-interner `v-check-access-key`/Sudo-Aufrufe pro User) zu unnötig hoher journald-Schreiblast. `mailcow-sync` bleibt unverändert bei `* * * * *`.

## [0.6.0] - 2026-08-09

### Hinzugefügt
- **DANE/TLSA für Mailcow-SMTP**: `deploy-certs`/`watch-certs` berechnen bei jedem erfolgreichen Deployment des `mailcow`-Cert-Targets automatisch einen `TLSA 3 1 1`-Record (DANE-EE, SPKI, SHA-256 — RFC 7671) aus dem aktuellen SMTP-Zertifikat und tragen ihn unter `_25._tcp.mail.<domain>` für jede bekannte Mail-Domain ein. Läuft komplett im bestehenden Zertifikats-Automatismus mit, keine manuelle Pflege bei Zertifikatserneuerung nötig. Wirkungslos (aber unschädlich) auf Zonen ohne aktives DNSSEC, da DANE-Validierung laut RFC 7672 eine DNSSEC-validierte Zone voraussetzt — kann also gefahrlos für alle Mail-Domains gepflegt werden, auch bevor DNSSEC dort aktiv ist.
- Neue Hilfsfunktion `publish_mailcow_tlsa`: läuft nur nach tatsächlich erfolgreichem Zertifikats-Deployment (nicht bei SSH/SCP-Fehlern), damit der TLSA-Record nie auf ein noch nicht live deploytes Zertifikat zeigt — DANE ist fail-closed, ein vorzeitiger TLSA-Eintrag wäre schlimmer als gar keiner.

## [0.5.1] - 2026-08-09

### Behoben
- `enable-dnssec`: PowerDNS liefert `dnskey`/`ds` nur als reine RDATA ohne Owner/TTL/Class/Type. Ausgabe erweitert um das volle DNSKEY-RR im Presentation-Format — die meisten Registrare (u.a. INWX) erwarten primär das DNSKEY und berechnen den DS-Record selbst daraus.

### Hinzugefügt
- Neuer Befehl `fix-soa <domain>`: Korrigiert die SOA einer bestehenden Zone auf den konfigurierten Wert. Betrifft Zonen, die vor dns-mgr existierten (z.B. per `add-mail` nur ergänzt statt über `create_zone` angelegt) und dadurch noch PowerDNS' Platzhalter-SOA `a.misconfigured.dns.server.invalid.` tragen.

## [0.5.0] - 2026-08-09

### Hinzugefügt
- Neuer Befehl `enable-dnssec <domain>`: Aktiviert DNSSEC-Signing für eine PowerDNS-Zone (Combined Signing Key, Algorithmus ECDSAP256SHA256) über die PowerDNS-API und gibt den fertigen DS-Record zum manuellen Eintrag beim Registrar (z.B. INWX) aus. Bewusst ein einmaliger, manuell angestoßener Befehl ohne Cron-Anbindung — PowerDNS rotiert Keys nicht automatisch, der DS-Record bleibt nach dem einmaligen Eintrag stabil, solange kein manuelles Key-Rollover erfolgt. Erkennt bereits aktive Keys und zeigt in dem Fall nur deren DS-Record erneut an, statt einen weiteren Key anzulegen.

## [0.4.4] - 2026-08-08

### Behoben
- `rebuild_mailcow_san_route`/`write_traefik_http_san`: Der SAN-Sammelrouter `mailcow-all` setzte kein `serversTransport: insecure`. Da das Mailcow-Backend per IP (`https://192.168.1.106:443`) statt Hostname angesprochen wird, sendet Traefik kein SNI (bei IP-Zielen laut RFC 6066 nicht zulässig), Mailcows nginx liefert dadurch das falsche/Default-Zertifikat aus, Traefik lehnt die Backend-Verbindung ab → `Internal Server Error`, obwohl der direkte Zugriff auf Mailcow funktioniert. `write_traefik_http_san` unterstützt jetzt wie `write_traefik_http` ein `--insecure`-Flag, `rebuild_mailcow_san_route` nutzt es. Zusätzlich fehlten dem SAN-Router die `middlewares` (`https-forward`, `hsts`) der Einzelrouter (`mail-<domain>`) — beide Router matchen dieselben Hosts und müssen sich daher identisch verhalten, unabhängig davon welcher bei Traefik den Zuschlag bekommt.

### Technisch
- Neue Hilfsfunktion `atomic_write_traefik`: Traefik-Configs (`write_traefik_http`, `write_traefik_http_san`) werden jetzt über Temp-Datei + `mv` atomar geschrieben statt per `cat > datei.yml`, inkl. optionaler YAML-Validierung (falls `python3`+`pyyaml` vorhanden) vor dem Ersetzen — verhindert dass Traefiks `watch: true` eine angeschnittene Datei einliest.

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
