# 🌿 Git Flow

## 🎯 Objectif

Définir une stratégie de gestion des branches claire et adaptée à un workflow avec CI/CD.

---

## 🌳 Branches principales

### 🔹 `main`

* Branche de production
* Code stable uniquement
* Protégée (PR obligatoire)

---

### 🔹 `develop`

* Branche d’intégration
* Regroupe les fonctionnalités validées
* Base pour les futures releases

---

## 🌱 Branches secondaires

### ✨ `feature/*`

* Nouvelles fonctionnalités
* Créées depuis `develop`

```bash
git checkout develop
git checkout -b feature/ma-feature
```

➡️ Merge vers `develop` via Pull Request

---

### 🐛 `hotfix/*`

* Corrections urgentes en production
* Créées depuis `main`

```bash
git checkout main
git checkout -b hotfix/bug-critique
```

➡️ Merge vers :

* `main`
* `develop`

---

## 🔀 Workflow global

1. Créer une branche `feature/*`
2. Développer
3. Ouvrir une Pull Request vers `develop`
4. CI exécutée automatiquement
5. Validation + merge

---

## 🧪 Intégration Continue (CI)

La CI est déclenchée :

* À chaque Pull Request

Objectifs :

* Tests automatiques
* Vérification qualité
* Build (project) # Test que le projet le code se build bien. Outil d'analyse de code.

---


## 🚀 Construction Continu (CD)

La construction de l'image applicative est déclenché uniquement via **tags Git** pour définir la version (type semVer ou latest).

### Exemple :

```bash
git checkout main
git tag v1.0.0
git push origin v1.0.0
```
👉 Cela déclenche :

* Toutes les étapes de la CI
* Build production
* Tests finaux

---

## 🚀 Déploiement Continu (CD)

Le déploiement est déclenché uniquement via **tags Git** pour définir la cible (environnement) et la version a déployer.

### Exemple :

```bash
git checkout main
git tag ppr-1.0.0
git push origin ppr-1.0.0
```

👉 Cela déclenche :

* Déploiement

---

## 🏷️ Stratégie de release

1. Merge `develop` → `main`
2. Créer un tag
3. Déployer automatiquement

---

## 📌 Bonnes pratiques

* ❌ Pas de commit direct sur `main`
* ❌ Pas de merge sans PR
* ✅ PR petites et ciblées
* ✅ Revue de code obligatoire
* ✅ CI verte avant merge

---

## 🧠 Résumé

| Type    | Base    | Merge vers     |
| ------- | ------- | -------------- |
| feature | develop | develop        |
| hotfix  | main    | main + develop |

---

## ✅ Conclusion

Ce workflow permet :

* Une qualité continue via CI
* Un contrôle total des releases via tags
* Une séparation claire entre développement et production
