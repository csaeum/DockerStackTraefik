# Traefik Reverse Proxy Stack

[![CI](https://github.com/csaeum/DockerStackTraefik/workflows/CI%20-%20Docker%20Stack%20Tests/badge.svg)](https://github.com/csaeum/DockerStackTraefik/actions)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Traefik Version](https://img.shields.io/badge/Traefik-v3.6-blue)](https://doc.traefik.io/traefik/)

[🇩🇪 Deutsch](README.md) | 🇬🇧 English | [🇫🇷 Français](README.fr.md)

Production-ready Traefik v3.6 stack with security best practices, Let's Encrypt, HTTP/3, geo-blocking, and comprehensive middlewares.

## 🚀 Features

### Security
- ✅ **TLS 1.3 Only** - Maximum encryption
- ✅ **Security Headers** - HSTS, X-Frame-Options, CSP, etc.
- ✅ **Geo-Blocking** - Blocks 23 high-risk countries
- ✅ **Rate Limiting** - DoS protection (100 req/s)
- ✅ **BasicAuth** - For dashboard & metrics
- ✅ **Let's Encrypt** - Automatic SSL certificates

### Performance
- ✅ **HTTP/3 Support** - QUIC Protocol
- ✅ **Gzip/Brotli Compression** - ~70% smaller responses
- ✅ **Log Rotation** - 7 days, 5 backups, 50MB max

### Monitoring
- ✅ **Dashboard** - Web UI for Traefik
- ✅ **Prometheus Metrics** - `/metrics` endpoint
- ✅ **JSON Logs** - Structured logging

---

## 🆕 Quick Start - New Server

### 1. Prerequisites

```bash
# Docker & Docker Compose installed?
docker --version
docker compose version

# Ports available?
sudo netstat -tulpn | grep -E ':80|:443'
```

### 2. Clone Repository

```bash
cd /opt
git clone https://github.com/csaeum/DockerStackTraefik.git
cd DockerStackTraefik
```

### 3. Configure Environment

```bash
# Create .env
cp .env.example .env

# Edit .env
nano .env
```

**Important variables:**
```bash
COMPOSE_PROJECT_NAME=traefik
HOSTRULE=Host(`traefik.your-domain.com`)
LETSENCRYPT_EMAIL=your-email@example.com
PROXY_NETWORK=traefik_proxy_network
TIMEZONE=Europe/Berlin
```

### 4. Create Docker Network

```bash
docker network create traefik_proxy_network
```

### 5. Prepare Folder Structure

```bash
mkdir -p logs volumes
chmod 700 volumes
chmod 755 logs
```

### 6. Start Traefik

```bash
docker compose up -d
```

### 7. Validation

```bash
# Container running?
docker compose ps

# Check logs
docker compose logs -f traefik

# Dashboard accessible?
curl -I https://traefik.your-domain.com/dashboard/
# Expected: HTTP/2 401 (Auth required)
```

### 8. Dashboard Login

- URL: `https://traefik.your-domain.com/dashboard/`
- User: `traefik-admin`
- Password: **See password change below!**

---

## 🧩 Available Middlewares

All middlewares are defined in `configs/traefik-dynamic.yaml` and can be referenced via `@file`.

| Middleware | Function | Usage |
|------------|----------|-------|
| `redirect-to-https@file` | HTTP → HTTPS redirect | **REQUIRED** for HTTP router |
| `redirect-to-www@file` | Redirect to www subdomain | Optional for websites |
| `geo-block@file` | Blocks 23 countries | **Recommended** for public services |
| `security-headers@file` | HSTS, X-Frame-Options, etc. | **Recommended** for all projects |
| `compression@file` | Gzip/Brotli compression | **Recommended** for performance |
| `rate-limit@file` | 100 req/s DoS protection | **Recommended** for APIs & logins |
| `in-flight-limit@file` | Max 100 concurrent requests | Optional for high load |

---

## 📦 Complete Label Example - Shopware 6

```yaml
services:
  shopware:
    image: shopware/production:latest
    container_name: shopware
    networks:
      - traefik_proxy_network
    labels:
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

---

## 📚 Documentation

| File | Description |
|------|-------------|
| `README.md` | German - Complete overview |
| `README.en.md` | This file - English overview |
| `ENV-CONFIGURATION.md` | ⭐ .env control - password, rate limits, etc. |
| `LABELS-CHECKLIST.md` | Label templates & migration |
| `DEPLOYMENT.md` | Deployment guide & troubleshooting |
| `CHANGELOG.md` | All changes in detail |
| `.env.example` | Environment variable template |

---

## 🔐 Change Dashboard Password

**Now controllable via .env!** See [`ENV-CONFIGURATION.md`](ENV-CONFIGURATION.md) for details.

**Quick Guide:**

```bash
# 1. Generate hash
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "YOUR_NEW_PASSWORD"

# Output: traefik-admin:$apr1$xyz$abc

# 2. Enter in .env (escape $ as $$!)
# DASHBOARD_PASSWORD_HASH=$$apr1$$xyz$$abc
nano .env

# 3. Recreate container
docker compose up -d --force-recreate
```

**Important:** `$` must be escaped as `$$`!

---

## 📝 License

This project is licensed under the [GNU General Public License v3.0](LICENSE).

---

## 💖 Support

**Made with ❤️ by WSC - Web SEO Consulting**

This project is free and open source. If it helped you, I appreciate your support:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

## 🙏 Credits

Based on:
- [Traefik Official Docs](https://doc.traefik.io/)
- [OWASP Security Headers](https://owasp.org/www-project-secure-headers/)
- [Mozilla SSL Config](https://ssl-config.mozilla.org/)

---

## 🤝 Contributing

Contributions are welcome! Please create a pull request or open an issue.

---

**Version:** 2025-12-28 | **Traefik:** v3.6 | **TLS:** 1.3 Only

---

© 2025 WSC - Web SEO Consulting. All rights reserved.
