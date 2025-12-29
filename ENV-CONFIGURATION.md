# Environment-Variable Konfiguration (.env)

Vollständige Anleitung zur Steuerung des Traefik-Stacks über die `.env` Datei.

## 📋 Inhaltsverzeichnis

- [Übersicht](#übersicht)
- [Verfügbare Variablen](#verfügbare-variablen)
- [Dashboard-Passwort ändern](#dashboard-passwort-ändern)
- [Rate-Limiting anpassen](#rate-limiting-anpassen)
- [Wichtige Hinweise](#wichtige-hinweise)
- [Beispiele](#beispiele)

---

## Übersicht

**Was ist steuerbar?**

| Kategorie | Variablen | Neustart erforderlich? |
|-----------|-----------|------------------------|
| **Projekt-Config** | `COMPOSE_PROJECT_NAME`, `RESTART`, `TIMEZONE` | Ja (docker compose up -d) |
| **Dashboard** | `HOSTRULE` | Ja |
| **Let's Encrypt** | `LETSENCRYPT_EMAIL` | Ja |
| **Authentication** | `DASHBOARD_USER`, `DASHBOARD_PASSWORD_HASH` | Ja |
| **Rate Limiting** | `RATE_LIMIT_*` | Ja |
| **In-Flight Limiting** | `IN_FLIGHT_LIMIT` | Ja |

**Vorteil:** Alle sensiblen Daten und Limits in **einer Datei** steuerbar!

---

## Verfügbare Variablen

### 1. Projekt-Konfiguration

```bash
# Container-Name und Projekt-Präfix
COMPOSE_PROJECT_NAME=traefik

# Restart-Policy (unless-stopped, always, no)
RESTART=unless-stopped

# Zeitzone für Logs
TIMEZONE=Europe/Berlin
```

**Anpassung:**
- `COMPOSE_PROJECT_NAME` - Nur ändern wenn du mehrere Traefik-Instanzen betreibst
- `RESTART` - `always` für produktive Server empfohlen
- `TIMEZONE` - Wichtig für korrekte Log-Timestamps

---

### 2. Dashboard-Konfiguration

```bash
# Domain für Traefik Dashboard
HOSTRULE=Host(`traefik.server-erde.web-seo-consulting.eu`)
```

**Format:**
- Einzelne Domain: `Host(\`traefik.example.com\`)`
- Mehrere Domains: `Host(\`traefik.example.com\`) || Host(\`traefik2.example.com\`)`

**Wichtig:** Backticks escapen: \`

---

### 3. Let's Encrypt

```bash
# Email für Let's Encrypt Benachrichtigungen
LETSENCRYPT_EMAIL=info@web-seo-consulting.eu
```

**Wichtig:**
- Gültige Email-Adresse
- Let's Encrypt sendet Ablauf-Warnungen an diese Email
- Rate-Limit-Benachrichtigungen gehen an diese Email

---

### 4. Netzwerk

```bash
# Externes Docker-Netzwerk für alle Projekte
PROXY_NETWORK=traefik_proxy_network
```

**Nur ändern wenn:**
- Du ein anderes Netzwerk-Schema verwendest
- Konflikt mit bestehendem Netzwerk

---

### 5. Dashboard Authentication

```bash
# Username für Dashboard-Login
DASHBOARD_USER=traefik-admin

# Passwort-Hash (siehe unten wie generieren)
DASHBOARD_PASSWORD_HASH=$$apr1$$u5m91va6$$jYOH.sK1gKMaLmWlNxA7m/
```

**Wichtig:**
- `$` muss in .env als `$$` escaped werden
- Hash enthält User + Passwort
- Niemals Plain-Text Passwort!

---

### 6. Rate Limiting

```bash
# Durchschnittliche Requests pro Zeitraum
RATE_LIMIT_AVERAGE=100

# Maximale Burst-Größe
RATE_LIMIT_BURST=50

# Zeitraum (1s, 1m, 1h)
RATE_LIMIT_PERIOD=1s
```

**Bedeutung:**
- `AVERAGE` - Durchschnittliche Rate (100 req/s)
- `BURST` - Kurzfristige Spitzen erlaubt (50 extra)
- `PERIOD` - Zeitfenster (1s = pro Sekunde)

**Beispiele:**
```bash
# Sehr restriktiv (API mit wenig Traffic)
RATE_LIMIT_AVERAGE=10
RATE_LIMIT_BURST=5

# Standard (für die meisten Websites)
RATE_LIMIT_AVERAGE=100
RATE_LIMIT_BURST=50

# Lockerer (High-Traffic Website)
RATE_LIMIT_AVERAGE=500
RATE_LIMIT_BURST=200

# Pro Minute statt pro Sekunde
RATE_LIMIT_AVERAGE=1000
RATE_LIMIT_BURST=200
RATE_LIMIT_PERIOD=1m
```

---

### 7. In-Flight Limiting

```bash
# Maximale gleichzeitige Connections pro IP
IN_FLIGHT_LIMIT=100
```

**Bedeutung:** Max 100 gleichzeitige Requests pro IP-Adresse

**Anpassungen:**
```bash
# Sehr restriktiv
IN_FLIGHT_LIMIT=10

# Standard
IN_FLIGHT_LIMIT=100

# Lockerer (für High-Traffic)
IN_FLIGHT_LIMIT=500
```

---

## Dashboard-Passwort ändern

### Schritt 1: Neuen Hash generieren

```bash
# Passwort-Hash generieren
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "DEIN_NEUES_PASSWORT"
```

**Output:**
```
traefik-admin:$apr1$xyz123$abc456def789
```

### Schritt 2: In .env eintragen

**Wichtig:** `$` muss als `$$` escaped werden!

```bash
# Vorher (Output vom Command):
traefik-admin:$apr1$xyz123$abc456def789

# Nachher (in .env):
DASHBOARD_PASSWORD_HASH=$$apr1$$xyz123$$abc456def789
```

### Schritt 3: Container neu starten

```bash
docker compose up -d --force-recreate
```

### Schritt 4: Testen

```bash
curl -u traefik-admin:DEIN_NEUES_PASSWORT https://traefik.deine-domain.de/dashboard/
```

---

## Rate-Limiting anpassen

### Use Case 1: API mit wenig Traffic

```bash
# .env
RATE_LIMIT_AVERAGE=20
RATE_LIMIT_BURST=10
RATE_LIMIT_PERIOD=1s
```

**Ergebnis:** Max 20 req/s + 10 Burst

---

### Use Case 2: High-Traffic Website

```bash
# .env
RATE_LIMIT_AVERAGE=500
RATE_LIMIT_BURST=200
RATE_LIMIT_PERIOD=1s
```

**Ergebnis:** Max 500 req/s + 200 Burst

---

### Use Case 3: Rate-Limiting pro Minute

```bash
# .env
RATE_LIMIT_AVERAGE=1000
RATE_LIMIT_BURST=300
RATE_LIMIT_PERIOD=1m
```

**Ergebnis:** Max 1000 req/min + 300 Burst

---

### Use Case 4: Kein Rate-Limiting (nur für Tests!)

**Nicht möglich über .env** - würde 0 bedeuten = Blockiert alles

**Lösung:** Middleware aus Router entfernen:

```yaml
# In docker-compose.yaml Zeile 43:
# Vorher:
- traefik.http.routers.traefik-dashboard.middlewares=dashboard-auth-env,rate-limit-env

# Nachher:
- traefik.http.routers.traefik-dashboard.middlewares=dashboard-auth-env
```

---

## Wichtige Hinweise

### ⚠️ Dollar-Zeichen Escaping

**Problem:** Docker Compose interpretiert `$` als Variable

**Lösung:** In `.env` immer `$$` verwenden

```bash
# ❌ FALSCH (funktioniert nicht!)
DASHBOARD_PASSWORD_HASH=$apr1$xyz$abc

# ✅ RICHTIG
DASHBOARD_PASSWORD_HASH=$$apr1$$xyz$$abc
```

---

### ⚠️ Backticks in HOSTRULE

**Problem:** Shell interpretiert Backticks als Command-Substitution

**Lösung:** Backticks escapen

```bash
# ✅ RICHTIG
HOSTRULE=Host(`traefik.example.com`)

# ❌ FALSCH (ohne Backticks)
HOSTRULE=Host(traefik.example.com)
```

---

### ⚠️ Änderungen aktivieren

**Nach jeder .env-Änderung:**

```bash
# Container neu erstellen (liest .env neu)
docker compose up -d --force-recreate

# Oder: Down + Up
docker compose down
docker compose up -d
```

**Nicht ausreichend:**
```bash
# ❌ Restart liest .env NICHT neu!
docker compose restart
```

---

### ⚠️ .env niemals committen

```bash
# Prüfen ob .env in .gitignore
cat .gitignore | grep .env

# Sollte ausgeben: .env

# Falls nicht:
echo ".env" >> .gitignore
```

---

## Beispiele

### Beispiel 1: Neuer Server Setup

```bash
# .env erstellen
cp .env.example .env

# .env anpassen
nano .env

# Wichtigste Variablen:
HOSTRULE=Host(`traefik.myserver.com`)
LETSENCRYPT_EMAIL=admin@myserver.com

# Passwort-Hash generieren
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "MeinSicheresPasswort123"

# Output in .env eintragen (mit $$ statt $)
DASHBOARD_PASSWORD_HASH=$$apr1$$xyz...

# Traefik starten
docker compose up -d
```

---

### Beispiel 2: Passwort ändern

```bash
# 1. Hash generieren
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "NeuesPasswort"

# 2. In .env eintragen (Zeile 31)
nano .env

# 3. Container neu erstellen
docker compose up -d --force-recreate

# 4. Testen
curl -u traefik-admin:NeuesPasswort https://traefik.myserver.com/dashboard/
```

---

### Beispiel 3: Rate-Limiting lockerer machen

```bash
# .env bearbeiten
nano .env

# Ändern:
RATE_LIMIT_AVERAGE=200   # War: 100
RATE_LIMIT_BURST=100     # War: 50

# Speichern + Container neu erstellen
docker compose up -d --force-recreate

# Prüfen
docker compose logs traefik | grep -i ratelimit
```

---

### Beispiel 4: Multi-Domain Dashboard

```bash
# .env bearbeiten
nano .env

# Ändern:
HOSTRULE=Host(`traefik.example.com`) || Host(`admin.example.com`)

# Speichern + Container neu erstellen
docker compose up -d --force-recreate

# Beide Domains testen
curl -I https://traefik.example.com/dashboard/
curl -I https://admin.example.com/dashboard/
```

---

## Validierung

### .env prüfen

```bash
# Env-Vars anzeigen
docker compose config | grep -A 50 environment

# Alle substituierten Werte sehen
docker compose config
```

---

### Dashboard-Auth testen

```bash
# Sollte 401 zurückgeben (Auth required)
curl -I https://traefik.deine-domain.de/dashboard/

# Mit Credentials
curl -u $DASHBOARD_USER:DEIN_PASSWORT https://traefik.deine-domain.de/dashboard/
```

---

### Rate-Limiting testen

```bash
# 200 schnelle Requests senden
for i in {1..200}; do
  curl -I https://traefik.deine-domain.de/dashboard/ 2>&1 | grep -E "HTTP|429"
done

# Sollte ab Request ~150 "429 Too Many Requests" zeigen
```

---

## Troubleshooting

### Problem: Passwort funktioniert nicht

**Ursache:** `$` nicht escaped

**Check:**
```bash
# In .env nachsehen
cat .env | grep DASHBOARD_PASSWORD_HASH

# Sollte $$ enthalten, NICHT $
```

**Fix:**
```bash
# Alle $ durch $$ ersetzen
nano .env
# Manuell ändern: $apr1$xyz -> $$apr1$$xyz
```

---

### Problem: Rate-Limiting greift nicht

**Ursache:** Container nicht neu erstellt

**Fix:**
```bash
docker compose up -d --force-recreate
```

---

### Problem: Traefik startet nicht nach .env-Änderung

**Ursache:** Syntax-Fehler in .env

**Check:**
```bash
# Config validieren
docker compose config

# Fehler sollte angezeigt werden
```

**Häufige Fehler:**
- Fehlende Anführungszeichen
- Spaces um `=`
- Unescaped Sonderzeichen

---

## Best Practices

### ✅ DO

- `.env` IMMER in `.gitignore`
- `$$` für Dollar-Zeichen in Passwort-Hashes
- Starke Passwörter generieren (min. 16 Zeichen)
- Rate-Limits nach Traffic anpassen
- `.env.example` im Repo für Dokumentation

### ❌ DON'T

- `.env` NIEMALS committen
- Plain-Text Passwörter in `.env`
- Rate-Limits zu niedrig (DoS gegen dich selbst)
- `docker compose restart` nach .env-Änderung (funktioniert nicht!)

---

## Mailcow Integration (Optional)

### Hinweis

Dieser Stack enthält eine vorkonfigurierte Mailcow ACME Challenge Integration. Die Mailcow-Domains sind **nicht** über .env steuerbar, sondern müssen in `configs/traefik-dynamic.yaml` angepasst werden.

**Warum nicht in .env?**
- Router-Rules sind zu komplex für Key-Value Format
- Mailcow-Integration wird selten geändert (einmal bei Setup)
- Dynamische File Provider unterstützt keine Env-Variablen in Rules

### Mailcow-Domains anpassen

```bash
nano configs/traefik-dynamic.yaml
# Zeile 134: mailcow-acme-challenge Router
```

**Standard-Domains (anpassen auf deine Installation):**
```yaml
rule: "(Host(`autodiscover.deine-domain.de`) || Host(`autoconfig.deine-domain.de`) || Host(`mail.deine-domain.de`)) && PathPrefix(`/.well-known/acme-challenge/`)"
```

**Nach Änderung:**
```bash
# Traefik lädt die Konfiguration automatisch neu (watch: true)
# Kein Neustart erforderlich!
```

**Siehe auch:** README.md Abschnitt "Beispiel 5: Mailcow verbinden"

---

## Weitere Steuerungsmöglichkeiten

### Was ist NICHT über .env steuerbar?

| Feature | Warum nicht? | Wo konfigurieren? |
|---------|--------------|-------------------|
| **Geo-Blocking Länder** | Array/Liste | `configs/traefik-dynamic.yaml` |
| **Security Headers** | Komplex | `configs/traefik-dynamic.yaml` |
| **TLS Cipher Suites** | Sicherheitskritisch | `configs/traefik-dynamic.yaml` |
| **Log-Rotation** | Selten geändert | `configs/traefik.yaml` |
| **Trusted IPs** | Netzwerk-abhängig | `configs/traefik.yaml` |

**Grund:** Diese Settings sind zu komplex für Key-Value Format oder ändern sich sehr selten.

---

## Zusammenfassung

### Was du in .env steuern kannst:

✅ Dashboard-Passwort
✅ Rate-Limiting (Average, Burst, Period)
✅ In-Flight Limiting
✅ Dashboard-Domain
✅ Let's Encrypt Email
✅ Container-Name & Restart-Policy

### Was du in Config-Files steuern musst:

🔧 Geo-Blocking Länder
🔧 Security Headers
🔧 TLS-Version & Cipher Suites
🔧 Log-Rotation
🔧 Trusted IPs

---

**Du kannst jetzt den gesamten Stack über die .env steuern - ohne Config-Files anzufassen!** 🎉
