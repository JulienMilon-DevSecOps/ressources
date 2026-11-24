# 🔐 Bonnes pratiques de sécurité

## 🎯 Objectif

Intégrer la sécurité à chaque étape du cycle de vie d'un projet (**DevSecOps**),
plutôt que de la vérifier une fois, juste avant la mise en production.

Principe directeur : **détecter le plus tôt possible**, quand une correction coûte le moins cher.

---

## 🗺️ Vue d'ensemble

| Étape | Risque | Contrôle |
|-------|--------|----------|
| Code | secret commité, faille dans le code | détection de secrets, SAST, revue |
| Dépendances | bibliothèque vulnérable | analyse de dépendances (SCA), mises à jour automatiques |
| Image | paquet système vulnérable, exécution en root | scan d'image, image minimale, utilisateur non root |
| Pipeline | jeton trop puissant, variable exposée | variables protégées et masquées, moindre privilège |
| Déploiement | configuration faible, accès trop large | durcissement, réseau restreint, secrets externalisés |
| Exploitation | attaque non détectée | journalisation, alertes, mises à jour |

---

## 🔑 Secrets

Un secret (mot de passe, jeton, clé privée, chaîne de connexion) **ne doit jamais
être dans le dépôt Git**, même dans une branche ou un commit supprimé : l'historique le conserve.

- Stocker les secrets dans les **secrets GitHub Actions**, de préférence au niveau d'un
  **environment** protégé (lisibles uniquement par ses jobs), ou dans un coffre (Vault, gestionnaire de secrets du cloud).
- Lire les secrets depuis l'environnement dans le code, jamais en dur.
- Fournir un `.env.example` sans valeurs, et ajouter `.env` au `.gitignore`.
- **Détecter automatiquement** les secrets en CI sur tout l'historique (gitleaks, Nosey Parker).
- En cas de fuite : **révoquer d'abord**, nettoyer l'historique ensuite. Un secret
  publié doit être considéré comme compromis.

```yaml
secret-detection:
  runs-on: ubuntu-latest
  permissions:
    contents: read
  steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0          # analyser tout l'historique
        persist-credentials: false
    - run: |
        docker run --rm -v "$GITHUB_WORKSPACE:/repo" ghcr.io/gitleaks/gitleaks:v8.30.1 \
          git --redact --verbose /repo
```

Activer aussi **Secret scanning** et **Push protection** (*Settings* → *Code
security*) : GitHub bloque le push d'un secret reconnu avant qu'il n'atteigne le dépôt.

---

## 🧑‍💻 Code

- **SAST** (analyse statique) en CI : Ruff avec les règles `S` (Bandit) pour Python,
  Semgrep, ou CodeQL (*Code scanning* de GitHub).
- **Valider toutes les entrées** : types, tailles, formats (Pydantic en Python).
- **Requêtes paramétrées** ou ORM : jamais de SQL construit par concaténation.
- **Messages d'erreur génériques** côté client, détails dans les logs uniquement.
- **Revue de code obligatoire** via pull request sur les branches protégées
  (règles de protection ou *rulesets*, relecteurs désignés par `CODEOWNERS`).

---

## 📦 Dépendances

La majorité du code exécuté vient des dépendances : c'est une surface d'attaque majeure.

- **Épingler les versions** (fichier de verrouillage : `uv.lock`, `poetry.lock`, `package-lock.json`).
- **Analyser les vulnérabilités connues** en CI : `pip-audit`, OWASP Dependency-Check, `npm audit`.
- **Automatiser les mises à jour** avec Renovate ou Dependabot, pour corriger au fil de l'eau.
- Limiter le nombre de dépendances et vérifier leur maintenance avant de les ajouter.

---

## 🐳 Images de conteneur

- Partir d'une **image minimale et officielle** (`-slim`, `distroless`, `alpine`), à version épinglée.
- **Exécuter en utilisateur non root** (`USER` dans le Dockerfile).
- **Scanner l'image** avant publication (Trivy, Grype) et bloquer sur les vulnérabilités critiques corrigeables.
- Ne jamais copier de secret dans une image : il reste lisible dans les couches.
- **Signer les images** publiées (cosign) pour garantir leur provenance.

```yaml
trivy-scan:
  stage: scan
  image:
    name: aquasec/trivy:0.74.0
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed "$IMAGE"
```

Voir aussi le guide [écrire un Dockerfile qui passe l'analyse de sécurité](dockerfile-security.md)
et les [bonnes pratiques Docker](../devops/docker-best-practices.md).

---

## ⚙️ Pipeline CI/CD

La pipeline a souvent accès à la production : elle doit être protégée comme telle.

- **Branches et tags protégés** pour tout ce qui déploie.
- **Variables protégées** : disponibles uniquement sur les références protégées.
- **Jetons à portée minimale** et à durée limitée (deploy tokens, project access tokens).
- **Épingler les images et les `include`** de pipeline à une version (tag), pas à `main` ou `latest`.
- **Environnements protégés** avec validation manuelle pour la production.
- Décider explicitement ce qui est **bloquant** : un contrôle en `allow_failure: true`
  doit être une transition temporaire, pas un état permanent.

---

## 🚀 Déploiement et infrastructure

- **Moindre privilège** partout : comptes de service, utilisateurs de déploiement, rôles Kubernetes.
- **Chiffrement en transit** (TLS) et au repos pour les données sensibles.
- **Durcir les serveurs** : SSH par clé uniquement (voir [clés SSH](../devops/generer-cles-ssh.md)),
  pare-feu, mises à jour de sécurité automatiques.
- **Kubernetes** : `securityContext` (`runAsNonRoot`, `readOnlyRootFilesystem`),
  `NetworkPolicy`, secrets externalisés (External Secrets, Sealed Secrets).
- **Infrastructure as Code** : versionnée, revue, et analysée (Trivy config, Checkov).

---

## 📊 Exploitation

- **Journaliser** les authentifications, les erreurs et les actions sensibles, sans données personnelles ni secrets.
- **Alerter** sur les comportements anormaux (pics d'erreurs 401/403, connexions inhabituelles).
- **Plan de réponse** : savoir qui prévenir, comment révoquer un accès, comment restaurer.

Voir [monitoring et logging](../devops/monitoring-logging.md).

---

## ✅ Checklist minimale d'un projet

- [ ] Aucun secret dans le dépôt, détection de secrets en CI
- [ ] SAST et analyse des dépendances en CI
- [ ] Mises à jour automatiques des dépendances
- [ ] Image minimale, non root, scannée avant publication
- [ ] Branches, tags et variables de déploiement protégés
- [ ] Versions des images épinglées, actions tierces et workflows réutilisables épinglés par SHA ou tag immuable
- [ ] `permissions:` du `GITHUB_TOKEN` déclarées au plus juste dans chaque workflow
- [ ] TLS sur tous les échanges
- [ ] Logs sans données sensibles, alertes en place

---

## 📚 Références

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [Sécurité dans GitHub](https://docs.github.com/fr/code-security)
- [Durcissement de GitHub Actions](https://docs.github.com/fr/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- Mise en pratique : [pipeline-python-centralisee-portfolio](https://github.com/JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio)
