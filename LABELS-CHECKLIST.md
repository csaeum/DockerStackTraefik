# Traefik Labels - Checkliste & Migration Guide

## Übersicht verfügbarer Middlewares

Nach dem Update stehen folgende Middlewares zur Verfügung:

| Middleware | Zweck | Verwendung |
|------------|-------|------------|
| `redirect-to-https@file` | HTTP → HTTPS Redirect | **PFLICHT** für HTTP-Router |
| `redirect-to-www@file` | Redirect auf www-Subdomain | Optional (nur für Websites) |
| `geo-block@file` | Blockiert 23 Länder | **Empfohlen** für öffentliche Services |
| `security-headers@file` | HSTS, X-Frame-Options, etc. | **Empfohlen** für alle Projekte |
| `compression@file` | Gzip/Brotli Kompression | **Empfohlen** für Performance |
| `rate-limit@file` | 100 req/s DoS-Schutz | **Empfohlen** für APIs & Logins |
| `in-flight-limit@file` | Max 100 gleichzeitige Requests | Optional (bei hoher Last) |
| `dashboard-auth@file` | BasicAuth für Traefik | **Nur für Traefik** |

---

## Label-Templates nach Projekt-Typ

### Template 1: E-Commerce / Produktiv-Website (Shopware, WordPress, etc.)

**Verwendung:** Öffentliche Websites mit maximaler Sicherheit

```yaml
labels:
  - traefik.enable=true

  # HTTP Router (nur für HTTPS-Redirect)
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

  # HTTPS Router
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=redirect-to-www@file,geo-block@file,security-headers@file,compression@file,rate-limit@file
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadbalancer.server.port=8000
```

**Middleware-Chain erklärt:**
1. `redirect-to-www@file` - Erzwingt www-Subdomain
2. `geo-block@file` - Blockiert 23 Risiko-Länder
3. `security-headers@file` - Setzt HSTS, X-Frame-Options, etc.
4. `compression@file` - Komprimiert Responses
5. `rate-limit@file` - 100 req/s Limit

---

### Template 2: Interne Tools mit Login (JTL, Matomo, Adminer)

**Verwendung:** Admin-Panels, Analytics, Datenbank-Tools

```yaml
labels:
  - traefik.enable=true

  # HTTP Router
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

  # HTTPS Router
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=geo-block@file,security-headers@file,compression@file,rate-limit@file
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadbalancer.passHostHeader=true
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadbalancer.server.port=8080
```

**Middleware-Chain erklärt:**
1. `geo-block@file` - Blockiert 23 Risiko-Länder
2. `security-headers@file` - Security Headers
3. `compression@file` - Performance
4. `rate-limit@file` - Brute-Force Schutz

**KEIN** `redirect-to-www@file` - Tools laufen oft auf Subdomains

---

### Template 3: Public API (Ohne Geo-Blocking)

**Verwendung:** REST APIs, Webhooks, öffentliche Services

```yaml
labels:
  - traefik.enable=true

  # HTTP Router
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

  # HTTPS Router
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=security-headers@file,compression@file,rate-limit@file,in-flight-limit@file
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadbalancer.server.port=3000
```

**Middleware-Chain erklärt:**
1. `security-headers@file` - Security
2. `compression@file` - Performance
3. `rate-limit@file` - 100 req/s Limit
4. `in-flight-limit@file` - Max 100 gleichzeitige Connections

**KEIN** Geo-Blocking - APIs müssen global erreichbar sein

---

### Template 4: Staging/Dev Environment

**Verwendung:** Test-Umgebungen, Entwicklung

```yaml
labels:
  - traefik.enable=true

  # HTTP Router
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

  # HTTPS Router
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=security-headers@file,compression@file
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadbalancer.server.port=8000
```

**Minimale Security** - Nur essentials, kein Geo-Blocking, kein Rate-Limiting

---

## Migration-Checkliste

### Deine bestehenden Projekte:

#### ✅ Shopware (aktuell)
```yaml
middlewares=redirect-to-www@file,redirect-to-https@file
```

**Zu migrieren auf:**
```yaml
# HTTP Router hinzufügen:
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

# HTTPS Router ändern:
middlewares=redirect-to-www@file,geo-block@file,security-headers@file,compression@file,rate-limit@file

# TLS-Options hinzufügen:
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
```

---

#### ✅ JTL (aktuell)
```yaml
# Kein HTTP-Router!
# Keine Middlewares!
```

**Zu migrieren auf:**
```yaml
# HTTP Router hinzufügen:
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

# Middlewares hinzufügen:
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=geo-block@file,security-headers@file,compression@file,rate-limit@file

# TLS-Options hinzufügen:
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
```

---

#### ✅ Matomo (aktuell)
```yaml
# Kein HTTP-Router!
# Keine Middlewares!
```

**Zu migrieren auf:**
```yaml
# HTTP Router hinzufügen:
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
- traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

# Middlewares hinzufügen:
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=geo-block@file,security-headers@file,compression@file,rate-limit@file

# TLS-Options hinzufügen:
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
```

---

## Wichtige Hinweise

### ⚠️ BREAKING CHANGES

1. **HTTP-Router ist jetzt Pflicht** - Ohne HTTP-Router gibt's keine HTTPS-Weiterleitung!
2. **TLS-Options explizit setzen** - `tls.options=modern@file` für TLS 1.3
3. **Middleware-Reihenfolge beachten** - Erst Redirects, dann Security, dann Performance

### ✅ Was du NICHT ändern musst

- `.env` Datei (funktioniert weiter)
- Bestehende Domains (${HOSTRULE} bleibt gleich)
- Port-Konfigurationen (bleiben wie sie sind)

### 🔍 Testen nach Migration

1. **HTTP → HTTPS Redirect testen:**
   ```bash
   curl -I http://deine-domain.de
   # Sollte 301 Moved Permanently zurückgeben
   ```

2. **Security Headers prüfen:**
   ```bash
   curl -I https://deine-domain.de
   # Sollte Strict-Transport-Security enthalten
   ```

3. **Compression testen:**
   ```bash
   curl -H "Accept-Encoding: gzip" -I https://deine-domain.de
   # Sollte Content-Encoding: gzip haben
   ```

4. **Geo-Blocking testen:**
   - Über VPN mit russischer IP verbinden
   - Sollte 403 Forbidden zurückgeben

---

## Schnell-Referenz: Middleware-Kombinationen

| Use Case | Middleware-Chain |
|----------|------------------|
| **E-Commerce** | `redirect-to-www,geo-block,security-headers,compression,rate-limit` |
| **Admin-Panel** | `geo-block,security-headers,compression,rate-limit` |
| **Public API** | `security-headers,compression,rate-limit,in-flight-limit` |
| **Staging** | `security-headers,compression` |
| **Nur HTTPS** | `redirect-to-https` (im HTTP-Router) |

---

## Fragen?

- Dashboard-Auth Passwort ändern: Siehe `configs/traefik-dynamic.yaml:13`
- Geo-Blocking anpassen: Siehe `configs/traefik-dynamic.yaml:42-64`
- Rate-Limit erhöhen: Siehe `configs/traefik-dynamic.yaml:76-78`
- Neue Middleware hinzufügen: In `configs/traefik-dynamic.yaml` unter `http.middlewares`
