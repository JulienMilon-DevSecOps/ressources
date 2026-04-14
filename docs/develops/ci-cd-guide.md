# 🔄 CI/CD Guide

## 📌 Introduction

Ce document propose un **exemple de mise en place d’un pipeline CI/CD** applicable à la majorité des projets modernes.
Il a pour objectif de servir de base et peut être adapté selon le contexte technique (stack, infrastructure, organisation). L'example proposé est évidement basée sur GitHub Actions, GitLab à un fonctionnement différent (je propose un autre exemple dans cet autre projet GitLab que je maintien [lien](url))

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

```bash id="x0rsxp"
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

```id="u8sdtq"
vMAJOR.MINOR.PATCH
```

---

## 🛠️ Outils recommandés

* GitHub Actions
* Docker
* Registry (Docker Hub / GHCR)

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

```yaml id="fp7exk"
name: CI

on:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: echo "Run tests"
```
