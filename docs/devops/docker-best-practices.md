# 🐳 Bonnes pratiques Docker

## 🎯 Objectif

Construire des images **légères, rapides à construire, reproductibles et sûres**,
et les exploiter proprement.

Pour la partie sécurité et le passage des scans en CI (Trivy, hadolint), voir le guide dédié :
[écrire un Dockerfile qui passe l'analyse de sécurité](../security/dockerfile-security.md).

---

## 🧱 Choisir l'image de base

- Partir d'une **image officielle** ou maintenue par un éditeur de confiance.
- **Épingler la version** : `python:3.12-slim`, jamais `latest` ni une image sans tag.
- Préférer les variantes **`-slim`** ou **`distroless`** : moins de paquets, image plus petite, moins de CVE.
- Utiliser la même image de base pour toute l'équipe ou l'organisation (images « socles »
  construites et maintenues dans un dépôt dédié).

---

## ⚡ Exploiter le cache de build

Docker réutilise une couche tant que l'instruction et ses fichiers n'ont pas changé.
**Ordonner du moins changeant au plus changeant** :

```dockerfile
FROM python:3.12-slim
WORKDIR /app

# 1. Dépendances : changent rarement → couche en cache la plupart du temps
COPY pyproject.toml uv.lock ./
RUN pip install --no-cache-dir uv && uv sync --frozen --no-dev

# 2. Code : change à chaque commit → seules ces couches sont reconstruites
COPY app ./app
```

❌ `COPY . .` en début de Dockerfile invalide le cache à chaque modification du code
et réinstalle toutes les dépendances.

---

## 🪜 Multi-stage build

Séparer **ce qui sert à construire** de **ce qui sert à exécuter** :

```dockerfile
FROM node:22-alpine AS build
WORKDIR /src
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY --from=build /src/dist /usr/share/nginx/html
```

L'image finale ne contient ni Node, ni `node_modules`, ni le code source : seulement le résultat du build.

---

## 📦 Réduire la taille

- Regrouper les commandes liées dans un seul `RUN` et **nettoyer dans la même couche** :

  ```dockerfile
  RUN apt-get update \
      && apt-get install -y --no-install-recommends libpq5 \
      && rm -rf /var/lib/apt/lists/*
  ```

  Un fichier supprimé dans une couche suivante occupe toujours de la place dans l'image.
- Désactiver les caches des gestionnaires de paquets (`pip --no-cache-dir`, `apk --no-cache`).
- Un **`.dockerignore`** pour ne pas envoyer `.git`, `node_modules`, `.venv`, les tests
  ou les fichiers `.env` dans le contexte de build.
- Vérifier le résultat :

  ```bash
  docker image ls mon-app
  docker history mon-app:1.0.0
  ```

  [dive](https://github.com/wagoodman/dive) permet d'explorer le contenu de chaque couche.

---

## ▶️ Démarrage et exécution

- **Un processus principal par conteneur** : base de données, API et worker dans des conteneurs séparés.
- **Forme exec** pour `CMD` et `ENTRYPOINT`, afin que le processus reçoive les signaux d'arrêt (`SIGTERM`) :

  ```dockerfile
  CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
  ```

  ❌ `CMD uvicorn app.main:app` lance un shell intermédiaire qui ne transmet pas les signaux :
  le conteneur est tué brutalement après le délai d'arrêt.
- **Logs sur la sortie standard** (`stdout` / `stderr`), jamais dans des fichiers du conteneur
  (`PYTHONUNBUFFERED=1` en Python).
- **`HEALTHCHECK`** pour que l'orchestrateur sache si l'application répond.
- **Utilisateur non root** (`USER`), voir le [guide sécurité](../security/dockerfile-security.md).

---

## ⚙️ Configuration

- **Même image pour tous les environnements** : seule la configuration change
  (variables d'environnement, fichiers montés), jamais l'image.
- Aucune valeur propre à un environnement ni aucun secret dans l'image.
- `ARG` pour les paramètres de build (version de Python), `ENV` pour les valeurs d'exécution par défaut.

---

## 🏷️ Tags et traçabilité

- **Tag de version immuable** (`1.4.2`) pour chaque release ; `latest` uniquement comme alias de commodité.
- Déployer **par tag de version ou par digest**, jamais `latest`.
- Ajouter les **labels OCI** pour retrouver l'origine d'une image :

  ```dockerfile
  ARG VCS_REF
  ARG VERSION
  LABEL org.opencontainers.image.source="https://github.com/organisation/projet" \
        org.opencontainers.image.revision="${VCS_REF}" \
        org.opencontainers.image.version="${VERSION}"
  ```

  ```bash
  docker build --build-arg VCS_REF="$GITHUB_SHA" --build-arg VERSION=1.4.2 -t mon-app:1.4.2 .
  ```

  Ou directement via les options du builder en CI (`labels:` de `docker/build-push-action`, `--label` avec `docker build`).
  Sur GHCR, le label `org.opencontainers.image.source` relie aussi l'image à son dépôt.

---

## 🏗️ Construire en CI

- Sur les runners GitHub hébergés, **Buildx** (`docker/setup-buildx-action` + `docker/build-push-action`) ;
  sur des runners auto-hébergés partagés, préférer **Buildah** ou **Kaniko** au Docker-in-Docker : pas de démon privilégié.
- Activer le **cache de build** distant (`cache-from` / `cache-to: type=gha` avec Buildx) pour accélérer les pipelines.
- Enchaîner **build → scan → publication** : une image n'est publiée sous ses tags
  définitifs qu'après avoir passé le scan.

Exemple complet : [pipeline-python-centralisee-portfolio](https://github.com/JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio).

---

## 🧪 Docker Compose en local

- Un `compose.yaml` pour lancer l'application et ses dépendances (base, cache) en une commande.
- Variables dans un `.env` local non versionné, avec un `.env.example` versionné.
- Volumes nommés pour les données, `depends_on` avec `condition: service_healthy`.

```yaml
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes: [db-data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

volumes:
  db-data:
```

---

## ✅ Checklist

- [ ] Image de base officielle, minimale, version épinglée
- [ ] Dépendances copiées et installées avant le code (cache)
- [ ] Multi-stage si une étape de build existe
- [ ] Nettoyage dans la même couche, `.dockerignore` présent
- [ ] Un processus par conteneur, `CMD` en forme exec, logs sur stdout
- [ ] `HEALTHCHECK` et utilisateur non root
- [ ] Même image pour tous les environnements, configuration externe
- [ ] Tags de version immuables, labels OCI
- [ ] Contrôles de sécurité passés ([guide](../security/dockerfile-security.md))
