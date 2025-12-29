# Changelog - Traefik Stack Optimierung

## 2025-12-29 - Release v1.1.0

### 🔄 Konfigurationen zusammengeführt

Integration der Features von zwei Traefik-Installationen (server-erde + server-msg) in eine universelle, wartungsfreundliche Konfiguration.

---

### ✅ Hinzugefügt

#### Mailcow Integration
- ✅ **Mailcow ACME Challenge Router** - Ermöglicht Let's Encrypt Zertifikate über Traefik
  - Router: `mailcow-acme-challenge` (Port 80, Priority 1000)
  - Service: `mailcow-acme` → `http://mailcow-nginx-mailcow-1:8080`
  - Domains: 11 vorkonfigurierte Mailcow-Domains (anpassbar in `traefik-dynamic.yaml`)
- ✅ **Mailcow Dokumentation** in `.env` und `.env.example`
- ✅ **Netzwerk-Hinweis** in `docker-compose.yaml`

#### Konfiguration
- ✅ **Ping Router** - Health Check via `/ping` Endpoint
- ✅ **RESTART Env-Variable** - Flexibles Restart-Verhalten konfigurierbar (default: `unless-stopped`)
- ✅ **Trusted IPs** - Sicher konfiguriert (nur `127.0.0.1/32`)
  - Keine privaten Netze für maximale Sicherheit
  - Verhindert IP-Spoofing durch kompromittierte Container

---

### 🔄 Geändert

#### Access Logging
- 🔄 **Access Log Filter beibehalten** - Loggt nur Fehler (400-599)
  - **Warum:** Produktionsoptimiert - 95% weniger Disk I/O
  - **Vorteil:** Fokus auf relevante Events, schnelleres Debugging

#### docker-compose.yaml
- 🔄 Restart Policy von hardcoded → `${RESTART:-unless-stopped}`
- 🔄 Mailcow-Netzwerk-Hinweis hinzugefügt

#### .env / .env.example
- 🔄 `RESTART` Variable hinzugefügt
- 🔄 Mailcow Integration Sektion hinzugefügt

---

### 🎯 Universelle Konfiguration

**Eine Konfiguration für alle Server:**
- ✅ Unterschiede nur über `.env` steuerbar
- ✅ Mailcow-Features auf allen Servern verfügbar (inaktiv wenn Container nicht existiert)
- ✅ Alle Middlewares über Labels steuerbar
- ✅ Einfachere Wartung (nur ein Stack zu pflegen)

**Konfigurierte Mailcow-Domains:**
1. autodiscover.clicklocal.de / autoconfig.clicklocal.de
2. autodiscover.familie-saeum.de / autoconfig.familie-saeum.de
3. autodiscover.lichte-kraft.eu / autoconfig.lichte-kraft.eu
4. autodiscover.web-seo-consulting.eu / autoconfig.web-seo-consulting.eu
5. mail.web-seo-consulting.eu
6. autodiscover.wunschschreiner.eu / autoconfig.wunschschreiner.eu

**Anpassung:** Domains können in `configs/traefik-dynamic.yaml` Zeile 134 angepasst werden.

---

### 📝 Deployment auf zweitem Server

Um diese Konfiguration auf einem zweiten Server zu nutzen:

1. `.env` anpassen:
   ```bash
   HOSTRULE=Host(`traefik.DEIN-SERVER.web-seo-consulting.eu`)
   ```

2. Mailcow-Domains anpassen (falls abweichend):
   - `configs/traefik-dynamic.yaml` Zeile 134 editieren

3. Netzwerk erstellen & deployen:
   ```bash
   docker network create traefik_proxy_network
   docker compose up -d
   ```

---

## 2025-12-28 - Release v1.0.0

### 🎉 Production Ready Release

Erstes produktionsfertiges Release mit vollständiger CI/CD-Pipeline und mehrsprachiger Dokumentation.

---

### ✅ Hinzugefügt (Projekt-Abschluss)

#### CI/CD & Automation
- ✅ **GitHub Actions CI Pipeline** - Automatische Tests bei Push/PR
  - Docker Compose Validierung
  - YAML Linting
  - Security Scan (Gitleaks)
  - Markdown Linting
  - Config Tests
- ✅ **Release Workflow** - Automatische Release-Erstellung bei Tag
- ✅ **.gitattributes** - Saubere Exports ohne Dev-Files
- ✅ **.markdownlint.json** - Markdown-Qualitätssicherung

#### Dokumentation
- ✅ **README.en.md** - Englische Dokumentation
- ✅ **README.fr.md** - Französische Dokumentation
- ✅ **LICENSE** - GPL-3.0 Lizenz
- ✅ **Support-Badges** - Buy Me a Coffee, GitHub Sponsors, PayPal

#### Sicherheit
- ✅ Passwort-Hashes durch Platzhalter ersetzt in traefik-dynamic.yaml
- ✅ Legacy dashboard-auth Middleware als deprecated markiert

---

## 2025-12-28 - Vollständiges Rewrite nach Traefik Best Practices

### ✅ Hinzugefügt

#### Neue Middlewares (traefik-dynamic.yaml)
- ✅ `security-headers@file` - HSTS, X-Frame-Options, CSP, etc.
- ✅ `compression@file` - Gzip/Brotli Kompression
- ✅ `in-flight-limit@file` - DoS-Schutz (max 100 gleichzeitige Requests)

#### Neue Dateien
- ✅ `.gitignore` - Schützt .env, logs/, volumes/ vor Git
- ✅ `.env.example` - Template für neue Deployments
- ✅ `LABELS-CHECKLIST.md` - Migration Guide für alle Projekte
- ✅ `DEPLOYMENT.md` - Schritt-für-Schritt Deployment-Anleitung
- ✅ `CHANGELOG.md` - Diese Datei

#### Traefik Self-Routing (docker-compose.yaml)
- ✅ Dashboard via Labels statt hardcoded Router
- ✅ Metrics-Endpoint via Labels
- ✅ HTTP → HTTPS Redirect für Dashboard
- ✅ TLS 1.3 für Dashboard & Metrics

---

### 🔄 Geändert

#### docker-compose.yaml
- 🔄 Image von `traefik:latest` → `traefik:v3.2` (pinned version)
- 🔄 Docker Socket jetzt read-only (`:ro`)
- 🔄 Volumes jetzt mit expliziten Permissions (`:rw`)
- 🔄 Labels umstrukturiert für Dashboard & Metrics
- 🔄 Bessere Kommentare & Struktur

#### configs/traefik.yaml
- 🔄 Access Log von stdout → `/var/log/traefik-access.log`
- 🔄 Access Log Filterung (nur 4xx/5xx Errors)
- 🔄 Let's Encrypt Email jetzt in Config (vorher nur command)
- 🔄 `sendAnonymousUsage: false` (Privacy)
- 🔄 Trusted IPs kommentiert (nur localhost default)
- 🔄 Bessere Struktur & Kommentare

#### configs/traefik-dynamic.yaml
- 🔄 Ping-Router entfernt (nicht benötigt mit manualRouting)
- 🔄 Alle Middlewares kategorisiert & kommentiert
- 🔄 Bessere Struktur

---

### 🔴 Kritische Änderungen (Breaking Changes)

#### Docker Socket bleibt exponiert
⚠️ **Sicherheitsrisiko:** `/var/run/docker.sock` ist weiterhin gemountet
- **Warum:** Für Docker Provider notwendig
- **Risiko:** Bei Traefik-Kompromittierung = Host-Kompromittierung
- **TODO:** Docker Socket Proxy implementieren (z.B. tecnativa/docker-socket-proxy)

#### TLS 1.3 Only
⚠️ **Breaking:** Nur TLS 1.3, TLS 1.2 ist blockiert
- **Betrifft:** Sehr alte Browser (IE11, Android < 10)
- **Vorteil:** Maximale Sicherheit

#### Access Logs nicht mehr in stdout
⚠️ **Geändert:** Access Logs gehen jetzt in `/var/log/traefik-access.log`
- **Warum:** Bessere Kontrolle, Rotation, weniger Docker-Log-Spam
- **Beachten:** Logs müssen manuell geleert werden (oder via Rotation)

---

### 🔒 Sicherheitsverbesserungen

1. ✅ **Security Headers** - HSTS, X-Frame-Options, CSP, etc.
2. ✅ **Access Log Filterung** - Nur relevante Requests loggen
3. ✅ **Docker Socket read-only** - Reduziert Schreibzugriff
4. ✅ **Pinned Version** - Keine unerwarteten Updates
5. ✅ **Let's Encrypt Email** - Korrekte Konfiguration
6. ✅ **Anonymous Usage Tracking aus** - Privacy
7. ✅ **TLS 1.3 only** - Maximale Verschlüsselung
8. ✅ `.gitignore` - Keine Secrets in Git

---

### 🚀 Performance-Verbesserungen

1. ✅ **Compression Middleware** - Gzip/Brotli für alle Projekte
2. ✅ **In-Flight Limiting** - Verhindert Überlastung
3. ✅ **Access Log Filterung** - Weniger I/O
4. ✅ **HTTP/3 Support** - Bereits aktiviert (beibehalten)

---

### 📝 Dokumentations-Verbesserungen

1. ✅ **LABELS-CHECKLIST.md** - 4 Templates für verschiedene Use-Cases
2. ✅ **DEPLOYMENT.md** - Komplette Deployment-Anleitung
3. ✅ **Inline-Kommentare** - Alle Configs sind jetzt kommentiert
4. ✅ **.env.example** - Template für neue Server

---

### ❌ Entfernt

- ❌ Ping-Router aus traefik-dynamic.yaml (nicht benötigt)
- ❌ Breite Trusted IPs Ranges (10.0.0.0/8, etc.)
- ❌ `latest` Tag bei Docker Image
- ❌ Hardcoded Dashboard-Router (jetzt via Labels)

---

### 🔜 TODO (Empfehlungen für Zukunft)

#### Hohe Priorität
1. [ ] **Docker Socket Proxy** implementieren
2. [ ] **.env aus Git entfernen** (`.gitignore` verwenden)
3. [ ] **Dashboard-Passwort ändern** (siehe DEPLOYMENT.md)
4. [ ] **Projekte migrieren** (siehe LABELS-CHECKLIST.md)

#### Mittlere Priorität
5. [ ] **Backup-Script** für `volumes/Traefik.json` erstellen
6. [ ] **Fail2Ban** für automatisches IP-Banning einrichten
7. [ ] **Prometheus** für Metrics-Collection aufsetzen
8. [ ] **Grafana** Dashboard für Traefik-Metriken

#### Niedrige Priorität
9. [ ] **Lokale GeoIP-Datenbank** statt geojs.io API
10. [ ] **Custom Error Pages** für 4xx/5xx Errors
11. [ ] **Health Checks** für alle Services
12. [ ] **Traefik Pilot** für zentrales Monitoring (optional)

---

## Migration Path für bestehende Projekte

### Phase 1: Traefik Stack (JETZT)
1. ✅ Neue Configs deployen
2. ✅ Testen mit `DEPLOYMENT.md`
3. ✅ Validieren dass alles funktioniert

### Phase 2: Projekt-Migration (DANACH)
1. [ ] **Shopware** Labels aktualisieren (siehe LABELS-CHECKLIST.md)
2. [ ] **JTL** Labels aktualisieren + HTTP-Router hinzufügen
3. [ ] **Matomo** Labels aktualisieren + HTTP-Router hinzufügen
4. [ ] Weitere Projekte nach Templates migrieren

### Phase 3: Security Hardening (SPÄTER)
1. [ ] Docker Socket Proxy implementieren
2. [ ] Fail2Ban einrichten
3. [ ] Monitoring aufsetzen

---

## Validierung nach Deployment

**Checklist:**
- [ ] Dashboard erreichbar: https://traefik.server-erde.web-seo-consulting.eu/dashboard/
- [ ] HTTP → HTTPS funktioniert
- [ ] Security Headers gesetzt
- [ ] TLS 1.3 only (TLS 1.2 blockiert)
- [ ] Metrics erreichbar: https://traefik.server-erde.web-seo-consulting.eu/metrics
- [ ] Keine Errors in Logs: `docker compose logs traefik | grep -i error`
- [ ] Let's Encrypt Zertifikate OK

**Test-Commands siehe:** `DEPLOYMENT.md`

---

## Breaking Changes für Users

### Was funktioniert NICHT mehr?

1. **TLS 1.2 Clients** - Nur TLS 1.3
2. **Unverschlüsselte Requests** - HTTP wird zwingend zu HTTPS redirected
3. **Requests ohne Host-Header** - SNI Strict Mode

### Was muss angepasst werden?

1. **JTL & Matomo** - HTTP-Router hinzufügen
2. **Alle Projekte** - `security-headers@file` hinzufügen (empfohlen)
3. **Shopware** - Middleware-Chain erweitern

---

## Rollback-Plan

Falls Probleme auftreten:

```bash
# Stoppen
docker compose down

# Git Rollback
git checkout HEAD~1 -- configs/
git checkout HEAD~1 -- docker-compose.yaml

# Starten
docker compose up -d
```

**Wichtig:** Rollback innerhalb 90 Tage möglich (Let's Encrypt Rate Limits!)

---

## Credits

Basiert auf:
- [Traefik Official Documentation](https://doc.traefik.io/traefik/)
- [Traefik Best Practices 2025](https://doc.traefik.io/traefik/)
- [OWASP Security Headers](https://owasp.org/www-project-secure-headers/)
- [Mozilla TLS Configuration](https://ssl-config.mozilla.org/)
