# Installation

[🏠 Back to Main Documentation](README.en.md) | [🇩🇪 Deutsche Version](README-Installation.md)

---

## 📦 Quick Start

### 1. Clone Repository

```bash
# Change to desired directory
cd /opt  # or your preferred path

# Clone the repository
git clone https://github.com/csaeum/DockerStackTraefik.git
cd DockerStackTraefik
```

---

### 2. Create Environment File

```bash
# Copy .env.example to .env
cp .env.example .env

# Edit the .env file
nano .env
```

---

### 3. Configure Environment Variables

Adjust at least these variables in `.env`:

```bash
# Project name (for container names)
COMPOSE_PROJECT_NAME=traefik

# Timezone
TIMEZONE=Europe/Berlin

# Domain for Traefik Dashboard
HOSTRULE=Host(`traefik.your-domain.com`)

# Email for Let's Encrypt
LETSENCRYPT_EMAIL=your-email@example.com

# Network name
PROXY_NETWORK=traefik_proxy_network

# Restart policy
RESTART=unless-stopped
```

---

### 4. Generate Dashboard Password

**Important:** Change the default password!

```bash
# Generate a password hash
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "YOUR_SECURE_PASSWORD"

# Output: traefik-admin:$apr1$xyz123...
```

**Enter the hash in `.env`:**

```bash
# IMPORTANT: $ must be escaped as $$!
DASHBOARD_USER=traefik-admin
DASHBOARD_PASSWORD_HASH=$$apr1$$xyz123$$...
```

**Example:**
```bash
# Generated:
traefik-admin:$apr1$u5m91va6$jYOH.sK1gKMaLmWlNxA7m/

# Enter in .env:
DASHBOARD_PASSWORD_HASH=$$apr1$$u5m91va6$$jYOH.sK1gKMaLmWlNxA7m/
```

---

### 5. Create Docker Network

```bash
# Create the external network for all services
docker network create traefik_proxy_network
```

**Note:** This network is shared by all services that should run behind Traefik.

---

### 6. Prepare Directory Structure

```bash
# Create necessary directories
mkdir -p logs volumes

# Set permissions
chmod 700 volumes  # Root access only
chmod 755 logs     # Readable for logs
```

---

### 7. Validate Configuration

```bash
# Check docker-compose.yaml syntax
docker compose config

# Should show no errors and output the final config
```

---

### 8. Start Traefik

```bash
# Start Traefik
docker compose up -d

# Show logs
docker compose logs -f traefik
```

**Wait for:**
```
✅ Server listening on :80
✅ Server listening on :443 (TCP/HTTP/2)
✅ Server listening on :443 (UDP/HTTP/3)
✅ Certificate obtained for domain [traefik.your-domain.com]
```

---

## ✅ Validate Installation

### 1. Check Container Status

```bash
# Check if container is running
docker compose ps

# Should show: traefik (running)
```

### 2. Test Dashboard

```bash
# Test HTTP → HTTPS redirect
curl -I http://traefik.your-domain.com
# Expect: 301 Moved Permanently
# Location: https://traefik.your-domain.com/

# Access dashboard (with BasicAuth)
curl -I https://traefik.your-domain.com/dashboard/ \
  -u traefik-admin:YOUR_PASSWORD
# Expect: 200 OK
```

### 3. Check TLS Certificate

```bash
# Check Let's Encrypt certificate
openssl s_client -connect traefik.your-domain.com:443 -servername traefik.your-domain.com < /dev/null 2>/dev/null | openssl x509 -noout -text | grep -E 'Issuer|Subject|Not After'

# Should show:
# Issuer: CN = R3 (Let's Encrypt)
# Subject: CN = traefik.your-domain.com
# Not After: (expiration date in 90 days)
```

### 4. Test HTTP/3

```bash
# Check HTTP/3 support (requires curl with HTTP/3)
curl -I --http3 https://traefik.your-domain.com/dashboard/

# Or in browser:
# Chrome DevTools → Network → Protocol should show "h3"
```

### 5. Check Security Headers

```bash
# Check Security Headers
curl -I https://traefik.your-domain.com/dashboard/ \
  -u traefik-admin:YOUR_PASSWORD | grep -E 'Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options'

# Should show:
# Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
# X-Frame-Options: DENY
# X-Content-Type-Options: nosniff
```

---

## 🔧 Troubleshooting

### Problem: Container won't start

```bash
# Check logs
docker compose logs traefik

# Common errors:
# - Port 80/443 already in use → sudo netstat -tulpn | grep -E ':80|:443'
# - .env error → docker compose config
# - Network missing → docker network create traefik_proxy_network
```

### Problem: Let's Encrypt Error

```bash
# Check DNS resolution
dig +short traefik.your-domain.com

# Check if domain is reachable
curl -I http://traefik.your-domain.com

# Check logs for ACME errors
docker compose logs traefik | grep -i acme

# Common errors:
# - DNS not pointing to server → wait for DNS propagation
# - Firewall blocking port 80/443 → check firewall
# - Rate limit reached → wait 1 week or use staging
```

### Problem: Dashboard not reachable

```bash
# Check container status
docker compose ps

# Check ports
docker compose port traefik 443

# Check BasicAuth
# IMPORTANT: $ must be escaped as $$ in .env!
cat .env | grep DASHBOARD_PASSWORD_HASH
```

### Problem: Password not working

```bash
# Check if $$ are escaped
grep DASHBOARD_PASSWORD_HASH .env
# Should show $$, not $

# Regenerate if necessary
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "NEW_PASSWORD"

# Enter in .env ($ → $$)
nano .env

# Restart container
docker compose up -d --force-recreate
```

---

## 🚀 Next Steps

Installation successful? Now you can:

1. **Add services** → Put other containers behind Traefik
2. **Adjust configuration** → [📝 Configuration](README-Konfiguration.en.md)
3. **Set up monitoring** → Prometheus + Grafana
4. **Integrate Mailcow** → See README.en.md Example 5

---

## 🔄 Updates

```bash
# Update repository
git pull

# Reload configs (automatic through watch: true)
# No restart needed!

# For docker-compose.yaml changes:
docker compose up -d --force-recreate

# For image updates:
docker compose pull
docker compose up -d
```

---

**Made with ❤️ by WSC - Web SEO Consulting**

This project is free and Open Source (GPL-3.0). If it helped you, I appreciate your support:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

[🏠 Back to Main Documentation](README.en.md) | [➡️ Continue to Configuration](README-Konfiguration.en.md)
