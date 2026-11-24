# 🧱 Découper une fonctionnalité en tâches

## 🎯 Objectif

Transformer une fonctionnalité en **tâches simples, réalisables et vérifiables**,
pour mieux estimer, paralléliser le travail et livrer régulièrement.

---

## 📌 Pourquoi découper ?

- **Estimer** : une petite tâche s'estime bien, une grosse presque jamais.
- **Avancer visiblement** : chaque tâche terminée est un progrès mesurable.
- **Relire facilement** : une petite pull request est relue vite et bien.
- **Réduire le risque** : un problème est détecté tôt, sur un petit périmètre.
- **Lever l'ambiguïté** : découper oblige à préciser ce qui est attendu.

---

## 🪜 Les niveaux

| Niveau | Rôle | Durée type | Dans GitHub |
|--------|------|------------|-------------|
| **Epic** | grand objectif métier, regroupe plusieurs user stories | plusieurs semaines | issue parente avec sous-issues, ou milestone |
| **User story** | besoin d'un utilisateur, apporte une valeur visible | quelques jours | issue (ou sous-issue de l'epic) |
| **Tâche** | travail technique concret pour réaliser la story | moins d'une journée | sous-issue, ou case à cocher dans l'issue |

Formulation d'une user story :

> En tant que **[type d'utilisateur]**, je veux **[action]** afin de **[bénéfice]**.

---

## 🧩 Exemple : l'authentification

**Epic** : Gestion des comptes utilisateurs

**User story** : *En tant qu'utilisateur, je veux me connecter avec mon e-mail et mon mot de passe
afin d'accéder à mon espace personnel.*

**Critères d'acceptation**
- un identifiant valide renvoie un jeton d'accès ;
- un identifiant invalide renvoie `401` avec le code `INVALID_CREDENTIALS`, sans préciser le champ en erreur ;
- après 5 échecs en 15 minutes, le compte est temporairement bloqué.

**Tâches**
- [ ] Créer le modèle utilisateur et la migration (mot de passe haché avec Argon2 ou bcrypt)
- [ ] Créer l'endpoint `POST /auth/login`
- [ ] Générer et signer le jeton JWT (durée de vie courte)
- [ ] Ajouter le middleware de vérification du jeton sur les routes protégées
- [ ] Limiter les tentatives de connexion
- [ ] Écrire les tests : nominal, mauvais mot de passe, compte bloqué, jeton expiré
- [ ] Documenter l'endpoint dans le contrat OpenAPI

Chaque tâche peut faire l'objet d'une pull request distincte (`Closes #42` dans la description ferme l'issue au merge).

---

## ✂️ Comment découper

### Découpage vertical plutôt qu'horizontal

| ❌ Horizontal (par couche) | ✅ Vertical (par fonctionnalité) |
|---------------------------|---------------------------------|
| 1. toute la base de données | 1. connexion simple, de bout en bout |
| 2. toute l'API | 2. blocage après échecs répétés |
| 3. tout le front-end | 3. mot de passe oublié |
| → rien d'utilisable avant la fin | → une valeur livrée à chaque étape |

### Techniques utiles

- **Par cas d'usage** : le cas nominal d'abord, les cas d'erreur ensuite.
- **Par règle métier** : une règle = une tâche.
- **Par type de données** : d'abord un format d'import, puis les autres.
- **Simple puis complet** : une version minimale, puis les améliorations.
- **Isoler l'inconnu** : une tâche de **spike** à durée limitée pour explorer une technologie avant de s'engager.

---

## ✅ Une bonne tâche est…

| Critère | Signification |
|---------|---------------|
| **Petite** | réalisable en moins d'une journée |
| **Claire** | quelqu'un d'autre comprend ce qui est attendu sans explication orale |
| **Indépendante** | peut être faite sans attendre une autre tâche, autant que possible |
| **Testable** | on sait vérifier qu'elle est terminée |
| **Utile** | contribue directement à la user story |

### Définition de « terminé »

Une tâche est terminée quand :

- [ ] le code est relu et fusionné ;
- [ ] les tests sont écrits et la pipeline est verte ;
- [ ] la documentation est à jour ;
- [ ] les critères d'acceptation sont vérifiés.

---

## ❌ À éviter

- **Tâches trop larges** (« faire l'authentification ») : sujettes à interprétation, impossibles à estimer.
- **Tâches trop fines** (« renommer une variable ») : coût de suivi supérieur au travail.
- **Dépendances en chaîne** : une tâche bloquée en bloque cinq autres.
- **Tâches sans critère de fin** : elles ne se terminent jamais vraiment.
- **Oublier les tâches invisibles** : tests, documentation, déploiement, monitoring.

---

## 🔗 Voir aussi

- [Rédiger une issue](issue-guidelines.md)
- [Signaler un bug](bug-reporting.md)
- [Templates d'issues](../../templates/issues/)
