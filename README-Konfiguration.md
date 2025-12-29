# Konfiguration

[🏠 Zurück zur Hauptdokumentation](README.md) | [🇬🇧 English Version](README-Konfiguration.en.md)

---

## 📝 Übersicht

Diese Anleitung zeigt dir, wie du Traefik für deine spezifischen Anforderungen konfigurierst.

**Konfigurationsdateien:**
- `.env` - Environment-Variablen (Passwörter, Domains, Rate Limits)
- `configs/traefik.yaml` - Statische Konfiguration (Entry Points, Providers)
- `configs/traefik-dynamic.yaml` - Dynamische Konfiguration (Middlewares, Router)
- `docker-compose.yaml` - Service-Definition

---

## 🔐 Basis-Konfiguration (.env)

Alle sensiblen Daten und häufig geänderte Werte sind in der `.env` Datei.

### Vollständige .env Referenz

```bash
# ========================================
# Projekt
# ========================================
COMPOSE_PROJECT_NAME=traefik
TIMEZONE=Europe/Berlin
RESTART=unless-stopped

# ========================================
# Dashboard
# ========================================
HOSTRULE=Host(`traefik.deine-domain.de`)

# ========================================
# Let's Encrypt
# ========================================
LETSENCRYPT_EMAIL=deine-email@example.com

# ========================================
# Netzwerk
# ========================================
PROXY_NETWORK=traefik_proxy_network

# ========================================
# Authentication
# ========================================
DASHBOARD_USER=traefik-admin
DASHBOARD_PASSWORD_HASH=$$apr1$$CHANGE_ME$$CHANGE_ME

# ========================================
# Rate Limiting
# ========================================
RATE_LIMIT_AVERAGE=100
RATE_LIMIT_BURST=50
RATE_LIMIT_PERIOD=1s

# ========================================
# In-Flight Limiting
# ========================================
IN_FLIGHT_LIMIT=100
```

**Detaillierte Erklärungen:** Siehe [ENV-CONFIGURATION.md](ENV-CONFIGURATION.md)

---

## 🛡️ Middlewares

Traefik bietet 8 vorkonfigurierte Middlewares. Nutze sie über Labels in deinen Services.

### Verfügbare Middlewares

| Middleware | Zweck | Verwendung |
|------------|-------|------------|
| `redirect-to-https@file` | HTTP → HTTPS Redirect | Alle Services |
| `redirect-to-www@file` | Subdomain → www | E-Commerce, Websites |
| `geo-block@file` | Länder blockieren | E-Commerce, Admin-Tools |
| `security-headers@file` | HSTS, X-Frame-Options | Alle Services |
| `compression@file` | Gzip/Brotli | Alle Services |
| `rate-limit@file` | DoS-Schutz | Alle Services |
| `in-flight-limit@file` | Concurrent Requests | High-Traffic Services |
| `dashboard-auth@file` | BasicAuth (Legacy) | Nicht mehr verwendet |

---

## 🎯 Service-Labels

So verbindest du deine Services mit Traefik:

### Minimal-Beispiel (HTTPS)

```yaml
services:
  myservice:
    image: nginx:alpine
    labels:
      - traefik.enable=true
      - traefik.http.routers.myservice.rule=Host(`example.com`)
      - traefik.http.routers.myservice.entrypoints=websecure-https
      - traefik.http.routers.myservice.tls.certresolver=letsEncrypt
    networks:
      - traefik_proxy_network

networks:
  traefik_proxy_network:
    external: true
```

### Mit HTTP → HTTPS Redirect

```yaml
labels:
  - traefik.enable=true

  # HTTP Router (Port 80)
  - traefik.http.routers.myservice-http.rule=Host(`example.com`)
  - traefik.http.routers.myservice-http.entrypoints=web-http
  - traefik.http.routers.myservice-http.middlewares=redirect-to-https@file

  # HTTPS Router (Port 443)
  - traefik.http.routers.myservice.rule=Host(`example.com`)
  - traefik.http.routers.myservice.entrypoints=websecure-https
  - traefik.http.routers.myservice.tls.certresolver=letsEncrypt
  - traefik.http.routers.myservice.middlewares=security-headers@file,compression@file
```

### Produktiv-Website (alle Middlewares)

```yaml
labels:
  - traefik.enable=true

  # HTTP → HTTPS
  - traefik.http.routers.shop-http.rule=Host(`shop.example.com`)
  - traefik.http.routers.shop-http.entrypoints=web-http
  - traefik.http.routers.shop-http.middlewares=redirect-to-https@file

  # HTTPS mit Security
  - traefik.http.routers.shop.rule=Host(`shop.example.com`)
  - traefik.http.routers.shop.entrypoints=websecure-https
  - traefik.http.routers.shop.tls.certresolver=letsEncrypt
  - traefik.http.routers.shop.middlewares=redirect-to-www@file,geo-block@file,security-headers@file,compression@file,rate-limit@file
```

**Weitere Beispiele:** Siehe [LABELS-CHECKLIST.md](LABELS-CHECKLIST.md)

---

## 🌍 Geo-Blocking anpassen

Die blockierten Länder sind in `configs/traefik-dynamic.yaml` definiert.

### Standard (23 Länder)

```yaml
geo-block:
  plugin:
    geoblock:
      countries:
        - RU  # Russland
        - CN  # China
        - IR  # Iran
        - IN  # Indien
        # ... 19 weitere
      blackListMode: true
      allowUnknownCountries: false
```

### Anpassen

```bash
# Bearbeite die Dynamic Config
nano configs/traefik-dynamic.yaml

# Suche nach "geo-block" (Zeile 36)
# Füge hinzu oder entferne Ländercodes

# Änderungen werden automatisch geladen (watch: true)
# Kein Neustart nötig!
```

**Ländercodes:** [ISO 3166-1 alpha-2](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2)

---

## 🔒 TLS-Konfiguration

### TLS 1.3 Only (Standard)

```yaml
# configs/traefik-dynamic.yaml
tls:
  options:
    modern:
      minVersion: "VersionTLS13"
      sniStrict: true
      cipherSuites:
        - "TLS_AES_128_GCM_SHA256"
        - "TLS_AES_256_GCM_SHA384"
        - "TLS_CHACHA20_POLY1305_SHA256"
```

**Blockiert:** TLS 1.2, TLS 1.1, TLS 1.0
**Kompatibilität:** IE11, Android < 10, iOS < 12.2

### TLS 1.2 + TLS 1.3 (Kompatibel)

Falls du ältere Browser unterstützen musst:

```yaml
tls:
  options:
    modern:
      minVersion: "VersionTLS12"
      sniStrict: true
      cipherSuites:
        # TLS 1.3
        - "TLS_AES_128_GCM_SHA256"
        - "TLS_AES_256_GCM_SHA384"
        - "TLS_CHACHA20_POLY1305_SHA256"
        # TLS 1.2
        - "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
        - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
```

---

## 📊 Rate Limiting anpassen

### Via .env (empfohlen)

```bash
# Durchschnittliche Requests pro Sekunde
RATE_LIMIT_AVERAGE=100

# Burst-Größe
RATE_LIMIT_BURST=50

# Zeitraum
RATE_LIMIT_PERIOD=1s

# Neustart nach Änderung:
docker compose up -d --force-recreate
```

### Via traefik-dynamic.yaml

```yaml
rate-limit:
  rateLimit:
    average: 200      # Requests/Sekunde
    burst: 100        # Burst-Limit
    period: 1s        # Zeitraum
```

**Empfehlungen:**
- **API:** 50-100 req/s
- **Website:** 100-200 req/s
- **High-Traffic:** 500+ req/s

---

## 📧 Mailcow Integration

### Domains anpassen

```bash
# Bearbeite Dynamic Config
nano configs/traefik-dynamic.yaml

# Zeile 134: mailcow-acme-challenge Router
rule: "(Host(`autodiscover.deine-domain.de`) || Host(`autoconfig.deine-domain.de`) || Host(`mail.deine-domain.de`)) && PathPrefix(`/.well-known/acme-challenge/`)"
```

### Mailcow Container verbinden

```bash
# Füge Mailcow zum Traefik-Netzwerk hinzu
docker network connect traefik_proxy_network mailcow-nginx-mailcow-1

# Prüfe Verbindung
docker network inspect traefik_proxy_network | grep mailcow
```

---

## 🎨 Custom Error Pages

### Error Page Middleware erstellen

```yaml
# In traefik-dynamic.yaml
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
        - url: "http://error-pages-container:80"
```

---

## 🔍 Logging anpassen

### Access Log Level ändern

```yaml
# configs/traefik.yaml
accessLog:
  filePath: "/var/log/traefik-access.log"
  format: "json"
  filters:
    statusCodes:
      - "400-599"  # Nur Fehler
      # Oder: - "200-599"  # Alles loggen
    minDuration: "10ms"
```

### Application Log Level

```yaml
log:
  level: INFO  # DEBUG, INFO, WARN, ERROR
```

---

## 🔄 Änderungen anwenden

### .env Änderungen

```bash
# .env bearbeiten
nano .env

# Container neu erstellen (WICHTIG!)
docker compose up -d --force-recreate

# NICHT verwenden: docker compose restart
```

### traefik.yaml Änderungen

```bash
# Config bearbeiten
nano configs/traefik.yaml

# Container neu starten
docker compose restart traefik
```

### traefik-dynamic.yaml Änderungen

```bash
# Config bearbeiten
nano configs/traefik-dynamic.yaml

# Automatisch neu geladen (watch: true)
# Kein Neustart nötig!
```

---

## 📚 Weitere Dokumentation

- **ENV-Variablen:** [ENV-CONFIGURATION.md](ENV-CONFIGURATION.md)
- **Label-Templates:** [LABELS-CHECKLIST.md](LABELS-CHECKLIST.md)
- **Deployment-Guide:** [DEPLOYMENT.md](DEPLOYMENT.md)
- **Changelog:** [CHANGELOG.md](CHANGELOG.md)

---

**Made with ❤️ by WSC - Web SEO Consulting**

Dieses Projekt ist kostenlos und Open Source (GPL-3.0). Wenn es dir geholfen hat, freue ich mich über deine Unterstützung:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

[🏠 Zurück zur Hauptdokumentation](README.md)
