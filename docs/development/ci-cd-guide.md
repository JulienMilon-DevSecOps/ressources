# 🔄 CI/CD Guide

## 📌 Introduction

Ce document propose un **exemple de mise en place d’un pipeline CI/CD** applicable à la majorité des projets modernes.
Il a pour objectif de servir de base et peut être adapté selon le contexte technique (stack, infrastructure, organisation). Les exemples sont donnés pour GitHub Actions, avec leur équivalent GitLab CI. Une pipeline GitHub Actions complète, centralisée et réutilisable est disponible dans [pipeline-python-centralisee-portfolio](https://github.com/JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio).

---

## 🎯 Objectif

L’objectif de la CI/CD est d’automatiser et de sécuriser le cycle de vie du développement logiciel, depuis l’écriture du code jusqu’à sa mise en production.

Plus concrètement, cela permet de :

### ⚡ Accélérer le développement

* Réduire le temps entre une idée et sa mise en production
* Automatiser les tâches répétitives (tests, build, déploiement)

### 🧪 Améliorer la qualité du code

* Détecter rapidement les erreurs grâce aux tests automatisés
* Appliquer des règles de qualité (linting, formatting)
* Éviter les régressions

### 🔐 Sécuriser les déploiements

* Standardiser les processus de release
* Réduire les erreurs humaines
* Garantir que chaque version déployée est validée

### 🤝 Faciliter la collaboration

* Permettre à plusieurs développeurs de travailler simultanément
* Valider automatiquement les contributions via Pull Requests
* Assurer une base de code stable

### 📈 Assurer la traçabilité

* Historique des builds et déploiements
* Versionning clair via tags
* Possibilité de rollback

---

## 🧩 Concepts

### 🔁 Continuous Integration (CI)

La **CI** consiste à intégrer régulièrement le code dans une branche commune (souvent `develop` ou `main`) et à vérifier automatiquement sa validité.

#### 🔍 Principes clés :

* Chaque modification déclenche un pipeline automatique
* Le code est testé et validé en continu
* Les erreurs sont détectées le plus tôt possible

#### 📌 Dans ce projet :

* Déclenchée à chaque **Pull Request**
* Permet de :

  * Lancer les tests
  * Vérifier la qualité du code
  * Construire le projet

👉 Objectif : empêcher l’introduction de code instable

---

### 🚀 Continuous Delivery (CD)

La **Continuous Delivery** consiste à préparer automatiquement une version prête à être déployée.

Cette partie peut parfois être une étapes ajouter après la CI pour construire l'image applicative. Elle est activée lors de la création de tag d'un certain format du type semVer `1.0.0` ou `v-1.0.0`.

#### 🔍 Principes :

* Le code est toujours dans un état “déployable”
* Les artefacts sont générés automatiquement
* Le déploiement peut être déclenché manuellement ou automatiquement

---

### 🚀 Continuous Deployment

La **Continuous Deployment** va plus loin :

* Chaque changement validé peut être **déployé automatiquement en production** ou **sous reserve d'action de validation** (création de tag spécifique pour le déploiement)

⚠️ Ce niveau nécessite :

* Une forte confiance dans les tests
* Une bonne gestion des risques

---

### 🔄 Différence Delivery vs Deployment

| Concept    | Description             |
| ---------- | ----------------------- |
| Delivery   | Prêt à être déployé     |
| Deployment | Déployé automatiquement |

---

### 🏷️ Versioning et déclenchement

Dans ce projet, le choix est le suivant :

* **CI** → déclenchée sur Pull Request
* **CD** → déclenchée via **tags semVer Git**
* **Continuous Deployment** → déclenchée via **tags spécifique Git**

Exemple :

```bash
git tag v1.0.0
git push origin v1.0.0
```

👉 Cela permet :

* Un contrôle total des releases
* Une séparation claire entre développement et production

---

## ⚙️ Pipeline global

### 🔁 CI (Pull Requests)

1. Installation des dépendances
2. Lint (qualité du code)
3. Tests unitaires
4. Build

---

### 🚀 CD Globale (Release via tag)

1. Build production
2. Tests finaux
3. Packaging (Docker, artefacts)

4. Déploiement

---

## 🏷️ Gestion des versions

Utilisation du **Semantic Versioning** :

```text
vMAJOR.MINOR.PATCH
```

---

## 🛠️ Outils recommandés

* GitHub Actions
* Docker
* Registry (GHCR / Docker Hub)

---

## 📌 Bonnes pratiques

* ✅ Pipeline rapide (fail fast)
* ✅ Tests automatisés
* ✅ Logs exploitables
* ✅ Rollback possible
* ✅ Environnements isolés

---

## 🔐 Sécurité

* Utiliser des secrets sécurisés
* Limiter les permissions
* Scanner les dépendances

---

## 📊 Environnements (minimum)

* dev
* staging/pré-production
* production

---

## ✅ Résumé

| Action | Déclencheur  | Objectif        |
| ------ | ------------ | --------------- |
| CI     | Pull Request | Qualité du code |
| CD     | Tag Git      | Déploiement     |

---

## 🚀 Exemple GitHub Actions

Sur GitHub, chaque workflow est un fichier `.github/workflows/*.yml`. CI sur
chaque pull request, CD sur tag de version :

```yaml
name: CI

on:
  pull_request:          # CI sur pull request
  push:
    branches: [main]
    tags: ["v*.*.*"]     # CD sur tag

permissions:
  contents: read         # droits du GITHUB_TOKEN au plus juste

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: echo "Run tests"

  build:
    needs: test
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Build release ${{ github.ref_name }}"
```

En production, épingler les actions tierces par **SHA** de commit plutôt que par
tag (`uses: actions/checkout@<sha> # v4`) : un tag peut être déplacé, pas un SHA.

---

## ♻️ Mutualiser la pipeline : workflows réutilisables

Plutôt que de copier ce fichier dans chaque dépôt, les jobs sont écrits **une
fois** dans un dépôt dédié, sous forme de workflow réutilisable
(`on: workflow_call` avec des `inputs` typés). Chaque projet l'appelle à une
**version épinglée** :

```yaml
# .github/workflows/ci.yml d'un projet consommateur
on:
  push:
    branches: ["**"]
    tags: ["V-*"]

permissions:
  contents: read
  packages: write

jobs:
  pipeline:
    uses: my-org/pipeline-python-factory/.github/workflows/python-docker.example.yml@1.0.0
    with:
      python_version: "3.13"
      pytest_cov_source: app
    secrets: inherit
```

Une nouvelle version de la pipeline ne change rien chez le consommateur tant
que le numéro n'est pas modifié, par une pull request testée comme le reste.
Exemple complet (brique commune, job `rules`, scan, publication, déploiement
illustratif) : [pipeline-python-centralisee-portfolio](https://github.com/JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio).

---

## 🦊 Équivalent GitLab CI

Sur GitLab, la pipeline est décrite dans un fichier `.gitlab-ci.yml` à la racine du dépôt,
et une **merge request** joue le rôle de la pull request.

```yaml
stages:
  - test
  - build

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"   # CI sur merge request
    - if: $CI_COMMIT_TAG                                  # CD sur tag
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

test:
  stage: test
  image: python:3.12-slim
  script:
    - echo "Run tests"

build:
  stage: build
  image: python:3.12-slim
  script:
    - echo "Build release $CI_COMMIT_TAG"
  rules:
    - if: $CI_COMMIT_TAG
```

### 🔄 Correspondance GitHub Actions / GitLab CI

| GitHub Actions | GitLab CI |
|----------------|-----------|
| `.github/workflows/*.yml` | `.gitlab-ci.yml` |
| `on: pull_request` | `rules: - if: $CI_PIPELINE_SOURCE == "merge_request_event"` |
| `on: push: tags` | `rules: - if: $CI_COMMIT_TAG` |
| `jobs.<id>.needs` | `needs` |
| `runs-on` | `tags` (choix du runner) + `image` |
| `actions/checkout` | automatique (clone du dépôt) |
| `uses:` (workflow réutilisable, `workflow_call`) | `include` / CI/CD components |
| `secrets.MON_SECRET` | variables CI/CD masquées et protégées |
| `environment` (+ *Required reviewers*) | `environment` (+ `when: manual`) |
| `concurrency` | `resource_group` |
| `actions/upload-artifact` | `artifacts` |
| `actions/cache` | `cache` |
