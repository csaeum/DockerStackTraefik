# Traefik Deployment Guide

## Deployment auf Live-Server

### 1. Vorbereitung

**Externe Netzwerk erstellen (falls nicht vorhanden):**
```bash
docker network create traefik_proxy_network
```

**Verzeichnisse vorbereiten:**
```bash
# Sicherstellen dass alle Ordner existieren
mkdir -p logs volumes

# Richtige Permissions setzen
chmod 700 volumes
chmod 755 logs
```

---

### 2. Stoppen & Aufräumen

**Traefik stoppen:**
```bash
docker compose down
```

**Optional: Alte Logs löschen (VORSICHT!):**
```bash
# Nur wenn du die Logs nicht brauchst
rm -f logs/*.log
```

---

### 3. Deployment

**Container starten:**
```bash
docker compose up -d
```

**Logs überprüfen:**
```bash
# Live-Logs anschauen
docker compose logs -f traefik

# Nach Fehlern suchen
docker compose logs traefik | grep -i error
docker compose logs traefik | grep -i warn
```

---

### 4. Validierung

#### a) Container-Status prüfen
```bash
docker compose ps
```
Erwartetes Ergebnis: `State: Up`

---

#### b) Dashboard erreichbar?
```bash
curl -I https://traefik.server-erde.web-seo-consulting.eu/dashboard/
```
Erwartetes Ergebnis: `HTTP/2 401` (Auth required) oder `HTTP/2 200` (nach Login)

---

#### c) HTTP → HTTPS Redirect
```bash
curl -I http://traefik.server-erde.web-seo-consulting.eu
```
Erwartetes Ergebnis: `301 Moved Permanently` mit `Location: https://...`

---

#### d) Security Headers
```bash
curl -I https://traefik.server-erde.web-seo-consulting.eu/dashboard/
```
Sollte enthalten:
- `Strict-Transport-Security: max-age=31536000`
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`

---

#### e) Prometheus Metrics
```bash
curl -u traefik-admin:DEIN_PASSWORT https://traefik.server-erde.web-seo-consulting.eu/metrics
```
Erwartetes Ergebnis: Prometheus-Metriken im Text-Format

---

#### f) TLS 1.3 Check
```bash
curl -I --tlsv1.3 https://traefik.server-erde.web-seo-consulting.eu/dashboard/
```
Sollte funktionieren.

```bash
curl -I --tlsv1.2 https://traefik.server-erde.web-seo-consulting.eu/dashboard/
```
Sollte NICHT funktionieren (TLS 1.2 ist deaktiviert).

---

### 5. Häufige Fehler & Lösungen

#### Problem: "Cannot create container: network not found"
**Lösung:**
```bash
docker network create traefik_proxy_network
```

---

#### Problem: "Cannot stat /letsencrypt/Traefik.json: no such file or directory"
**Lösung:**
```bash
mkdir -p volumes
chmod 700 volumes
docker compose up -d
```

---

#### Problem: Dashboard zeigt "Bad Gateway"
**Ursache:** Traefik kann sich selbst nicht finden

**Debug:**
```bash
# Prüfen ob Labels gesetzt sind
docker inspect traefik | grep -A 50 Labels

# Prüfen ob Router erstellt wurden
docker compose logs traefik | grep -i dashboard
```

---

#### Problem: Let's Encrypt schlägt fehl
**Mögliche Ursachen:**
1. Port 80/443 nicht erreichbar von außen
2. Domain zeigt nicht auf Server
3. Rate-Limit von Let's Encrypt erreicht

**Debug:**
```bash
# Logs durchsuchen
docker compose logs traefik | grep -i acme
docker compose logs traefik | grep -i certificate
```

**Temporäre Lösung (Staging CA):**
```yaml
# In traefik.yaml ändern:
caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
```

---

#### Problem: Geo-Blocking funktioniert nicht
**Debug:**
```bash
# Plugin-Status prüfen
docker compose logs traefik | grep -i geoblock

# API-Calls loggen ist aktiviert
docker compose logs traefik | grep geojs.io
```

**Bekanntes Problem:** Bei Start dauert Plugin-Download ~30 Sekunden

---

### 6. Rollback

Falls etwas schief geht:

```bash
# Stoppen
docker compose down

# Alte Config wiederherstellen (Git)
git checkout HEAD -- configs/
git checkout HEAD -- docker-compose.yaml

# Neu starten
docker compose up -d
```

---

### 7. Performance-Check

**Response-Zeiten messen:**
```bash
curl -w "@-" -o /dev/null -s https://traefik.server-erde.web-seo-consulting.eu/dashboard/ <<'EOF'
time_namelookup:  %{time_namelookup}s\n
time_connect:     %{time_connect}s\n
time_appconnect:  %{time_appconnect}s\n
time_pretransfer: %{time_pretransfer}s\n
time_starttransfer: %{time_starttransfer}s\n
time_total:       %{time_total}s\n
EOF
```

**Erwartete Werte:**
- `time_total` < 1s (für Dashboard)
- Compression sollte Response-Size um ~70% reduzieren

---

### 8. Monitoring

**Wichtige Metriken im Dashboard beobachten:**
- https://traefik.server-erde.web-seo-consulting.eu/dashboard/
  - Router-Status (grün = OK)
  - Services-Status (grün = OK)
  - HTTP → HTTPS Redirects sichtbar
  - Middleware-Chain sichtbar

**Prometheus Metriken:**
- https://traefik.server-erde.web-seo-consulting.eu/metrics
  - `traefik_entrypoint_requests_total`
  - `traefik_entrypoint_request_duration_seconds`
  - `traefik_router_requests_total`

---

## Sicherheits-Checkliste nach Deployment

- [ ] Dashboard nur über HTTPS erreichbar
- [ ] BasicAuth für Dashboard aktiv
- [ ] HTTP → HTTPS Redirect funktioniert
- [ ] Security Headers gesetzt (HSTS, X-Frame-Options, etc.)
- [ ] TLS 1.3 only (TLS 1.2 blockiert)
- [ ] Geo-Blocking aktiv (Test mit VPN)
- [ ] Rate-Limiting funktioniert
- [ ] Let's Encrypt Zertifikate werden erneuert
- [ ] Logs werden rotiert
- [ ] `.env` NICHT in Git committed
- [ ] `volumes/Traefik.json` hat Permissions 600

**Permissions prüfen:**
```bash
ls -la volumes/Traefik.json
# Sollte sein: -rw------- (600)

# Falls nicht:
chmod 600 volumes/Traefik.json
```

---

## Nächste Schritte

1. **Andere Projekte migrieren** - Siehe `LABELS-CHECKLIST.md`
2. **Dashboard-Passwort ändern** - Siehe unten
3. **Geo-Blocking anpassen** - Optional
4. **Backup einrichten** - Für `volumes/Traefik.json`

---

## Dashboard-Passwort ändern

**Neues Passwort generieren:**
```bash
# Mit htpasswd (Apache Utils)
htpasswd -nbB traefik-admin "DEIN_NEUES_PASSWORT"

# Oder mit Docker:
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "DEIN_NEUES_PASSWORT"
```

**Output kopieren und in `configs/traefik-dynamic.yaml` einfügen:**
```yaml
dashboard-auth:
  basicAuth:
    users:
      - "traefik-admin:$2y$05$..."  # Hier dein neuer Hash
```

**Traefik neu starten:**
```bash
docker compose restart traefik
```

---

## Support

Bei Problemen:
1. Logs prüfen: `docker compose logs traefik`
2. Config validieren: `docker compose config`
3. Traefik Debug-Mode: In `traefik.yaml` setzen: `level: DEBUG`
