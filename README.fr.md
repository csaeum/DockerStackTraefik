# Traefik Reverse Proxy Stack

[![CI](https://github.com/csaeum/DockerStackTraefik/workflows/CI%20-%20Docker%20Stack%20Tests/badge.svg)](https://github.com/csaeum/DockerStackTraefik/actions)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Traefik Version](https://img.shields.io/badge/Traefik-v3.6-blue)](https://doc.traefik.io/traefik/)

[🇩🇪 Deutsch](README.md) | [🇬🇧 English](README.en.md) | 🇫🇷 Français

Stack Traefik v3.6 prêt pour la production avec les meilleures pratiques de sécurité, Let's Encrypt, HTTP/3, géo-blocage et middlewares complets.

## 🚀 Fonctionnalités

### Sécurité
- ✅ **TLS 1.3 uniquement** - Chiffrement maximum
- ✅ **En-têtes de sécurité** - HSTS, X-Frame-Options, CSP, etc.
- ✅ **Géo-blocage** - Bloque 23 pays à haut risque
- ✅ **Limitation de débit** - Protection DoS (100 req/s)
- ✅ **BasicAuth** - Pour le tableau de bord et les métriques
- ✅ **Let's Encrypt** - Certificats SSL automatiques

### Performance
- ✅ **Support HTTP/3** - Protocole QUIC
- ✅ **Compression Gzip/Brotli** - Réponses ~70% plus petites
- ✅ **Rotation des logs** - 7 jours, 5 sauvegardes, 50 Mo max

### Surveillance
- ✅ **Tableau de bord** - Interface Web pour Traefik
- ✅ **Métriques Prometheus** - Point de terminaison `/metrics`
- ✅ **Logs JSON** - Journalisation structurée

---

## 🆕 Démarrage rapide - Nouveau serveur

### 1. Prérequis

```bash
# Docker & Docker Compose installés?
docker --version
docker compose version

# Ports disponibles?
sudo netstat -tulpn | grep -E ':80|:443'
```

### 2. Cloner le dépôt

```bash
cd /opt
git clone https://github.com/csaeum/DockerStackTraefik.git
cd DockerStackTraefik
```

### 3. Configurer l'environnement

```bash
# Créer .env
cp .env.example .env

# Éditer .env
nano .env
```

**Variables importantes:**
```bash
COMPOSE_PROJECT_NAME=traefik
HOSTRULE=Host(`traefik.votre-domaine.com`)
LETSENCRYPT_EMAIL=votre-email@example.com
PROXY_NETWORK=traefik_proxy_network
TIMEZONE=Europe/Paris
```

### 4. Créer le réseau Docker

```bash
docker network create traefik_proxy_network
```

### 5. Préparer la structure des dossiers

```bash
mkdir -p logs volumes
chmod 700 volumes
chmod 755 logs
```

### 6. Démarrer Traefik

```bash
docker compose up -d
```

### 7. Validation

```bash
# Conteneur en cours d'exécution?
docker compose ps

# Vérifier les logs
docker compose logs -f traefik

# Tableau de bord accessible?
curl -I https://traefik.votre-domaine.com/dashboard/
# Attendu: HTTP/2 401 (Auth requise)
```

### 8. Connexion au tableau de bord

- URL: `https://traefik.votre-domaine.com/dashboard/`
- Utilisateur: `traefik-admin`
- Mot de passe: **Voir changement de mot de passe ci-dessous!**

---

## 🧩 Middlewares disponibles

Tous les middlewares sont définis dans `configs/traefik-dynamic.yaml` et peuvent être référencés via `@file`.

| Middleware | Fonction | Utilisation |
|------------|----------|-------------|
| `redirect-to-https@file` | Redirection HTTP → HTTPS | **REQUIS** pour le routeur HTTP |
| `redirect-to-www@file` | Redirection vers sous-domaine www | Optionnel pour les sites web |
| `geo-block@file` | Bloque 23 pays | **Recommandé** pour les services publics |
| `security-headers@file` | HSTS, X-Frame-Options, etc. | **Recommandé** pour tous les projets |
| `compression@file` | Compression Gzip/Brotli | **Recommandé** pour les performances |
| `rate-limit@file` | Protection DoS 100 req/s | **Recommandé** pour APIs & connexions |
| `in-flight-limit@file` | Max 100 requêtes simultanées | Optionnel pour charge élevée |

---

## 📦 Exemple complet de labels - Shopware 6

```yaml
services:
  shopware:
    image: shopware/production:latest
    container_name: shopware
    networks:
      - traefik_proxy_network
    labels:
      - traefik.enable=true

      # Routeur HTTP (Port 80 -> Redirection HTTPS)
      - traefik.http.routers.shopware-http.rule=Host(`shop.example.com`) || Host(`www.shop.example.com`)
      - traefik.http.routers.shopware-http.entrypoints=web-http
      - traefik.http.routers.shopware-http.middlewares=redirect-to-https@file

      # Routeur HTTPS (Port 443)
      - traefik.http.routers.shopware.rule=Host(`shop.example.com`) || Host(`www.shop.example.com`)
      - traefik.http.routers.shopware.entrypoints=websecure-https
      - traefik.http.routers.shopware.tls.certresolver=letsEncrypt
      - traefik.http.routers.shopware.tls.options=modern@file
      - traefik.http.routers.shopware.middlewares=redirect-to-www@file,geo-block@file,security-headers@file,compression@file,rate-limit@file

      # Service (Port Backend)
      - traefik.http.services.shopware.loadbalancer.server.port=8000

networks:
  traefik_proxy_network:
    external: true
```

---

## 📚 Documentation

| Fichier | Description |
|---------|-------------|
| `README.md` | Allemand - Vue d'ensemble complète |
| `README.en.md` | Anglais - Vue d'ensemble |
| `README.fr.md` | Ce fichier - Vue d'ensemble en français |
| `ENV-CONFIGURATION.md` | ⭐ Contrôle .env - mot de passe, limites de débit, etc. |
| `LABELS-CHECKLIST.md` | Modèles de labels & migration |
| `DEPLOYMENT.md` | Guide de déploiement & dépannage |
| `CHANGELOG.md` | Tous les changements en détail |
| `.env.example` | Modèle de variables d'environnement |

---

## 🔐 Changer le mot de passe du tableau de bord

**Maintenant contrôlable via .env!** Voir [`ENV-CONFIGURATION.md`](ENV-CONFIGURATION.md) pour les détails.

**Guide rapide:**

```bash
# 1. Générer le hash
docker run --rm httpd:alpine htpasswd -nbB traefik-admin "VOTRE_NOUVEAU_MOT_DE_PASSE"

# Sortie: traefik-admin:$apr1$xyz$abc

# 2. Entrer dans .env (échapper $ comme $$!)
# DASHBOARD_PASSWORD_HASH=$$apr1$$xyz$$abc
nano .env

# 3. Recréer le conteneur
docker compose up -d --force-recreate
```

**Important:** `$` doit être échappé comme `$$`!

---

## 📝 Licence

Ce projet est sous licence [GNU General Public License v3.0](LICENSE).

---

## 💖 Soutien

**Made with ❤️ by WSC - Web SEO Consulting**

Ce projet est gratuit et open source. Si cela vous a aidé, j'apprécie votre soutien:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

## 🙏 Crédits

Basé sur:
- [Traefik Official Docs](https://doc.traefik.io/)
- [OWASP Security Headers](https://owasp.org/www-project-secure-headers/)
- [Mozilla SSL Config](https://ssl-config.mozilla.org/)

---

## 🤝 Contribuer

Les contributions sont les bienvenues! Veuillez créer une pull request ou ouvrir un issue.

---

**Version:** 2025-12-28 | **Traefik:** v3.6 | **TLS:** 1.3 uniquement

---

© 2025 WSC - Web SEO Consulting. Tous droits réservés.
