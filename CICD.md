# CI/CD — Traccar Web

Pipeline GitHub Actions complet : **build Docker → push registry privé → deploy VPS**.

---

## Architecture

```
Git push (master / deploy)
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  GitHub Actions                                      │
│                                                      │
│  JOB 1 — build-push                                  │
│  ┌──────────────────────────────────────────────┐   │
│  │  1. Checkout code                             │   │
│  │  2. Docker Buildx (cache GHA)                 │   │
│  │  3. Génère les tags (sha / branch / latest)   │   │
│  │  4. Login registry privé                      │   │
│  │  5. Build multi-stage (node → nginx)          │   │
│  │  6. Push image → registry privé               │   │
│  └──────────────────┬───────────────────────────┘   │
│                     │ needs: build-push               │
│  JOB 2 — deploy     ▼                                │
│  ┌──────────────────────────────────────────────┐   │
│  │  1. SCP docker-compose.yml → VPS             │   │
│  │  2. SSH: login registry                       │   │
│  │  3. SSH: docker compose pull                  │   │
│  │  4. SSH: docker compose up --wait             │   │
│  │  5. SSH: health check (rollback si KO)        │   │
│  │  6. SSH: docker image prune                   │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
         │
         ▼
   VPS — conteneur traccar-web sur :3000
```

---

## Fichiers créés

| Fichier | Rôle |
|---|---|
| `Dockerfile` | Multi-stage : `node:22-alpine` (build) → `nginx:1.27-alpine` (serve) |
| `nginx.conf` | SPA routing, gzip, cache immutable Vite |
| `docker-compose.yml` | Définition du service (base) |
| `docker-compose.prod.yml` | Override production (port, limites mémoire, logs) |
| `.dockerignore` | Exclut `node_modules`, `.git`, `.env*`, etc. |
| `.github/workflows/deploy.yml` | Pipeline CI/CD |

---

## 1 — Prérequis VPS

### 1.1 Installer Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

### 1.2 Créer le dossier de déploiement et le réseau Docker

```bash
sudo mkdir -p /opt/traccar-web
sudo chown $USER:$USER /opt/traccar-web
docker network create traccar-net
```

### 1.3 Créer la clé SSH dédiée (sur votre machine locale)

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_deploy
```

Copier la clé publique sur le VPS :

```bash
ssh-copy-id -i ~/.ssh/github_deploy.pub user@IP_VPS
```

Garder la clé **privée** (`~/.ssh/github_deploy`) pour l'ajouter en secret GitHub (voir section suivante).

---

## 2 — Secrets GitHub à configurer

Aller dans **GitHub → Settings → Secrets and variables → Actions → New repository secret**.

| Secret | Description | Exemple |
|---|---|---|
| `REGISTRY_URL` | Hôte du registry privé | `docker.io` ou `ghcr.io` ou `registry.mondomaine.com` |
| `REGISTRY_IMAGE` | Nom de l'image | `monorg/traccar-web` |
| `REGISTRY_USERNAME` | Login registry | `monusername` |
| `REGISTRY_PASSWORD` | Token/password registry | `dckr_pat_xxxxx` |
| `VPS_HOST` | IP ou domaine du VPS | `1.2.3.4` |
| `VPS_USER` | Utilisateur SSH | `ubuntu` |
| `VPS_SSH_KEY` | Contenu entier de la clé privée | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `VPS_PORT` | Port SSH (optionnel) | `22` |
| `DEPLOY_PATH` | Dossier de déploiement (optionnel) | `/opt/traccar-web` |
| `APP_URL` | URL publique de l'app | `https://traccar.mondomaine.com` |

### Obtenir un token Docker Hub

Docker Hub → **Account Settings → Security → New Access Token** → copiez le token dans `REGISTRY_PASSWORD`.

### Utiliser GitHub Container Registry (GHCR) à la place

```
REGISTRY_URL     = ghcr.io
REGISTRY_IMAGE   = votre-github-username/traccar-web
REGISTRY_USERNAME = votre-github-username
REGISTRY_PASSWORD = un Personal Access Token GitHub avec scope packages:write
```

---

## 3 — Déclencheurs du pipeline

| Événement | build-push | deploy |
|---|---|---|
| `push` sur `master` | oui | oui |
| `push` sur `deploy` | oui | oui |
| Pull Request → `master` | non | non |
| `workflow_dispatch` (manuel) | oui | oui |

> Les PR ne déclenchent ni build Docker ni deploy — seulement le lint/build existants (`lint.yml`, `build.yml`).

---

## 4 — Tags Docker générés automatiquement

Pour un push sur `master` avec le SHA `abc1234` :

| Tag | Valeur |
|---|---|
| SHA court | `monorg/traccar-web:sha-abc1234` |
| Branche | `monorg/traccar-web:master` |
| Latest | `monorg/traccar-web:latest` |

Pour un tag git `v6.12.2` :

| Tag | Valeur |
|---|---|
| Semver | `monorg/traccar-web:6.12.2` |

---

## 5 — Déploiement manuel

Depuis l'interface GitHub, aller dans **Actions → CI/CD — Build, Push & Deploy → Run workflow**.

---

## 6 — Fonctionnement du HEALTHCHECK

Le `Dockerfile` embarque un HEALTHCHECK nginx :

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost/favicon.ico || exit 1
```

La commande `docker compose up --wait` attend que le conteneur soit `healthy` avant de continuer. Si le conteneur est `unhealthy`, le pipeline effectue un rollback automatique (`docker compose down`) et renvoie une erreur.

---

## 7 — Vérifier le déploiement sur le VPS

```bash
# Statut du conteneur
docker compose -f /opt/traccar-web/docker-compose.yml \
               -f /opt/traccar-web/docker-compose.prod.yml ps

# Logs en temps réel
docker logs -f traccar-web

# Test HTTP local
curl -I http://localhost:3000
```

---

## 8 — Variables d'environnement Vite (build-time)

Les variables Vite doivent être injectées **au moment du build** (pas au runtime). Pour ajouter une variable :

1. Ajouter le secret dans GitHub : ex. `VITE_API_URL`
2. L'exposer comme `build-arg` dans `deploy.yml` :

```yaml
build-args: |
  VITE_API_URL=${{ secrets.VITE_API_URL }}
```

3. L'accepter dans le `Dockerfile` (avant `npm run build`) :

```dockerfile
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build
```

---

## 9 — Structure des fichiers Docker Compose

```
docker-compose.yml          ← définition du service (image, réseau)
docker-compose.prod.yml     ← override prod (port 3000, limites ressources, logs)
```

En production les deux fichiers sont mergés :

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Le service est accessible sur `http://IP_VPS:3000`. Si vous avez un reverse proxy (Nginx, Caddy, NPM), pointez-le vers ce port.

---

## 10 — Reverse proxy Nginx (optionnel)

Si vous gérez Nginx directement sur le VPS :

```nginx
server {
    listen 80;
    server_name traccar.mondomaine.com;

    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

Pour HTTPS avec Certbot :

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d traccar.mondomaine.com
```

---

## Résumé des commandes utiles

```bash
# Build local de l'image
docker build -t traccar-web:local .

# Test local du conteneur
docker run --rm -p 3000:80 traccar-web:local

# Voir les images en registry
docker images | grep traccar-web

# Forcer un redéploiement sans code change
# → GitHub Actions → Run workflow (workflow_dispatch)
```
