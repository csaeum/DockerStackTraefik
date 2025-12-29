# Installation

[🏠 Zurück zur Hauptdokumentation](README.md) | [🇬🇧 English Version](README-Installation.en.md)

---

## 📦 Schnellstart

### 1. Repository klonen

```bash
# Wechsle ins gewünschte Verzeichnis
cd /opt  # oder dein bevorzugter Pfad

# Clone das Repository
git clone https://github.com/csaeum/DockerStackTraefik.git
cd DockerStackTraefik
```

---

### 2. Environment-Datei erstellen

```bash
# Kopiere .env.example zu .env
cp .env.example .env

# Bearbeite die .env Datei
nano .env
```

---

### 3. Environment-Variablen konfigurieren

Passe mindestens diese Variablen in der `.env` an:

```bash
# Projekt-Name (für Container-Namen)
COMPOSE_PROJECT_NAME=traefik

# Timezone
TIMEZONE=Europe/Berlin

# Domain für Traefik Dashboard
HOSTRULE=Host(`traefik.deine-domain.de`)

# Email für Let's Encrypt
LETSENCRYPT_EMAIL=deine-email@example.com

# Netzwerk-Name
PROXY_NETWORK=traefik_proxy_network

# Restart-Policy
RESTART=unless-stopped
```

---

### 4. Dashboard-Passwort generieren

**Wichtig:** Ändere das Standard-Passwort!

```bash
# Generiere einen Passwort-Hash
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "DEIN_SICHERES_PASSWORT"

# Output: traefik-admin:$apr1$xyz123...
```

**Trage den Hash in die `.env` ein:**

```bash
# WICHTIG: $ muss als $$ escaped werden!
DASHBOARD_USER=traefik-admin
DASHBOARD_PASSWORD_HASH=$$apr1$$xyz123$$...
```

**Beispiel:**
```bash
# Generiert:
traefik-admin:$apr1$u5m91va6$jYOH.sK1gKMaLmWlNxA7m/

# In .env eintragen:
DASHBOARD_PASSWORD_HASH=$$apr1$$u5m91va6$$jYOH.sK1gKMaLmWlNxA7m/
```

---

### 5. Docker-Netzwerk erstellen

```bash
# Erstelle das externe Netzwerk für alle Services
docker network create traefik_proxy_network
```

**Hinweis:** Dieses Netzwerk wird von allen Services geteilt, die hinter Traefik laufen sollen.

---

### 6. Ordnerstruktur vorbereiten

```bash
# Erstelle notwendige Verzeichnisse
mkdir -p logs volumes

# Setze Berechtigungen
chmod 700 volumes  # Nur Root-Zugriff
chmod 755 logs     # Lesbar für Logs
```

---

### 7. Konfiguration validieren

```bash
# Prüfe die docker-compose.yaml Syntax
docker compose config

# Sollte keine Fehler zeigen und die finale Config ausgeben
```

---

### 8. Traefik starten

```bash
# Starte Traefik
docker compose up -d

# Logs anzeigen
docker compose logs -f traefik
```

**Warte auf:**
```
✅ Server listening on :80
✅ Server listening on :443 (TCP/HTTP/2)
✅ Server listening on :443 (UDP/HTTP/3)
✅ Certificate obtained for domain [traefik.deine-domain.de]
```

---

## ✅ Installation validieren

### 1. Container-Status prüfen

```bash
# Prüfe ob Container läuft
docker compose ps

# Sollte zeigen: traefik (running)
```

### 2. Dashboard testen

```bash
# HTTP → HTTPS Redirect testen
curl -I http://traefik.deine-domain.de
# Expect: 301 Moved Permanently
# Location: https://traefik.deine-domain.de/

# Dashboard aufrufen (mit BasicAuth)
curl -I https://traefik.deine-domain.de/dashboard/ \
  -u traefik-admin:DEIN_PASSWORT
# Expect: 200 OK
```

### 3. TLS-Zertifikat prüfen

```bash
# Prüfe Let's Encrypt Zertifikat
openssl s_client -connect traefik.deine-domain.de:443 -servername traefik.deine-domain.de < /dev/null 2>/dev/null | openssl x509 -noout -text | grep -E 'Issuer|Subject|Not After'

# Sollte zeigen:
# Issuer: CN = R3 (Let's Encrypt)
# Subject: CN = traefik.deine-domain.de
# Not After: (Ablaufdatum in 90 Tagen)
```

### 4. HTTP/3 testen

```bash
# HTTP/3 Support prüfen (benötigt curl mit HTTP/3)
curl -I --http3 https://traefik.deine-domain.de/dashboard/

# Oder im Browser:
# Chrome DevTools → Network → Protocol sollte "h3" zeigen
```

### 5. Security Headers prüfen

```bash
# Prüfe Security Headers
curl -I https://traefik.deine-domain.de/dashboard/ \
  -u traefik-admin:DEIN_PASSWORT | grep -E 'Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options'

# Sollte zeigen:
# Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
# X-Frame-Options: DENY
# X-Content-Type-Options: nosniff
```

---

## 🔧 Troubleshooting

### Problem: Container startet nicht

```bash
# Logs prüfen
docker compose logs traefik

# Häufige Fehler:
# - Port 80/443 bereits belegt → sudo netstat -tulpn | grep -E ':80|:443'
# - .env Fehler → docker compose config
# - Netzwerk fehlt → docker network create traefik_proxy_network
```

### Problem: Let's Encrypt Fehler

```bash
# Prüfe DNS-Auflösung
dig +short traefik.deine-domain.de

# Prüfe ob Domain erreichbar ist
curl -I http://traefik.deine-domain.de

# Logs für ACME-Fehler prüfen
docker compose logs traefik | grep -i acme

# Häufige Fehler:
# - DNS zeigt nicht auf Server → DNS-Propagation abwarten
# - Firewall blockiert Port 80/443 → Firewall prüfen
# - Rate Limit erreicht → 1 Woche warten oder Staging nutzen
```

### Problem: Dashboard nicht erreichbar

```bash
# Prüfe Container-Status
docker compose ps

# Prüfe Ports
docker compose port traefik 443

# Prüfe BasicAuth
# WICHTIG: $ muss als $$ escaped sein in .env!
cat .env | grep DASHBOARD_PASSWORD_HASH
```

### Problem: Passwort funktioniert nicht

```bash
# Prüfe ob $$ escaped sind
grep DASHBOARD_PASSWORD_HASH .env
# Sollte $$ zeigen, nicht $

# Neu generieren falls nötig
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "NEUES_PASSWORT"

# In .env eintragen ($ → $$)
nano .env

# Container neu starten
docker compose up -d --force-recreate
```

---

## 🚀 Nächste Schritte

Installation erfolgreich? Jetzt kannst du:

1. **Services hinzufügen** → Andere Container hinter Traefik schalten
2. **Konfiguration anpassen** → [📝 Konfiguration](README-Konfiguration.md)
3. **Monitoring einrichten** → Prometheus + Grafana
4. **Mailcow integrieren** → Siehe README.md Beispiel 5

---

## 🔄 Updates

```bash
# Repository aktualisieren
git pull

# Configs neu laden (automatisch durch watch: true)
# Kein Neustart nötig!

# Bei docker-compose.yaml Änderungen:
docker compose up -d --force-recreate

# Bei Image-Updates:
docker compose pull
docker compose up -d
```

---

**Made with ❤️ by WSC - Web SEO Consulting**

Dieses Projekt ist kostenlos und Open Source (GPL-3.0). Wenn es dir geholfen hat, freue ich mich über deine Unterstützung:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

[🏠 Zurück zur Hauptdokumentation](README.md) | [➡️ Weiter zur Konfiguration](README-Konfiguration.md)
