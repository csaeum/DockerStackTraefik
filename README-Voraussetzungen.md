# Voraussetzungen

[🏠 Zurück zur Hauptdokumentation](README.md) | [🇬🇧 English Version](README-Voraussetzungen.en.md)

---

## 📋 System-Anforderungen

### Server
- **OS:** Linux (empfohlen: Debian 11+, Ubuntu 20.04+, CentOS 8+)
- **RAM:** Minimum 1 GB, empfohlen 2 GB+
- **CPU:** 1 Core minimum, 2+ Cores empfohlen
- **Disk:** 10 GB freier Speicher (für Logs und Zertifikate)
- **Netzwerk:** Öffentliche IPv4-Adresse

### Software
- **Docker:** Version 20.10+
- **Docker Compose:** Version 2.0+ (Plugin-Version)
- **Git:** Für Repository-Verwaltung
- **curl:** Für Health-Checks

---

## 🔐 Ports

Folgende Ports müssen auf dem Server **frei** sein:

| Port | Protokoll | Verwendung |
|------|-----------|------------|
| **80** | TCP | HTTP (wird auf HTTPS umgeleitet) |
| **443** | TCP | HTTPS (HTTP/2) |
| **443** | UDP | HTTP/3 (QUIC) |

### Ports prüfen

```bash
# Prüfe ob Ports frei sind
sudo netstat -tulpn | grep -E ':80|:443'
# Sollte leer sein oder nur Traefik zeigen
```

---

## 🌐 DNS

### A-Record für Traefik Dashboard

Erstelle einen DNS A-Record für dein Traefik Dashboard:

```
traefik.deine-domain.de  →  Server-IP (1.2.3.4)
```

**Wichtig:** Der DNS-Record muss **vor** dem ersten Start gesetzt sein, damit Let's Encrypt funktioniert!

### DNS propagieren lassen

```bash
# Prüfe DNS-Auflösung
dig +short traefik.deine-domain.de
# Sollte deine Server-IP zurückgeben

# Alternative
nslookup traefik.deine-domain.de
```

⏱️ **DNS-Propagation kann 5-60 Minuten dauern**

---

## 🔧 Docker Installation

Falls Docker noch nicht installiert ist:

### Debian/Ubuntu

```bash
# Docker installieren
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Docker Compose Plugin installieren
sudo apt-get update
sudo apt-get install docker-compose-plugin

# User zu docker-Gruppe hinzufügen (optional)
sudo usermod -aG docker $USER
newgrp docker
```

### Versionen prüfen

```bash
docker --version
# Docker version 20.10.x oder höher

docker compose version
# Docker Compose version v2.x.x oder höher
```

---

## 🛡️ Firewall

### UFW (Ubuntu/Debian)

```bash
# Ports freigeben
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp

# Status prüfen
sudo ufw status
```

### firewalld (CentOS/RHEL)

```bash
# Ports freigeben
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=443/udp
sudo firewall-cmd --reload

# Status prüfen
sudo firewall-cmd --list-all
```

### iptables (Manuell)

```bash
# Ports freigeben
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 443 -j ACCEPT

# Persistent machen
sudo netfilter-persistent save
```

---

## 📧 Email für Let's Encrypt

Du benötigst eine **gültige Email-Adresse** für Let's Encrypt:
- Wird für Zertifikats-Ablauf-Warnungen verwendet
- Wird für Rate-Limit-Benachrichtigungen verwendet
- **Wichtig:** Muss erreichbar sein!

---

## 🎯 Optionale Voraussetzungen

### Mailcow Integration (optional)

Falls du Mailcow mit Traefik integrieren möchtest:
- Mailcow muss bereits installiert sein
- Mailcow Container-Name: `mailcow-nginx-mailcow-1`
- Mailcow muss im gleichen Docker-Netzwerk sein

### Monitoring (optional)

Für Prometheus Metrics:
- Prometheus Server (separate Installation)
- Grafana für Dashboards (optional)

---

## ✅ Voraussetzungen-Checkliste

Prüfe vor der Installation:

- [ ] Linux Server mit Root-Zugriff
- [ ] Docker 20.10+ installiert
- [ ] Docker Compose 2.0+ installiert
- [ ] Ports 80, 443 (TCP+UDP) sind frei
- [ ] DNS A-Record für Dashboard gesetzt
- [ ] DNS ist propagiert (dig-Test erfolgreich)
- [ ] Firewall-Regeln konfiguriert
- [ ] Gültige Email-Adresse für Let's Encrypt
- [ ] Mindestens 10 GB freier Speicher

---

## 🚀 Weiter zur Installation

Alle Voraussetzungen erfüllt? → [📦 Installation starten](README-Installation.md)

---

**Made with ❤️ by WSC - Web SEO Consulting**

Dieses Projekt ist kostenlos und Open Source (GPL-3.0). Wenn es dir geholfen hat, freue ich mich über deine Unterstützung:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

[🏠 Zurück zur Hauptdokumentation](README.md)
