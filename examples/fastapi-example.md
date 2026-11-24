# ⚡ Exemple : une API FastAPI qui applique les guides du dépôt

## 🎯 Objectif

Montrer, sur un projet concret, comment s'articulent les ressources de ce dépôt :
structure du code, gestion des erreurs, tests, conteneurisation, CI/CD et sécurité.

Une version exécutable de cet exemple tourne dans la CI de
[pipeline-python-centralisee-portfolio](https://github.com/JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio)
(`examples/sample-app`).

---

## 📂 Structure du projet

```text
mon-api/
├── .github/
│   ├── ISSUE_TEMPLATE/         ← copiés depuis templates/issues/
│   ├── workflows/
│   │   └── ci.yml              ← appelle la pipeline centralisée
│   ├── CODEOWNERS
│   └── pull_request_template.md
├── app/
│   ├── __init__.py
│   ├── main.py                 ← création de l'application, routes techniques
│   ├── config.py               ← configuration lue depuis l'environnement
│   ├── errors.py               ← format d'erreur unique
│   └── routers/
│       └── users.py            ← une ressource = un routeur
├── tests/
│   └── test_main.py
├── .env.example                ← variables attendues, sans valeurs
├── .gitignore
├── Dockerfile
├── pyproject.toml
└── README.md                   ← basé sur templates/readme/api-readme-template.md
```

---

## ⚙️ Configuration par l'environnement

Aucun secret dans le code ([sécurité](../docs/security/security-best-practices.md)) :
la configuration est lue depuis les variables d'environnement et validée au démarrage.

```python
# app/config.py
"""Configuration de l'application."""

from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    """Paramètres lus depuis l'environnement (ou un fichier .env en local)."""

    database_url: str
    log_level: str = "INFO"

    model_config = {"env_file": ".env"}


settings = Settings()
```

```bash
# .env.example
DATABASE_URL=postgresql://user:password@localhost:5432/mon_api
LOG_LEVEL=INFO
```

---

## ⚠️ Un format d'erreur unique

Conforme au [catalogue d'erreurs](../docs/api/error-catalog.md) : toutes les
erreurs ont la même forme, avec un code stable que le client peut exploiter.

```python
# app/errors.py
"""Erreurs métier et leur conversion en réponse HTTP."""

from fastapi import Request
from fastapi.responses import JSONResponse


class ApiError(Exception):
    """Erreur métier exposée au client."""

    def __init__(self, status_code: int, code: str, message: str) -> None:
        self.status_code = status_code
        self.code = code
        self.message = message


async def api_error_handler(_: Request, exc: ApiError) -> JSONResponse:
    """Convertit une ApiError en réponse JSON normalisée."""
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.code, "message": exc.message}},
    )
```

---

## 🚪 Application et routes

```python
# app/main.py
"""Point d'entrée de l'API."""

from importlib.metadata import version

from fastapi import FastAPI

from app.errors import ApiError, api_error_handler
from app.routers import users

app = FastAPI(
    title="mon-api",
    version=version("mon-api"),
    description="API d'exemple appliquant les guides du dépôt ressources.",
)
app.add_exception_handler(ApiError, api_error_handler)
app.include_router(users.router, prefix="/api/v1")


@app.get("/health", tags=["technique"])
def health() -> dict[str, str]:
    """Indique que l'API répond (sonde de vie, HEALTHCHECK Docker)."""
    return {"status": "ok"}
```

```python
# app/routers/users.py
"""Ressource utilisateurs."""

from fastapi import APIRouter, status
from pydantic import BaseModel

from app.errors import ApiError

router = APIRouter(prefix="/users", tags=["users"])


class User(BaseModel):
    """Représentation publique d'un utilisateur."""

    id: int
    name: str


_USERS = {1: User(id=1, name="Ada")}


@router.get("/{user_id}", responses={404: {"description": "Utilisateur introuvable"}})
def get_user(user_id: int) -> User:
    """Retourne un utilisateur par son identifiant."""
    user = _USERS.get(user_id)
    if user is None:
        raise ApiError(status.HTTP_404_NOT_FOUND, "USER_NOT_FOUND", "Utilisateur introuvable")
    return user
```

Ce que ça applique de la [checklist API](../docs/api/api-design-checklist.md) :
ressource au pluriel, version dans l'URL, codes HTTP corrects, modèles typés
qui génèrent la documentation [OpenAPI](../docs/api/openapi-guide.md) (`/docs`).

---

## 🧪 Tests

Selon la [stratégie de tests](../docs/testing/testing-strategy.md) : un test par
comportement, y compris les cas d'erreur.

```python
# tests/test_main.py
"""Tests de l'API."""

from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_health() -> None:
    """La sonde de vie répond 200."""
    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}


def test_get_user_inconnu_renvoie_404_normalise() -> None:
    """Un utilisateur absent renvoie le code d'erreur du catalogue."""
    response = client.get("/api/v1/users/999")

    assert response.status_code == 404
    assert response.json()["error"]["code"] == "USER_NOT_FOUND"
```

---

## 📦 `pyproject.toml`

```toml
[project]
name = "mon-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["fastapi>=0.115", "uvicorn>=0.30", "pydantic-settings>=2.4"]

[project.optional-dependencies]
test = ["pytest>=8", "pytest-cov>=5", "httpx>=0.27"]

[tool.setuptools]
packages = ["app", "app.routers"]

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "S"]   # S : règles de sécurité (Bandit)

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]                        # assert est normal dans un test
```

---

## 🐳 Dockerfile

Selon les [bonnes pratiques Docker](../docs/devops/docker-best-practices.md) :
multi-stage, image slim, utilisateur non root, sonde de santé.

```dockerfile
FROM python:3.12-slim AS build
WORKDIR /build
COPY pyproject.toml ./
COPY app ./app
RUN pip wheel --no-cache-dir --wheel-dir /wheels .

FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/* && rm -rf /wheels \
    && useradd --create-home --uid 10001 appuser
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 🔄 CI/CD

Plutôt que d'écrire toute la pipeline, le projet **appelle un workflow
réutilisable centralisé à une version épinglée** et ne déclare que ce qui lui
est propre :

```yaml
# .github/workflows/ci.yml
name: ci

on:
  push:
    branches: ["**"]
    tags: ["V-*"]          # V-1.2.0 : release de l'image

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.ref_type == 'branch' }}

permissions:
  contents: read
  packages: write          # publication de l'image sur ghcr.io

jobs:
  pipeline:
    uses: JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio/.github/workflows/python-docker.example.yml@0.1.0
    with:
      python_version: "3.12"
      pytest_cov_source: app
      scan_severity: CRITICAL
    secrets: inherit
```

Un input inconnu est refusé au démarrage du run ; la liste complète est dans le
[contrat des inputs](https://github.com/JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio/blob/main/docs/inputs.md).

Ce qui est exécuté :

| Étape | Contrôles |
|-------|-----------|
| `check` | détection de secrets (gitleaks), lint et formatage (Ruff) |
| `test` | pytest + couverture, vulnérabilités des dépendances |
| `build` | image Docker (sans push sur les branches) |
| `scan` | vulnérabilités de l'image (Trivy) |
| `publish` | tag `latest` posé sur l'image scannée, sur tag `V-x.y.z` |

Le détail des étapes et des déclencheurs est dans le [guide CI/CD](../docs/development/ci-cd-guide.md)
et le [Git flow](../docs/development/git-flow.md).

---

## ✅ Récapitulatif

| Guide | Appliqué par |
|-------|--------------|
| [Checklist API](../docs/api/api-design-checklist.md) | routeurs versionnés, codes HTTP, modèles typés |
| [Catalogue d'erreurs](../docs/api/error-catalog.md) | `ApiError` + handler unique |
| [Stratégie de tests](../docs/testing/testing-strategy.md) | tests du nominal et des erreurs, couverture en CI |
| [Docker](../docs/devops/docker-best-practices.md) | multi-stage, non root, healthcheck |
| [Sécurité](../docs/security/security-best-practices.md) | configuration par l'environnement, scans en CI |
| [Templates](../templates/) | issues, pull requests, README |
