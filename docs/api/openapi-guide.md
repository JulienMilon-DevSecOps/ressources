# 📘 Guide OpenAPI

## 🎯 Objectif

Décrire une API REST dans un **contrat OpenAPI** unique, lisible par les humains et
par les outils, qui sert de référence aux équipes back-end, front-end et aux clients.

---

## 🧠 Pourquoi un contrat OpenAPI

Un fichier OpenAPI (YAML ou JSON) permet de :

- générer une **documentation interactive** (Swagger UI, Redoc) ;
- générer des **clients** et des **squelettes de serveur** ;
- **valider** les requêtes et les réponses, et lancer des **tests de contrat** ;
- **détecter les changements cassants** entre deux versions ;
- créer des **mocks** pour que le front-end avance sans attendre le back-end.

---

## 🧭 Deux approches

| Approche | Principe | Avantages | Points d'attention |
|----------|----------|-----------|--------------------|
| **Code first** | le contrat est **généré** depuis le code (FastAPI, springdoc, NestJS) | toujours synchrone avec le code | la qualité dépend des annotations ; exporter et versionner le contrat |
| **Design first** | le contrat est **écrit d'abord**, le code suit | conception relue avant de coder, travail en parallèle | vérifier en CI que le code respecte le contrat |

Les deux sont valables ; l'essentiel est que **le contrat versionné et le code ne divergent pas**.

---

## 🧱 Structure d'un document

```yaml
openapi: 3.1.0
info:
  title: API Commandes
  version: 1.2.0
  description: Gestion des commandes clients.
servers:
  - url: https://api.example.com/api/v1
tags:
  - name: orders
    description: Commandes

paths:
  /orders/{orderId}:
    get:
      tags: [orders]
      summary: Lire une commande
      operationId: getOrder
      security:
        - bearerAuth: []
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        "200":
          description: Commande trouvée
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Order"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "404":
          $ref: "#/components/responses/NotFound"

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    Order:
      type: object
      required: [id, status, createdAt]
      properties:
        id:
          type: string
          format: uuid
        status:
          type: string
          enum: [pending, paid, shipped, cancelled]
        totalCents:
          type: integer
          minimum: 0
          example: 4990
        createdAt:
          type: string
          format: date-time

    Error:
      type: object
      required: [error]
      properties:
        error:
          type: object
          required: [code, message]
          properties:
            code:
              type: string
              example: ORDER_NOT_FOUND
            message:
              type: string

  responses:
    Unauthorized:
      description: Authentification requise ou jeton invalide
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/Error"
    NotFound:
      description: Ressource introuvable
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/Error"
```

Points clés :

- **`components`** : schémas, réponses et paramètres définis une fois, réutilisés par `$ref`.
- **`operationId`** unique : il devient le nom de la méthode dans les clients générés.
- **Toutes les réponses d'erreur** sont décrites, avec le format du [catalogue d'erreurs](error-catalog.md).
- **`required`, `enum`, `format`, `minimum`** : plus le schéma est précis, plus la validation est utile.
- **`example`** sur les champs : la documentation et les mocks deviennent parlants.

---

## ⚡ Avec FastAPI (code first)

FastAPI génère le contrat à partir des types Pydantic et des annotations :

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="API Commandes", version="1.2.0")


class Order(BaseModel):
    id: str = Field(examples=["5f0c…"])
    status: str
    total_cents: int = Field(ge=0, examples=[4990])


@app.get(
    "/orders/{order_id}",
    tags=["orders"],
    summary="Lire une commande",
    responses={404: {"description": "Commande introuvable"}},
)
def get_order(order_id: str) -> Order:
    ...
```

- Documentation interactive sur `/docs` (Swagger UI) et `/redoc`.
- Contrat brut sur `/openapi.json`, à **exporter et versionner** pour suivre ses évolutions :

```bash
python -c "import json; from app.main import app; print(json.dumps(app.openapi(), indent=2))" > openapi.json
```

Voir l'[exemple FastAPI](../../examples/fastapi-example.md).

---

## 🔄 OpenAPI dans la CI

| Contrôle | Outil | Ce qui est vérifié |
|----------|-------|--------------------|
| **Lint** du contrat | Spectral, Redocly CLI | syntaxe, règles de style (descriptions, `operationId`, exemples) |
| **Changements cassants** | oasdiff | champ supprimé, type modifié, paramètre devenu obligatoire |
| **Contrat respecté** par l'API | Schemathesis | l'API répond conformément au contrat, y compris sur des entrées inattendues |

```yaml
# .github/workflows/openapi.yml
name: openapi

on:
  pull_request:

permissions:
  contents: read

jobs:
  openapi-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          docker run --rm -v "$PWD:/work:ro" -w /work stoplight/spectral:6 \
            lint openapi.yaml --fail-severity error

  openapi-breaking-changes:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      # Contrat de la branche cible de la pull request
      - run: git show "origin/${GITHUB_BASE_REF}:openapi.yaml" > base.yaml
      - run: |
          docker run --rm -v "$PWD:/work:ro" -w /work tufin/oasdiff:latest \
            breaking base.yaml openapi.yaml --fail-on ERR
```

⚠️ En usage réel, épingler la version (ou le digest) de l'image `oasdiff` plutôt que `latest`.

---

## ✅ Bonnes pratiques

- [ ] Un seul contrat de référence, versionné dans le dépôt
- [ ] `info.version` suit la version de l'API
- [ ] Chaque opération a un `summary`, un `operationId` et un `tag`
- [ ] Toutes les réponses, y compris les erreurs, sont décrites
- [ ] Schémas réutilisés par `$ref`, pas copiés
- [ ] Exemples sur les champs et les réponses
- [ ] Schéma de sécurité déclaré
- [ ] Lint et détection des changements cassants en CI

Voir aussi la [checklist de conception d'API](api-design-checklist.md).

---

## 📚 Références

- [Spécification OpenAPI](https://spec.openapis.org/oas/latest.html)
- [Spectral](https://github.com/stoplightio/spectral)
- [oasdiff](https://github.com/oasdiff/oasdiff)
- [Schemathesis](https://schemathesis.readthedocs.io/)
