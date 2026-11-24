# 🧪 Stratégie de tests

## 🎯 Objectif

Définir **quoi tester, à quel niveau et à quel moment**, pour détecter les
régressions tôt sans ralentir le développement.

Un bon jeu de tests permet de modifier le code et de déployer **en confiance**.

---

## 🔺 La pyramide des tests

```text
            /\
           /E2E\           peu nombreux, lents, coûteux
          /------\
         /Intégr. \        nombre modéré
        /----------\
       / Unitaires  \      nombreux, rapides, isolés
      /--------------\
```

| Niveau | Ce qu'on vérifie | Vitesse | Exemple |
|--------|------------------|---------|---------|
| **Unitaire** | une fonction ou une classe, isolée | millisecondes | calcul d'un prix, validation d'un schéma |
| **Intégration** | plusieurs composants réels ensemble | secondes | endpoint d'API + base de données |
| **Bout en bout (E2E)** | un parcours utilisateur complet | minutes | connexion puis création d'une commande |

Règle : **plus un test est haut dans la pyramide, moins il y en a**. Un bug
détectable par un test unitaire ne doit pas dépendre d'un test E2E.

---

## 🧱 Tests unitaires

- Testent la **logique métier**, sans réseau, sans base, sans système de fichiers.
- Les dépendances externes sont remplacées par des doublures (mock, stub, fake).
- Un test = un comportement, avec un nom qui le décrit.

Structure **Arrange / Act / Assert** :

```python
def test_remise_appliquee_au_dela_de_100_euros() -> None:
    # Arrange
    panier = Panier(articles=[Article(prix=120)])

    # Act
    total = panier.total()

    # Assert
    assert total == 108  # 10 % de remise
```

---

## 🔗 Tests d'intégration

- Vérifient que les composants **fonctionnent ensemble** : API, base de données, cache, file de messages.
- Utilisent de **vrais services** jetables : conteneurs Docker, conteneurs de service GitHub Actions, Testcontainers.
- Chaque test part d'un **état connu** (base vidée ou transaction annulée).

Exemple avec une API FastAPI :

```python
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_health() -> None:
    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

---

## 🌐 Tests de bout en bout

- Réservés aux **parcours critiques** (connexion, paiement, fonctionnalité cœur).
- Exécutés sur un environnement proche de la production (recette, staging).
- Outils : Playwright, Cypress, ou des appels HTTP sur l'API déployée.

---

## 🧰 Autres types de tests

| Type | Objectif | Quand |
|------|----------|-------|
| **Contrat** | l'API respecte son schéma OpenAPI | à chaque pull request |
| **Performance / charge** | temps de réponse et tenue en charge (k6, Locust) | avant une release importante |
| **Sécurité** | SAST, dépendances, scan d'image (voir [sécurité](../security/security-best-practices.md)) | à chaque pipeline |
| **Smoke test** | l'application démarre et répond après déploiement | après chaque déploiement |

---

## 📏 Couverture de code

- La couverture indique **ce qui n'est pas testé**, pas la qualité des tests.
- Viser un seuil raisonnable (souvent **70 à 80 %**) et surtout **ne pas le faire baisser**.
- Couvrir en priorité la logique métier et les cas d'erreur, pas les getters triviaux.

```bash
pytest --cov=app --cov-report=term-missing --cov-fail-under=80
```

---

## 🔄 Les tests dans la CI/CD

| Moment | Tests exécutés | Bloquant |
|--------|----------------|----------|
| Chaque commit / pull request | unitaires, intégration, lint, sécurité | ✅ |
| Merge sur `develop` | + tests de contrat | ✅ |
| Déploiement en recette | smoke tests, E2E critiques | ✅ |
| Avant release majeure | performance | selon contexte |

Exemple de job GitHub Actions avec une base PostgreSQL de test et un résumé
de couverture visible sur la page du run :

```yaml
jobs:
  pytest:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    services:
      db:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 5s --health-retries 10
    env:
      DATABASE_URL: postgresql://test:test@localhost:5432/test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[test]" pytest-cov
      - run: |
          pytest --cov=app --cov-report=term-missing --junitxml=report.xml
          { echo "### Couverture"; coverage report --format=markdown; } >> "$GITHUB_STEP_SUMMARY"
      - if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pytest-report
          path: report.xml
```

- Le **résumé** (`$GITHUB_STEP_SUMMARY`) affiche le tableau de couverture sur la page du run.
- Le rapport **JUnit** est conservé en artefact ; une action de publication de rapports
  (par exemple `dorny/test-reporter`) peut l'afficher en annotations dans la pull request.
- Sur GitLab CI, l'équivalent passe par `services:` et `artifacts: reports: junit / coverage_report`.

---

## ✅ Bonnes pratiques

- [ ] Tests **rapides** : la suite unitaire tourne en moins d'une minute
- [ ] Tests **indépendants** : exécutables dans n'importe quel ordre
- [ ] Tests **déterministes** : pas de dépendance à l'heure, au réseau ou au hasard non maîtrisé
- [ ] Un bug corrigé = un test qui le reproduit
- [ ] Les tests sont relus en pull request comme le code
- [ ] Un test instable (flaky) est corrigé ou supprimé, jamais ignoré

## ❌ À éviter

- Tester l'implémentation plutôt que le comportement (tests cassés à chaque refactoring)
- Des tests E2E pour tout : suite lente et fragile
- Partager un état entre tests
- Courir après 100 % de couverture
