# Prerequisites

[🏠 Back to Main Documentation](README.en.md) | [🇩🇪 Deutsche Version](README-Voraussetzungen.md)

---

## 📋 System Requirements

### Server
- **OS:** Linux (recommended: Debian 11+, Ubuntu 20.04+, CentOS 8+)
- **RAM:** Minimum 1 GB, recommended 2 GB+
- **CPU:** 1 Core minimum, 2+ Cores recommended
- **Disk:** 10 GB free space (for logs and certificates)
- **Network:** Public IPv4 address

### Software
- **Docker:** Version 20.10+
- **Docker Compose:** Version 2.0+ (Plugin version)
- **Git:** For repository management
- **curl:** For health checks

---

## 🔐 Ports

The following ports must be **available** on the server:

| Port | Protocol | Usage |
|------|----------|-------|
| **80** | TCP | HTTP (redirects to HTTPS) |
| **443** | TCP | HTTPS (HTTP/2) |
| **443** | UDP | HTTP/3 (QUIC) |

### Check Ports

```bash
# Check if ports are free
sudo netstat -tulpn | grep -E ':80|:443'
# Should be empty or only show Traefik
```

---

## 🌐 DNS

### A-Record for Traefik Dashboard

Create a DNS A-Record for your Traefik Dashboard:

```
traefik.your-domain.com  →  Server-IP (1.2.3.4)
```

**Important:** The DNS record must be set **before** the first start for Let's Encrypt to work!

### Let DNS Propagate

```bash
# Check DNS resolution
dig +short traefik.your-domain.com
# Should return your server IP

# Alternative
nslookup traefik.your-domain.com
```

⏱️ **DNS propagation can take 5-60 minutes**

---

## 🔧 Docker Installation

If Docker is not yet installed:

### Debian/Ubuntu

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose plugin
sudo apt-get update
sudo apt-get install docker-compose-plugin

# Add user to docker group (optional)
sudo usermod -aG docker $USER
newgrp docker
```

### Check Versions

```bash
docker --version
# Docker version 20.10.x or higher

docker compose version
# Docker Compose version v2.x.x or higher
```

---

## 🛡️ Firewall

### UFW (Ubuntu/Debian)

```bash
# Open ports
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp

# Check status
sudo ufw status
```

### firewalld (CentOS/RHEL)

```bash
# Open ports
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=443/udp
sudo firewall-cmd --reload

# Check status
sudo firewall-cmd --list-all
```

### iptables (Manual)

```bash
# Open ports
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 443 -j ACCEPT

# Make persistent
sudo netfilter-persistent save
```

---

## 📧 Email for Let's Encrypt

You need a **valid email address** for Let's Encrypt:
- Used for certificate expiration warnings
- Used for rate limit notifications
- **Important:** Must be accessible!

---

## 🎯 Optional Prerequisites

### Mailcow Integration (optional)

If you want to integrate Mailcow with Traefik:
- Mailcow must already be installed
- Mailcow container name: `mailcow-nginx-mailcow-1`
- Mailcow must be in the same Docker network

### Monitoring (optional)

For Prometheus Metrics:
- Prometheus Server (separate installation)
- Grafana for dashboards (optional)

---

## ✅ Prerequisites Checklist

Check before installation:

- [ ] Linux server with root access
- [ ] Docker 20.10+ installed
- [ ] Docker Compose 2.0+ installed
- [ ] Ports 80, 443 (TCP+UDP) are free
- [ ] DNS A-Record for dashboard is set
- [ ] DNS is propagated (dig test successful)
- [ ] Firewall rules configured
- [ ] Valid email address for Let's Encrypt
- [ ] At least 10 GB free disk space

---

## 🚀 Continue to Installation

All prerequisites met? → [📦 Start Installation](README-Installation.en.md)

---

**Made with ❤️ by WSC - Web SEO Consulting**

This project is free and Open Source (GPL-3.0). If it helped you, I appreciate your support:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

[🏠 Back to Main Documentation](README.en.md)
