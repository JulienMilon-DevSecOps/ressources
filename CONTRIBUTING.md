# 🤝 Guide de contribution

Merci de vouloir contribuer à ce projet ! 🎉

Ce document décrit les règles à suivre pour garantir la qualité et la cohérence du dépôt.

---

## 📌 Objectif du projet

Ce projet personnel rassemble des ressources autour de :

* la conception d'API ;
* la CI/CD et le DevOps ;
* la sécurité ;
* les bonnes pratiques de développement et de gestion de projet.

Il est public pour partager ces ressources et recueillir des propositions
d'amélioration. Merci de respecter le travail réalisé.

Les branches `main` et `develop` sont protégées : toute modification passe par une pull request,
relue par les propriétaires définis dans [`.github/CODEOWNERS`](.github/CODEOWNERS).

---

## 🚀 Comment contribuer

### 1. Forker le dépôt

Créez un fork du projet depuis GitHub (bouton **Fork**), puis clonez-le :

```bash
git clone https://github.com/<votre-utilisateur>/ressources.git
cd ressources
```

---

### 2. Créer une branche

Ne travaillez **jamais directement sur `main`** ou `develop`.

```bash
git checkout develop
git checkout -b feature/nom-de-la-feature
```

Vous pouvez au préalable créer une issue (avec l'un des templates proposés),
puis créer la branche depuis l'issue (section **Development** → **Create a branch**),
en choisissant `develop` comme branche de départ.

---

### 3. Développer

* Respecter les conventions du projet (noms de fichiers en minuscules, séparés par des tirets)
* Ajouter de la documentation si nécessaire
* Ajouter des exemples concrets si pertinent

---

### 4. Commit

Utiliser la convention [Conventional Commits](https://www.conventionalcommits.org/fr/) :

```text
feat: ajout d'un template API
fix: correction du README
docs: ajout de documentation CI/CD
chore: mise à jour des dépendances
```

---

### 5. Push et pull request

```bash
git push origin feature/nom-de-la-feature
```

Puis :

* ouvrir une pull request vers `develop` ;
* remplir le template pré-rempli dans la description ;
* décrire clairement le contexte et les changements.

---

## 🔀 Processus de validation

Toute contribution passe par :

* ✅ une pull request ;
* ✅ une revue ;
* ✅ une validation par un mainteneur.

Aucun merge direct sur `main` n'est autorisé.

---

## 📋 Règles de pull request

Une pull request doit :

* avoir un titre explicite ;
* décrire le contexte et les changements ;
* rester de taille raisonnable et ne traiter qu'un sujet.

---

## 🧪 Qualité

Avant de soumettre une pull request :

* vérifier que les exemples de code ou de configuration fonctionnent ;
* vérifier le rendu Markdown (titres, blocs de code fermés, liens valides) ;
* relire l'orthographe.

---

## 📚 Documentation

Tout nouveau contenu doit :

* être placé dans le bon dossier de `docs/` ;
* être référencé dans le [README](README.md) ;
* être compréhensible par quelqu'un d'extérieur au projet.

---

## 🧩 Types de contributions

* 🐛 Correction d'erreurs
* ✨ Nouveau guide ou template
* 📚 Amélioration de la documentation
* 🧹 Restructuration

---

## 🧑‍⚖️ Revue

Les relecteurs peuvent demander des modifications, poser des questions ou
suggérer des améliorations. Une pull request peut être refusée si elle ne
respecte pas ces règles.

---

## ❓ Questions

Ouvrez une issue ou posez-la directement dans la pull request.

---

## 🙌 Merci

Merci pour votre contribution ! 🚀
