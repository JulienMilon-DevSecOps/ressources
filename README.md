
# 🚀 Ressources DevSecOps

Bonnes pratiques, guides et templates réutilisables pour livrer des projets
de qualité : API, CI/CD, DevOps, sécurité et gestion de projet.

Ces ressources sont issues de mon expérience de terrain (pipelines CI/CD
centralisées sur GitHub Actions et GitLab CI, chaîne de sécurité en CI,
conteneurisation, Kubernetes on-premise). Elles sont pensées pour être
**copiées et adaptées** dans vos projets.

---

## 📂 Structure

```text
.
├── .github/
│   ├── ISSUE_TEMPLATE/            ← templates d'issues de ce dépôt
│   ├── CODEOWNERS                 ← relecteurs par défaut
│   └── pull_request_template.md   ← template de pull request de ce dépôt
├── docs/
│   ├── api/                       ← conception et documentation d'API
│   ├── development/               ← Git flow, CI/CD
│   ├── devops/                    ← Docker, déploiement, observabilité, SSH
│   ├── project-management/        ← issues, bugs, découpage des tâches
│   ├── security/                  ← bonnes pratiques de sécurité
│   └── testing/                   ← stratégie de tests
├── examples/                      ← exemples concrets d'application des guides
├── templates/
│   ├── issues/                    ← templates d'issues (bug, feature, API, frontend)
│   └── readme/                    ← templates de README (projet, API, frontend)
├── CONTRIBUTING.md
└── LICENSE
```

---

## 📚 Documentation

### 💻 Développement
- [Git flow](docs/development/git-flow.md) : stratégie de branches adaptée à la CI/CD
- [Guide CI/CD](docs/development/ci-cd-guide.md) : concepts, déclencheurs, versioning

### ⚙️ DevOps
- [Bonnes pratiques Docker](docs/devops/docker-best-practices.md)
- [Stratégies de déploiement](docs/devops/deployment-strategies.md)
- [Monitoring et logging](docs/devops/monitoring-logging.md)
- [Générer et utiliser des clés SSH](docs/devops/generer-cles-ssh.md)

### 🔐 Sécurité
- [Bonnes pratiques de sécurité](docs/security/security-best-practices.md)
- [Écrire un Dockerfile qui passe l'analyse de sécurité en CI](docs/security/dockerfile-security.md)

### 🧪 Tests
- [Stratégie de tests](docs/testing/testing-strategy.md)

### 📡 API
- [Checklist de conception d'API](docs/api/api-design-checklist.md)
- [Guide OpenAPI](docs/api/openapi-guide.md)
- [Catalogue d'erreurs](docs/api/error-catalog.md)

### 📋 Gestion de projet
- [Rédiger une issue](docs/project-management/issue-guidelines.md)
- [Signaler un bug](docs/project-management/bug-reporting.md)
- [Découper une fonctionnalité en tâches](docs/project-management/task-breakdown.md)

### 🧩 Exemples
- [Projet FastAPI appliquant ces guides](examples/fastapi-example.md)

---

## 🧩 Templates

| Type | Emplacement | Usage |
|------|-------------|-------|
| Issues | [`templates/issues/`](templates/issues/) | à copier dans `.github/ISSUE_TEMPLATE/` de votre projet |
| README | [`templates/readme/`](templates/readme/) | point de départ du README d'un projet, d'une API ou d'un frontend |

Sur GitHub, un fichier Markdown placé dans `.github/ISSUE_TEMPLATE/` est proposé
à la création d'une issue, à condition de commencer par un en-tête YAML
(`name`, `about`) ; `.github/pull_request_template.md` pré-remplit la
description de chaque pull request.

```bash
mkdir -p .github/ISSUE_TEMPLATE
{
  printf -- '---\nname: "🐛 Bug"\nabout: Signaler un dysfonctionnement\nlabels: bug\n---\n\n'
  cat templates/issues/bug-template.md
} > .github/ISSUE_TEMPLATE/bug.md
```

---

## 🔗 Projets liés

- [pipeline-python-centralisee-portfolio](https://github.com/JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio) :
  pipeline GitHub Actions centralisée pour projets Python (packages et API conteneurisées),
  en workflows réutilisables, qui met en pratique les guides CI/CD et sécurité de ce dépôt.
- Une édition GitLab CI/CD de ces ressources et de la pipeline existe également.

---

## 👤 Auteur

**Julien Milon**, ingénieur DevSecOps freelance.

CI/CD GitHub Actions et GitLab CI, sécurité de la chaîne logicielle,
conteneurisation, Kubernetes, automatisation.
Pour une mission ou un échange, contactez-moi via mon
[profil GitHub](https://github.com/JulienMilon-DevSecOps).

---

## 🤝 Contribution

Les suggestions sont les bienvenues : voir [CONTRIBUTING.md](CONTRIBUTING.md).

## 📜 Licence

[MIT](LICENSE)
