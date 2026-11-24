# 🛡️ Écrire un Dockerfile qui passe l'analyse de sécurité en CI

## 🎯 Objectif

Écrire un Dockerfile qui passe **du premier coup** les contrôles de sécurité
d'une pipeline CI/CD, et savoir corriger rapidement quand un contrôle échoue.

Les exemples utilisent GitHub Actions, Buildx et Trivy, mais les principes valent pour
tout outil de CI et tout scanner (l'équivalent GitLab CI avec Kaniko est indiqué au fil du texte).
Une mise en pratique du build sans push, du scan Trivy et de la publication après scan
est visible dans [pipeline-python-centralisee-portfolio](https://github.com/JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio).

Pour les bonnes pratiques générales (cache, taille, structure), voir
[bonnes pratiques Docker](../devops/docker-best-practices.md).

---

## 🔎 Ce que vérifie la CI

Une pipeline DevSecOps type enchaîne trois contrôles, à trois moments différents
(les noms de jobs sont indicatifs) :

| Job | Outil | Étape | Quand | Ce qui est vérifié | Blocage type |
|-----|-------|-------|-------|--------------------|--------------|
| **`dockerfile-check`** | `trivy config` (ou hadolint) | `check` | chaque commit et tag | le **texte** du Dockerfile : mauvaises configurations | `HIGH`, `CRITICAL` |
| **`docker-build-check`** | `docker/build-push-action` avec `push: false` | `build` | branches | le Dockerfile **se construit** réellement | toute erreur de build |
| **`trivy-scan`** | `trivy image` | `scan` | tags de release | l'**image construite** : vulnérabilités des paquets système et applicatifs, secrets, SBOM | `CRITICAL` (souvent avec `--ignore-unfixed`) |

```text
Branche :  check ─────────────► build
           dockerfile-check     docker-build-check
           (trivy config)       (build sans push)

Tag :      check ─────────────► build ─────────────► scan ──────────► publish
           dockerfile-check     docker-build         trivy-scan        tags définitifs
           (trivy config)       (push tag temporaire)(par digest)      si le scan réussit
```

Conséquences :

- un Dockerfile **bien écrit** (`dockerfile-check` vert) peut produire une image
  **vulnérable** (`trivy-scan` rouge) parce que l'image de base ou une dépendance
  contient une CVE : les deux se corrigent différemment ;
- sans `docker-build-check`, un Dockerfile cassé n'est découvert **qu'au tag de release**,
  au pire moment.

---

## 1️⃣ Passer `dockerfile-check` (`trivy config`)

Le job lit le Dockerfile **sans le construire** : il est rapide et tourne dès l'étape `check`.
Le seuil de blocage habituel est `HIGH,CRITICAL`.

Sur un projet existant, le contrôle peut être introduit en **non bloquant**
(`continue-on-error: true` sur GitHub Actions, `allow_failure: true` sur GitLab CI)
le temps que l'équipe corrige, puis rendu bloquant. Ce mode doit rester une transition, pas un contournement durable.

### Les règles qui bloquent le plus souvent

Identifiants Trivy (affichés `DS002`, `DS-0002` ou `AVD-DS-0002` selon la version) :

| Règle | Sévérité | Problème | Correction |
|-------|----------|----------|------------|
| DS002 | HIGH | l'image tourne en **root** (pas de `USER`, ou `USER root` en dernier) | créer un utilisateur et finir par `USER` |
| DS010 | CRITICAL | `sudo` dans un `RUN` | inutile : le build tourne déjà en root avant le `USER` |
| DS017 | HIGH | `apt-get update` seul dans un `RUN` | `update` et `install` dans le **même** `RUN` |
| DS021 | HIGH | `apt-get install` sans `-y` | ajouter `-y` |
| DS029 | HIGH | `apt-get install` sans `--no-install-recommends` | l'ajouter |
| DS025 | HIGH | `apk add` sans `--no-cache` | l'ajouter |
| DS015 / DS019 / DS020 | HIGH | `yum` / `dnf` / `zypper` sans nettoyage | `yum clean all` (ou équivalent) dans le même `RUN` |
| DS024 | HIGH | `apt-get dist-upgrade` | ne pas l'utiliser, mettre à jour l'image de base |
| DS022 | HIGH | `MAINTAINER` (obsolète) | `LABEL org.opencontainers.image.authors=...` |
| DS009 | HIGH | `WORKDIR` relatif | chemin absolu (`/app`) |
| DS016 | HIGH | plusieurs `CMD` | un seul `CMD` |
| DS001 | MEDIUM | `FROM image:latest` ou sans tag | tag précis, voire digest |
| DS026 | LOW | pas de `HEALTHCHECK` | ajouter une sonde |
| DS005 | LOW | `ADD` au lieu de `COPY` | `COPY` (sauf archive locale à extraire) |

### ❌ Avant : ce qui échoue

```dockerfile
FROM python:latest
MAINTAINER moi@example.com
RUN apt-get update
RUN apt-get install curl gcc
WORKDIR app
ADD . .
ENV DATABASE_PASSWORD=changeme
RUN pip install -r requirements.txt
CMD python main.py
```

Problèmes détectés : `latest` (DS001), `MAINTAINER` (DS022), `update` seul (DS017),
`install` sans `-y` ni `--no-install-recommends` (DS021, DS029), `WORKDIR` relatif (DS009),
`ADD` (DS005), secret dans `ENV`, exécution en root (DS002), pas de `HEALTHCHECK` (DS026).

### ✅ Après : ce qui passe

```dockerfile
# syntax=docker/dockerfile:1
ARG PYTHON_VERSION=3.12

# ── Étape de build : outils de compilation, jamais dans l'image finale ──────
FROM python:${PYTHON_VERSION}-slim AS build
RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /build
COPY requirements.txt ./
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

# ── Image finale : le strict nécessaire ─────────────────────────────────────
FROM python:${PYTHON_VERSION}-slim
LABEL org.opencontainers.image.authors="equipe@example.com"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/* \
    && rm -rf /wheels \
    && useradd --create-home --uid 10001 --shell /usr/sbin/nologin appuser

COPY --chown=appuser:appuser app ./app

USER 10001
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Points clés :

- **`USER` numérique** (`10001`) : certains contrôles (Kubernetes `runAsNonRoot`)
  ne peuvent pas vérifier un nom d'utilisateur, seulement un UID.
- **Le code appartient à l'utilisateur, pas l'inverse** : si l'application n'a pas
  besoin d'écrire dans `/app`, laisser les fichiers à root (lecture seule pour `appuser`)
  est encore plus sûr. `--chown` n'est utile que si elle doit y écrire.
- **`HEALTHCHECK` sans `curl`** : on réutilise Python plutôt que d'installer un outil de plus.
- **Forme exec** (`["…"]`) pour `CMD` : le processus reçoit bien les signaux d'arrêt.

---

## 2️⃣ Passer `docker-build-check` (build sans push)

Sur les branches, l'image est construite **sans être poussée** : on vérifie que le
Dockerfile fonctionne avant d'arriver au tag de release, sans encombrer le registre.

```yaml
docker-build-check:
  if: github.ref_type == 'branch'
  runs-on: ubuntu-latest
  permissions:
    contents: read
  steps:
    - uses: actions/checkout@v4
    - uses: docker/setup-buildx-action@v3
    - uses: docker/build-push-action@v6
      with:
        context: .
        file: Dockerfile
        push: false
```

Sur GitLab CI, l'équivalent est Kaniko avec `--no-push` (pas de démon Docker
privilégié sur le runner).

Les erreurs les plus fréquentes :

| Erreur | Cause | Correction |
|--------|-------|------------|
| `COPY failed: file not found` | fichier absent du **contexte** ou exclu par `.dockerignore` | vérifier le dossier de contexte passé au build et le `.dockerignore` |
| `pip` / `apt` ne trouve pas un paquet | version épinglée retirée, proxy, dépôt privé | mettre à jour la version, configurer le proxy ou le dépôt |
| `--mount=type=secret` non reconnu | builder sans BuildKit (ancien `docker build`, Kaniko) | utiliser Buildx, voir la partie secrets plus bas |
| erreur de permission après `USER` | une commande qui écrit dans un dossier de root est placée après `USER` | déplacer la commande avant `USER` ou donner le dossier à l'utilisateur |
| `FROM` d'une image privée refusé | le job n'est pas authentifié au registre | fournir les identifiants du registre au builder |

---

## 3️⃣ Passer `trivy-scan` (`trivy image`)

Le scan regarde **ce qui est installé** dans l'image, pas comment le Dockerfile est écrit.

### Réduire la surface d'attaque

Chaque paquet installé est une source potentielle de CVE.

| Choix | Effet sur le scan |
|-------|-------------------|
| `python:3.12` (Debian complète) | des centaines de paquets, beaucoup de CVE |
| `python:3.12-slim` | bon compromis, recommandé par défaut |
| `python:3.12-alpine` | très peu de paquets, mais musl peut poser problème aux wheels Python |
| `distroless` / images Chainguard | quasi aucun paquet système, pas de shell |

- **Multi-stage** : compilateurs, en-têtes et outils de build restent dans l'étape `build`.
- **`--no-install-recommends`** et suppression de `/var/lib/apt/lists/*`.
- **Ne pas installer d'outils de confort** (`vim`, `curl`, `wget`, `git`, `ssh`) dans l'image finale.

### Rester à jour

La plupart des CVE bloquantes sont **déjà corrigées en amont** : il suffit de reconstruire.

- **Épingler une version mineure** (`python:3.12-slim`) et **reconstruire régulièrement**
  (pipeline planifiée hebdomadaire) pour récupérer les correctifs.
- **Épingler par digest** pour la reproductibilité, et laisser Renovate le mettre à jour :

  ```dockerfile
  FROM python:3.12-slim@sha256:<digest>
  ```

- **Mettre à jour les dépendances applicatives** : Trivy analyse aussi les paquets Python,
  Node ou Java présents dans l'image. Un fichier de verrouillage à jour corrige la majorité des alertes.

### Ne rien laisser traîner : secrets et fichiers sensibles

`trivy image` cherche aussi les **secrets** dans les fichiers de l'image.

- **Jamais de secret dans `ENV`, `ARG` ou `COPY`** : il reste lisible dans les couches,
  même supprimé dans une couche suivante (`docker history`, extraction des couches).
- **Un `.dockerignore` strict**, pour ne pas copier `.env`, `.git` ou des clés :

  ```text
  # .dockerignore
  .git
  .github
  .env
  .env.*
  !.env.example
  **/__pycache__
  **/*.pyc
  .venv
  tests
  *.pem
  *.key
  ```

- **Secret nécessaire au build** (dépôt de paquets privé) : avec BuildKit,
  `RUN --mount=type=secret` le rend disponible le temps d'une commande, sans l'écrire dans une couche
  (`secrets:` de `docker/build-push-action`).
  ⚠️ Kaniko (GitLab CI) ne prend pas en charge `--mount=type=secret` : passer alors par une étape
  de build dont rien n'est copié dans l'image finale, ou télécharger les dépendances avant le build.

---

## 4️⃣ Quand le scan reste rouge

### Lire le résultat

Pour chaque vulnérabilité, Trivy indique le paquet, la version installée et la
**version corrigée** (`Fixed Version`).

| Situation | Action |
|-----------|--------|
| une version corrigée existe pour un paquet système | reconstruire sur une image de base à jour |
| une version corrigée existe pour une dépendance applicative | monter la version dans le fichier de verrouillage |
| pas de correctif disponible (`will_not_fix`, `affected`) | `--ignore-unfixed` en CI, surveiller |
| CVE non exploitable dans votre contexte | l'accepter explicitement dans `.trivyignore` |

### Accepter une vulnérabilité en connaissance de cause

Un fichier `.trivyignore` à la racine du projet, **une ligne par exception,
avec justification et date d'expiration** :

```text
# .trivyignore
# libxyz : fonction vulnérable non utilisée (pas d'appel réseau). Revue le 2026-11-22.
CVE-2026-12345 exp:2027-02-28

# Pas de HEALTHCHECK : la sonde est gérée par Kubernetes (livenessProbe).
DS026
```

- `exp:` : l'exception expire et la CI redevient bloquante, ce qui force à la réexaminer.
- L'ajout d'une ligne dans `.trivyignore` passe par une pull request revue, comme du code
  (au besoin, `CODEOWNERS` impose la revue de l'équipe sécurité sur ce fichier).
- Ne **jamais** abaisser le seuil de blocage ou passer le job en `continue-on-error`
  pour « faire passer » une pipeline : c'est désactiver le contrôle pour tout le monde.

---

## 💻 Reproduire les contrôles en local

Corriger en local est beaucoup plus rapide que d'enchaîner les pipelines.

```bash
# 1. Mauvaises configurations du Dockerfile (comme l'étape check)
trivy config --severity HIGH,CRITICAL Dockerfile

# Alternative / complément : hadolint (bonnes pratiques + shellcheck des RUN)
docker run --rm -i hadolint/hadolint < Dockerfile

# 2. Construire l'image (comme docker-build-check)
docker build -t mon-app:test .

# 3. Scanner l'image (comme trivy-scan)
trivy image --severity CRITICAL --ignore-unfixed mon-app:test

# Vue complète, non bloquante
trivy image --severity HIGH,CRITICAL mon-app:test

# 4. Inventaire des composants (SBOM)
trivy image --format cyclonedx -o sbom.cdx.json mon-app:test
```

---

## ⚙️ Exemple de jobs GitHub Actions

```yaml
jobs:
  dockerfile-check:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - run: |
          docker run --rm -v "$GITHUB_WORKSPACE:/work:ro" -w /work aquasec/trivy:0.74.0 \
            config --exit-code 1 --severity HIGH,CRITICAL Dockerfile

  # docker-build-check : voir la partie 2️⃣
  # docker-build : construit l'image et la pousse sous un tag temporaire
  #                (ici ghcr.io/<owner>/<repo>:<sha>), expose son digest en output

  trivy-scan:
    needs: [docker-build]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: read      # lire l'image sur ghcr.io
    env:
      IMAGE: ${{ needs.docker-build.outputs.image }}@${{ needs.docker-build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4   # pour lire .trivyignore
      - env:
          TRIVY_USERNAME: ${{ github.actor }}
          TRIVY_PASSWORD: ${{ github.token }}
        run: |
          run_trivy() {
            docker run --rm -e TRIVY_USERNAME -e TRIVY_PASSWORD \
              -v "$GITHUB_WORKSPACE:/work" -w /work aquasec/trivy:0.74.0 image "$@"
          }
          run_trivy --exit-code 0 --severity HIGH,CRITICAL --format template \
            --template "@/contrib/html.tpl" -o trivy-report.html "$IMAGE"
          run_trivy --exit-code 1 --severity CRITICAL --ignore-unfixed "$IMAGE"
      - if: always()
        uses: actions/upload-artifact@v4
        with:
          name: trivy-report
          path: trivy-report.html
```

- Scanner **par digest** (`image@sha256:…`), transmis par le job de build en output,
  garantit que l'image analysée est exactement celle qui sera publiée.
- Les tags définitifs (`1.2.3`, `latest`) ne sont posés qu'**après** un scan réussi,
  par copie de registre à registre (`crane tag`, `docker buildx imagetools create`) : aucune reconstruction.
- Le rapport HTML complet est conservé même quand le job échoue (`if: always()`).
- Le `GITHUB_TOKEN` n'a que les droits nécessaires à chaque job (`packages: read` pour le scan,
  `packages: write` uniquement pour le build et la publication).
- Sur GitLab CI, la même chaîne s'écrit avec `stage: scan`, `$CI_REGISTRY_IMAGE`
  et `artifacts: when: always` ; le build passe alors par **Kaniko** ou **Buildah**.

---

## ✅ Checklist avant de pousser

**Dockerfile**
- [ ] `FROM` avec un tag précis (ou un digest), jamais `latest`
- [ ] Image de base minimale (`-slim`, `distroless`)
- [ ] Multi-stage : outils de build absents de l'image finale
- [ ] `apt-get update && apt-get install -y --no-install-recommends … && rm -rf /var/lib/apt/lists/*` dans un seul `RUN`
- [ ] Pas de `sudo`, pas de `MAINTAINER`, pas d'`ADD` sans raison
- [ ] `WORKDIR` absolu
- [ ] `USER` non root (UID numérique) en dernière position
- [ ] `HEALTHCHECK` présent (ou exception justifiée)
- [ ] `CMD` / `ENTRYPOINT` en forme exec

**Contenu de l'image**
- [ ] `.dockerignore` excluant `.git`, `.env`, clés et tests
- [ ] Aucun secret dans `ENV`, `ARG`, `COPY` ou les couches
- [ ] Dépendances verrouillées et à jour
- [ ] Pas d'outils superflus (`curl`, `vim`, `ssh`…) dans l'image finale

**Processus**
- [ ] `trivy config`, `docker build` et `trivy image` lancés en local avant de pousser
- [ ] Build sans push sur les branches, pour détecter un Dockerfile cassé avant la release
- [ ] Toute exception dans `.trivyignore` est justifiée, datée et revue
- [ ] Reconstruction planifiée pour récupérer les correctifs de l'image de base

---

## 📚 Références

- [Trivy : contrôles Dockerfile](https://avd.aquasec.com/misconfig/dockerfile/)
- [Trivy : `.trivyignore`](https://trivy.dev/latest/docs/configuration/filtering/)
- [hadolint](https://github.com/hadolint/hadolint)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Bonnes pratiques de sécurité du dépôt](security-best-practices.md)
