# Configuration

[🏠 Back to Main Documentation](README.en.md) | [🇩🇪 Deutsche Version](README-Konfiguration.md)

---

## 📝 Overview

This guide shows you how to configure Traefik for your specific requirements.

**Configuration files:**
- `.env` - Environment variables (passwords, domains, rate limits)
- `configs/traefik.yaml` - Static configuration (entry points, providers)
- `configs/traefik-dynamic.yaml` - Dynamic configuration (middlewares, routers)
- `docker-compose.yaml` - Service definition

---

## 🔐 Basic Configuration (.env)

All sensitive data and frequently changed values are in the `.env` file.

### Complete .env Reference

```bash
# ========================================
# Project
# ========================================
COMPOSE_PROJECT_NAME=traefik
TIMEZONE=Europe/Berlin
RESTART=unless-stopped

# ========================================
# Dashboard
# ========================================
HOSTRULE=Host(`traefik.your-domain.com`)

# ========================================
# Let's Encrypt
# ========================================
LETSENCRYPT_EMAIL=your-email@example.com

# ========================================
# Network
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

**Detailed explanations:** See [ENV-CONFIGURATION.md](ENV-CONFIGURATION.md)

---

## 🛡️ Middlewares

Traefik offers 8 pre-configured middlewares. Use them via labels in your services.

### Available Middlewares

| Middleware | Purpose | Usage |
|------------|---------|-------|
| `redirect-to-https@file` | HTTP → HTTPS redirect | All services |
| `redirect-to-www@file` | Subdomain → www | E-commerce, websites |
| `geo-block@file` | Block countries | E-commerce, admin tools |
| `security-headers@file` | HSTS, X-Frame-Options | All services |
| `compression@file` | Gzip/Brotli | All services |
| `rate-limit@file` | DoS protection | All services |
| `in-flight-limit@file` | Concurrent requests | High-traffic services |
| `dashboard-auth@file` | BasicAuth (Legacy) | No longer used |

---

## 🎯 Service Labels

How to connect your services with Traefik:

### Minimal Example (HTTPS)

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

### With HTTP → HTTPS Redirect

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

### Production Website (all middlewares)

```yaml
labels:
  - traefik.enable=true

  # HTTP → HTTPS
  - traefik.http.routers.shop-http.rule=Host(`shop.example.com`)
  - traefik.http.routers.shop-http.entrypoints=web-http
  - traefik.http.routers.shop-http.middlewares=redirect-to-https@file

  # HTTPS with security
  - traefik.http.routers.shop.rule=Host(`shop.example.com`)
  - traefik.http.routers.shop.entrypoints=websecure-https
  - traefik.http.routers.shop.tls.certresolver=letsEncrypt
  - traefik.http.routers.shop.middlewares=redirect-to-www@file,geo-block@file,security-headers@file,compression@file,rate-limit@file
```

**More examples:** See [LABELS-CHECKLIST.md](LABELS-CHECKLIST.md)

---

## 🌍 Adjust Geo-Blocking

Blocked countries are defined in `configs/traefik-dynamic.yaml`.

### Default (23 countries)

```yaml
geo-block:
  plugin:
    geoblock:
      countries:
        - RU  # Russia
        - CN  # China
        - IR  # Iran
        - IN  # India
        # ... 19 more
      blackListMode: true
      allowUnknownCountries: false
```

### Customize

```bash
# Edit dynamic config
nano configs/traefik-dynamic.yaml

# Search for "geo-block" (line 36)
# Add or remove country codes

# Changes are automatically loaded (watch: true)
# No restart needed!
```

**Country codes:** [ISO 3166-1 alpha-2](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2)

---

## 🔒 TLS Configuration

### TLS 1.3 Only (Default)

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

**Blocks:** TLS 1.2, TLS 1.1, TLS 1.0
**Compatibility:** IE11, Android < 10, iOS < 12.2

### TLS 1.2 + TLS 1.3 (Compatible)

If you need to support older browsers:

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

## 📊 Adjust Rate Limiting

### Via .env (recommended)

```bash
# Average requests per second
RATE_LIMIT_AVERAGE=100

# Burst size
RATE_LIMIT_BURST=50

# Period
RATE_LIMIT_PERIOD=1s

# Restart after changes:
docker compose up -d --force-recreate
```

### Via traefik-dynamic.yaml

```yaml
rate-limit:
  rateLimit:
    average: 200      # Requests/second
    burst: 100        # Burst limit
    period: 1s        # Period
```

**Recommendations:**
- **API:** 50-100 req/s
- **Website:** 100-200 req/s
- **High-traffic:** 500+ req/s

---

## 📧 Mailcow Integration

### Adjust Domains

```bash
# Edit dynamic config
nano configs/traefik-dynamic.yaml

# Line 134: mailcow-acme-challenge router
rule: "(Host(`autodiscover.your-domain.com`) || Host(`autoconfig.your-domain.com`) || Host(`mail.your-domain.com`)) && PathPrefix(`/.well-known/acme-challenge/`)"
```

### Connect Mailcow Container

```bash
# Add Mailcow to Traefik network
docker network connect traefik_proxy_network mailcow-nginx-mailcow-1

# Check connection
docker network inspect traefik_proxy_network | grep mailcow
```

---

## 🎨 Custom Error Pages

### Create Error Page Middleware

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

## 🔍 Adjust Logging

### Change Access Log Level

```yaml
# configs/traefik.yaml
accessLog:
  filePath: "/var/log/traefik-access.log"
  format: "json"
  filters:
    statusCodes:
      - "400-599"  # Only errors
      # Or: - "200-599"  # Log everything
    minDuration: "10ms"
```

### Application Log Level

```yaml
log:
  level: INFO  # DEBUG, INFO, WARN, ERROR
```

---

## 🔄 Apply Changes

### .env Changes

```bash
# Edit .env
nano .env

# Recreate container (IMPORTANT!)
docker compose up -d --force-recreate

# DO NOT use: docker compose restart
```

### traefik.yaml Changes

```bash
# Edit config
nano configs/traefik.yaml

# Restart container
docker compose restart traefik
```

### traefik-dynamic.yaml Changes

```bash
# Edit config
nano configs/traefik-dynamic.yaml

# Automatically reloaded (watch: true)
# No restart needed!
```

---

## 📚 Further Documentation

- **ENV Variables:** [ENV-CONFIGURATION.md](ENV-CONFIGURATION.md)
- **Label Templates:** [LABELS-CHECKLIST.md](LABELS-CHECKLIST.md)
- **Deployment Guide:** [DEPLOYMENT.md](DEPLOYMENT.md)
- **Changelog:** [CHANGELOG.md](CHANGELOG.md)

---

**Made with ❤️ by WSC - Web SEO Consulting**

This project is free and Open Source (GPL-3.0). If it helped you, I appreciate your support:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

[🏠 Back to Main Documentation](README.en.md)
