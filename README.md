# Traefik Reverse Proxy Stack

[![CI](https://github.com/csaeum/DockerStackTraefik/workflows/CI%20-%20Docker%20Stack%20Tests/badge.svg)](https://github.com/csaeum/DockerStackTraefik/actions)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Traefik Version](https://img.shields.io/badge/Traefik-v3.6-blue)](https://doc.traefik.io/traefik/)

🇩🇪 Deutsch | [🇬🇧 English](README.en.md) | [🇫🇷 Français](README.fr.md)

Produktions-fertiger Traefik v3.6 Stack mit Security Best Practices, Let's Encrypt, HTTP/3, Geo-Blocking und umfangreichen Middlewares.

## 📋 Inhaltsverzeichnis

- [Features](#-features)
- [Schnellstart - Neuer Server](#-schnellstart---neuer-server)
- [Verfügbare Middlewares](#-verfügbare-middlewares)
- [Label-Kombinationen](#-label-kombinationen)
- [Komplette Label-Beispiele](#-komplette-label-beispiele)
- [Projekt-Typen](#-projekt-typen)
- [Troubleshooting](#-troubleshooting)
- [Erweiterte Konfiguration](#-erweiterte-konfiguration)

---

## 🚀 Features

### Security
- ✅ **TLS 1.3 Only** - Maximale Verschlüsselung
- ✅ **Security Headers** - HSTS, X-Frame-Options, CSP, etc.
- ✅ **Geo-Blocking** - Blockiert 23 Hochrisiko-Länder
- ✅ **Rate Limiting** - DoS-Schutz (100 req/s)
- ✅ **In-Flight Limiting** - Max 100 gleichzeitige Connections
- ✅ **BasicAuth** - Für Dashboard & Metrics
- ✅ **Let's Encrypt** - Automatische SSL-Zertifikate

### Performance
- ✅ **HTTP/3 Support** - QUIC Protocol
- ✅ **Gzip/Brotli Compression** - ~70% kleinere Responses
- ✅ **Log Rotation** - 7 Tage, 5 Backups, 50MB max

### Monitoring
- ✅ **Dashboard** - Web-UI für Traefik
- ✅ **Prometheus Metrics** - `/metrics` Endpoint
- ✅ **JSON Logs** - Strukturiertes Logging
- ✅ **Access Logs** - Nur 4xx/5xx Errors

---

## 🆕 Schnellstart - Neuer Server

### 1. Voraussetzungen

```bash
# Docker & Docker Compose installiert?
docker --version
docker compose version

# Ports frei?
sudo netstat -tulpn | grep -E ':80|:443'
# Sollte leer sein oder nur Traefik zeigen
```

### 2. Repository klonen

```bash
cd /opt  # oder dein bevorzugter Pfad
git clone https://github.com/csaeum/DockerStackTraefik.git
cd DockerStackTraefik
```

### 3. Environment konfigurieren

```bash
# .env erstellen
cp .env.example .env

# .env bearbeiten
nano .env
```

**Wichtige Variablen:**
```bash
COMPOSE_PROJECT_NAME=traefik
HOSTRULE=Host(`traefik.deine-domain.de`)
LETSENCRYPT_EMAIL=deine-email@example.com
PROXY_NETWORK=traefik_proxy_network
RESTART=unless-stopped
TIMEZONE=Europe/Berlin
```

### 4. Docker-Netzwerk erstellen

```bash
docker network create traefik_proxy_network
```

### 5. Ordner-Struktur vorbereiten

```bash
# Verzeichnisse erstellen
mkdir -p logs volumes

# Richtige Permissions
chmod 700 volumes
chmod 755 logs
```

### 6. Traefik starten

```bash
docker compose up -d
```

### 7. Validierung

```bash
# Container läuft?
docker compose ps

# Logs checken
docker compose logs -f traefik

# Dashboard erreichbar?
curl -I https://traefik.deine-domain.de/dashboard/
# Erwartung: HTTP/2 401 (Auth required)

# HTTP → HTTPS Redirect?
curl -I http://traefik.deine-domain.de
# Erwartung: 301 Moved Permanently
```

### 8. Dashboard-Login

- URL: `https://traefik.deine-domain.de/dashboard/`
- User: `traefik-admin`
- Password: **Siehe Passwort ändern unten!**

---

## 🧩 Verfügbare Middlewares

Alle Middlewares sind in `configs/traefik-dynamic.yaml` definiert und können per `@file` referenziert werden.

### Redirects

| Middleware | Funktion | Verwendung |
|------------|----------|------------|
| `redirect-to-https@file` | HTTP → HTTPS Weiterleitung | **PFLICHT** für HTTP-Router |
| `redirect-to-www@file` | Redirect auf www-Subdomain | Optional für Websites |

**Beispiel:**
```yaml
# Im HTTP-Router
middlewares=redirect-to-https@file

# Im HTTPS-Router (optional)
middlewares=redirect-to-www@file,...
```

---

### Security

| Middleware | Funktion | Konfiguration |
|------------|----------|---------------|
| `security-headers@file` | Security Headers setzen | HSTS (1 Jahr), X-Frame-Options: DENY, X-Content-Type-Options: nosniff, Referrer-Policy, Permissions-Policy |
| `geo-block@file` | Geo-Blocking | Blockiert: RU, CN, IR, IN, PK, BD, ID, SG, PH, VN, TH, MY, KR, KZ, HK, TW, AE, QA, SA, JP, NP, LK |
| `dashboard-auth@file` | BasicAuth | User: traefik-admin (nur für Traefik) |

**Beispiel:**
```yaml
middlewares=security-headers@file,geo-block@file
```

**Security Headers im Detail:**
```yaml
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

---

### Rate Limiting & DoS Protection

| Middleware | Funktion | Limits |
|------------|----------|--------|
| `rate-limit@file` | Request Rate Limiting | Average: 100 req/s, Burst: 50, Period: 1s |
| `in-flight-limit@file` | Concurrent Request Limiting | Max: 100 gleichzeitige Requests pro IP |

**Beispiel:**
```yaml
# Für APIs und Login-Seiten
middlewares=rate-limit@file,in-flight-limit@file
```

**Wann welches verwenden?**
- `rate-limit@file` - **Immer empfohlen** für Login-Formulare, APIs
- `in-flight-limit@file` - Nur bei sehr hoher Last oder DoS-Angriffen

---

### Performance

| Middleware | Funktion | Details |
|------------|----------|---------|
| `compression@file` | Gzip/Brotli Kompression | Encodings: gzip, br, zstd. Min: 1024 bytes. Excludes: text/event-stream |

**Beispiel:**
```yaml
middlewares=compression@file
```

**Performance-Gewinn:**
- HTML/CSS/JS: ~70% kleiner
- JSON APIs: ~60% kleiner
- Bilder: Keine Kompression (bereits komprimiert)

---

### TLS Options

| Option | Funktion | Details |
|--------|----------|---------|
| `modern@file` | TLS 1.3 Only | Cipher Suites: AES-128-GCM, AES-256-GCM, ChaCha20-Poly1305, SNI Strict |

**Beispiel:**
```yaml
- traefik.http.routers.mein-projekt.tls.options=modern@file
```

---

## 🔗 Label-Kombinationen

### Kombination 1: Nur HTTPS-Redirect

**Use Case:** Minimale Konfiguration, nur HTTP → HTTPS

```yaml
# HTTP Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

# HTTPS Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
```

**Ergebnis:** HTTP wird zu HTTPS umgeleitet, Let's Encrypt Zertifikat

---

### Kombination 2: HTTPS + WWW-Redirect

**Use Case:** Erzwinge www-Subdomain für Website

```yaml
# HTTP Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

# HTTPS Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=redirect-to-www@file
```

**Beispiel-Flow:**
1. `http://example.com` → `https://example.com` (redirect-to-https)
2. `https://example.com` → `https://www.example.com` (redirect-to-www)

**Wichtig:** `${HOSTRULE}` muss beide Domains matchen:
```bash
HOSTRULE=Host(`example.com`) || Host(`www.example.com`)
```

---

### Kombination 3: HTTPS + Security Headers

**Use Case:** Basis-Sicherheit für alle Projekte

```yaml
# HTTPS Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=security-headers@file,compression@file
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
```

**Ergebnis:**
- HSTS Header (1 Jahr)
- X-Frame-Options: DENY
- TLS 1.3 only
- Gzip/Brotli Kompression

---

### Kombination 4: HTTPS + Geo-Blocking

**Use Case:** Öffentliche Website mit geografischem Schutz

```yaml
# HTTPS Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=geo-block@file,security-headers@file
```

**Ergebnis:** Besucher aus RU, CN, IR, etc. erhalten 403 Forbidden

---

### Kombination 5: Maximale Sicherheit (E-Commerce)

**Use Case:** Shopware, WooCommerce, produktive Websites

```yaml
# HTTP Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

# HTTPS Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=redirect-to-www@file,geo-block@file,security-headers@file,compression@file,rate-limit@file
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
```

**Middleware-Chain Reihenfolge:**
1. `redirect-to-www@file` - Domain normalisieren
2. `geo-block@file` - Länder blocken
3. `security-headers@file` - Security Headers
4. `compression@file` - Performance
5. `rate-limit@file` - DoS-Schutz

**Ergebnis:** Maximaler Schutz + Performance

---

### Kombination 6: API mit Rate Limiting

**Use Case:** REST API, GraphQL API

```yaml
# HTTPS Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=security-headers@file,compression@file,rate-limit@file,in-flight-limit@file
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
```

**Kein Geo-Blocking** - APIs müssen global erreichbar sein
**Doppelter DoS-Schutz** - Rate Limiting + In-Flight Limiting

---

### Kombination 7: Internes Tool (Admin-Panel)

**Use Case:** JTL, Matomo, Adminer, phpMyAdmin

```yaml
# HTTPS Router
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=geo-block@file,security-headers@file,compression@file,rate-limit@file
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
```

**Kein WWW-Redirect** - Tools laufen auf Subdomains
**Mit Geo-Blocking** - Zusätzlicher Schutz für Login-Seiten

---

## 📦 Komplette Label-Beispiele

### Beispiel 1: Shopware 6

```yaml
services:
  shopware:
    image: shopware/production:latest
    container_name: shopware
    environment:
      APP_URL: https://www.shop.example.com
    networks:
      - traefik_proxy_network
    labels:
      # Traefik aktivieren
      - traefik.enable=true

      # HTTP Router (Port 80 -> HTTPS Redirect)
      - traefik.http.routers.shopware-http.rule=Host(`shop.example.com`) || Host(`www.shop.example.com`)
      - traefik.http.routers.shopware-http.entrypoints=web-http
      - traefik.http.routers.shopware-http.middlewares=redirect-to-https@file

      # HTTPS Router (Port 443)
      - traefik.http.routers.shopware.rule=Host(`shop.example.com`) || Host(`www.shop.example.com`)
      - traefik.http.routers.shopware.entrypoints=websecure-https
      - traefik.http.routers.shopware.tls.certresolver=letsEncrypt
      - traefik.http.routers.shopware.tls.options=modern@file
      - traefik.http.routers.shopware.middlewares=redirect-to-www@file,geo-block@file,security-headers@file,compression@file,rate-limit@file

      # Service (Backend Port)
      - traefik.http.services.shopware.loadbalancer.server.port=8000

networks:
  traefik_proxy_network:
    external: true
```

**Was passiert:**
1. HTTP Request → HTTPS Redirect
2. HTTPS Request ohne www → www Redirect
3. Geo-Blocking prüft IP
4. Security Headers werden gesetzt
5. Response wird komprimiert
6. Rate Limiting aktiv
7. Let's Encrypt Zertifikat
8. TLS 1.3 only

---

### Beispiel 2: JTL-Wawi (Admin-Tool)

```yaml
services:
  jtl:
    image: jtl/wawi:latest
    container_name: jtl-wawi
    networks:
      - traefik_proxy_network
    labels:
      - traefik.enable=true

      # HTTP Router
      - traefik.http.routers.jtl-http.rule=Host(`jtl.example.com`)
      - traefik.http.routers.jtl-http.entrypoints=web-http
      - traefik.http.routers.jtl-http.middlewares=redirect-to-https@file

      # HTTPS Router
      - traefik.http.routers.jtl.rule=Host(`jtl.example.com`)
      - traefik.http.routers.jtl.entrypoints=websecure-https
      - traefik.http.routers.jtl.tls.certresolver=letsEncrypt
      - traefik.http.routers.jtl.tls.options=modern@file
      - traefik.http.routers.jtl.middlewares=geo-block@file,security-headers@file,compression@file,rate-limit@file

      # Service
      - traefik.http.services.jtl.loadbalancer.passHostHeader=true
      - traefik.http.services.jtl.loadbalancer.server.port=8080

networks:
  traefik_proxy_network:
    external: true
```

**Unterschiede zu Shopware:**
- Kein WWW-Redirect (läuft auf Subdomain)
- `passHostHeader=true` (wichtig für Reverse Proxy)

---

### Beispiel 3: Public REST API

```yaml
services:
  api:
    image: my-api:latest
    container_name: my-api
    networks:
      - traefik_proxy_network
    labels:
      - traefik.enable=true

      # HTTP Router
      - traefik.http.routers.api-http.rule=Host(`api.example.com`)
      - traefik.http.routers.api-http.entrypoints=web-http
      - traefik.http.routers.api-http.middlewares=redirect-to-https@file

      # HTTPS Router
      - traefik.http.routers.api.rule=Host(`api.example.com`)
      - traefik.http.routers.api.entrypoints=websecure-https
      - traefik.http.routers.api.tls.certresolver=letsEncrypt
      - traefik.http.routers.api.tls.options=modern@file
      - traefik.http.routers.api.middlewares=security-headers@file,compression@file,rate-limit@file,in-flight-limit@file

      # Service
      - traefik.http.services.api.loadbalancer.server.port=3000

networks:
  traefik_proxy_network:
    external: true
```

**Unterschiede:**
- **KEIN** Geo-Blocking (global erreichbar)
- **Doppelter DoS-Schutz** (rate-limit + in-flight-limit)

---

### Beispiel 4: WordPress (Staging)

```yaml
services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress-staging
    networks:
      - traefik_proxy_network
    labels:
      - traefik.enable=true

      # HTTP Router
      - traefik.http.routers.staging-http.rule=Host(`staging.example.com`)
      - traefik.http.routers.staging-http.entrypoints=web-http
      - traefik.http.routers.staging-http.middlewares=redirect-to-https@file

      # HTTPS Router (Minimale Security für Staging)
      - traefik.http.routers.staging.rule=Host(`staging.example.com`)
      - traefik.http.routers.staging.entrypoints=websecure-https
      - traefik.http.routers.staging.tls.certresolver=letsEncrypt
      - traefik.http.routers.staging.tls.options=modern@file
      - traefik.http.routers.staging.middlewares=security-headers@file,compression@file

      # Service
      - traefik.http.services.staging.loadbalancer.server.port=80

networks:
  traefik_proxy_network:
    external: true
```

**Staging-Besonderheiten:**
- Minimale Middlewares (nur essentials)
- Kein Geo-Blocking
- Kein Rate-Limiting (für Tests)

---

### Beispiel 5: Mailcow verbinden

```yaml
# Kein docker-compose.yaml - nur Command!

# Mailcow zum Traefik-Netzwerk hinzufügen
docker network connect traefik_proxy_network mailcow-nginx-mailcow-1
```

**Dann in Mailcow docker-compose.yaml Labels hinzufügen:**
```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.mailcow.rule=Host(`mail.example.com`)
  - traefik.http.routers.mailcow.entrypoints=websecure-https
  - traefik.http.routers.mailcow.tls.certresolver=letsEncrypt
  - traefik.http.services.mailcow.loadbalancer.server.port=443
  - traefik.http.services.mailcow.loadbalancer.server.scheme=https
```

---

## 🎯 Projekt-Typen

### Typ 1: E-Commerce / Produktiv-Website

**Charakteristik:** Öffentlich, hoher Traffic, Security wichtig

**Middlewares:**
```yaml
redirect-to-www@file,geo-block@file,security-headers@file,compression@file,rate-limit@file
```

**Projekte:** Shopware, WooCommerce, Magento, PrestaShop

---

### Typ 2: Admin-Tools / Backend

**Charakteristik:** Interne Tools, Login erforderlich, sensible Daten

**Middlewares:**
```yaml
geo-block@file,security-headers@file,compression@file,rate-limit@file
```

**Projekte:** JTL, Matomo, Adminer, phpMyAdmin, Portainer

---

### Typ 3: Public API

**Charakteristik:** Global erreichbar, hoher Traffic, DoS-Gefahr

**Middlewares:**
```yaml
security-headers@file,compression@file,rate-limit@file,in-flight-limit@file
```

**Projekte:** REST API, GraphQL API, Webhooks

---

### Typ 4: Staging / Dev

**Charakteristik:** Test-Umgebung, keine Production-Daten

**Middlewares:**
```yaml
security-headers@file,compression@file
```

**Projekte:** Alle Staging/Dev-Environments

---

## 🔧 Troubleshooting

### Dashboard nicht erreichbar

**Symptom:** `curl https://traefik.deine-domain.de/dashboard/` → Timeout

**Checks:**
```bash
# Container läuft?
docker compose ps

# Labels gesetzt?
docker inspect traefik | grep -A 30 Labels

# Logs prüfen
docker compose logs traefik | grep -i error
```

**Lösung:** Siehe `DEPLOYMENT.md`

---

### HTTP → HTTPS Redirect funktioniert nicht

**Symptom:** HTTP bleibt auf HTTP

**Ursache:** HTTP-Router fehlt in deinem Projekt

**Lösung:**
```yaml
# Diesen Router hinzufügen:
- traefik.http.routers.PROJEKT-http.rule=${HOSTRULE}
- traefik.http.routers.PROJEKT-http.entrypoints=web-http
- traefik.http.routers.PROJEKT-http.middlewares=redirect-to-https@file
```

---

### Let's Encrypt Fehler

**Symptom:** Kein Zertifikat nach 5 Minuten

**Checks:**
```bash
# ACME Logs
docker compose logs traefik | grep -i acme

# Port 80/443 erreichbar?
curl -I http://deine-domain.de
```

**Häufige Ursachen:**
1. Domain zeigt nicht auf Server
2. Port 80/443 durch Firewall blockiert
3. Rate-Limit erreicht (5 Certs/Woche pro Domain)

**Temporäre Lösung (Staging):**
```yaml
# In traefik.yaml:
caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
```

---

### Geo-Blocking zu strikt

**Symptom:** Legitime User werden geblockt

**Lösung:** Länder-Liste anpassen in `configs/traefik-dynamic.yaml`:

```yaml
geo-block:
  plugin:
    geoblock:
      countries:
        - RU  # Entfernen oder auskommentieren
        # - CN  # Auskommentiert = erlaubt
```

**Neu starten:**
```bash
docker compose restart traefik
```

---

### Rate-Limiting zu aggressiv

**Symptom:** 429 Too Many Requests bei normalem Traffic

**Lösung:** Limits erhöhen in `configs/traefik-dynamic.yaml`:

```yaml
rate-limit:
  rateLimit:
    average: 200  # War: 100
    burst: 100    # War: 50
    period: 1s
```

---

## 🔐 Erweiterte Konfiguration

### Dashboard-Passwort ändern

**Jetzt über .env steuerbar!** Siehe [`ENV-CONFIGURATION.md`](ENV-CONFIGURATION.md) für Details.

**Schnell-Anleitung:**

```bash
# 1. Hash generieren
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "DEIN_NEUES_PASSWORT"

# Output: traefik-admin:$apr1$xyz$abc

# 2. In .env eintragen ($ als $$ escapen!)
# DASHBOARD_PASSWORD_HASH=$$apr1$$xyz$$abc
nano .env

# 3. Container neu erstellen
docker compose up -d --force-recreate
```

**Wichtig:** `$` muss als `$$` escaped werden!

---

### Neue Middleware erstellen

**Beispiel: IP-Whitelist für Admin-Bereich**

In `configs/traefik-dynamic.yaml` hinzufügen:

```yaml
http:
  middlewares:
    admin-whitelist:
      ipAllowList:
        sourceRange:
          - "1.2.3.4/32"      # Deine IP
          - "192.168.1.0/24"  # Dein Netzwerk
```

**Verwenden:**
```yaml
labels:
  - traefik.http.routers.admin.middlewares=admin-whitelist@file,security-headers@file
```

---

### Custom Error Pages

```yaml
# In traefik-dynamic.yaml
http:
  middlewares:
    custom-errors:
      errors:
        status:
          - "400-599"
        service: error-pages
        query: "/{status}.html"

  services:
    error-pages:
      loadBalancer:
        servers:
          - url: "http://error-pages-container:8080"
```

---

### CORS Headers

```yaml
# In traefik-dynamic.yaml
http:
  middlewares:
    cors-headers:
      headers:
        accessControlAllowMethods:
          - GET
          - POST
          - PUT
        accessControlAllowOriginList:
          - https://example.com
        accessControlMaxAge: 100
        addVaryHeader: true
```

**Verwenden für API:**
```yaml
- traefik.http.routers.api.middlewares=cors-headers@file,security-headers@file
```

---

## 📚 Dokumentation

| Datei | Beschreibung |
|-------|--------------|
| `README.md` | Diese Datei - Komplette Übersicht |
| `ENV-CONFIGURATION.md` | ⭐ .env Steuerung - Passwort, Rate-Limits, etc. |
| `LABELS-CHECKLIST.md` | Label-Templates & Migration |
| `DEPLOYMENT.md` | Deployment-Anleitung & Troubleshooting |
| `CHANGELOG.md` | Alle Änderungen im Detail |
| `.env.example` | Template für Umgebungsvariablen |

---

## 🔒 Security Checklist

Vor Production-Deployment:

- [ ] Dashboard-Passwort geändert
- [ ] `.env` NICHT in Git
- [ ] `volumes/Traefik.json` hat Permissions 600
- [ ] Security Headers in allen Projekten
- [ ] TLS 1.3 aktiv (prüfen mit SSL Labs)
- [ ] Geo-Blocking nach Bedarf anpassen
- [ ] Rate-Limiting getestet
- [ ] Backups für `volumes/Traefik.json` eingerichtet
- [ ] Logs rotieren (automatisch aktiviert)

---

## 📊 Monitoring

### Dashboard
- URL: `https://traefik.deine-domain.de/dashboard/`
- Zeigt: Router, Services, Middlewares, TLS-Certs

### Prometheus Metrics
- URL: `https://traefik.deine-domain.de/metrics`
- Format: Prometheus Text-Format
- Metriken: Requests, Duration, Status Codes

### Logs
- Application: `/logs/traefik.log`
- Access: `/logs/traefik-access.log` (nur 4xx/5xx)
- Rotation: 7 Tage, 5 Backups, 50MB max

---

## 🚨 Wichtige Hinweise

### TLS 1.3 Only
Sehr alte Browser funktionieren **NICHT**:
- Internet Explorer 11
- Android < 10
- iOS < 12.2

**Kompromiss:** TLS 1.2 erlauben (in `traefik-dynamic.yaml` ändern)

### Docker Socket Security
⚠️ `/var/run/docker.sock` ist exponiert = Sicherheitsrisiko

**Empfehlung:** Docker Socket Proxy (TODO)

### Let's Encrypt Rate Limits
- **5 Zertifikate** pro Woche pro Domain
- **50 Zertifikate** pro Woche pro Account
- Bei Tests: Staging CA verwenden!

---

## 📞 Support

- **Issues:** GitHub Issues
- **Logs prüfen:** `docker compose logs traefik`
- **Debug Mode:** In `traefik.yaml` setzen: `level: DEBUG`

---

## 📝 Lizenz

Dieses Projekt ist unter der [GNU General Public License v3.0](LICENSE) lizenziert.

---

## 💖 Unterstützung

**Made with ❤️ by WSC - Web SEO Consulting**

Dieses Projekt ist kostenlos und Open Source. Wenn es dir geholfen hat, freue ich mich über deine Unterstützung:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

## 🙏 Credits

Basierend auf:
- [Traefik Official Docs](https://doc.traefik.io/)
- [OWASP Security Headers](https://owasp.org/www-project-secure-headers/)
- [Mozilla SSL Config](https://ssl-config.mozilla.org/)

---

## 🤝 Beitragen

Contributions sind willkommen! Bitte erstelle einen Pull Request oder öffne ein Issue.

---

**Version:** 2025-12-28 | **Traefik:** v3.6 | **TLS:** 1.3 Only

---

© 2025 WSC - Web SEO Consulting. Alle Rechte vorbehalten.
