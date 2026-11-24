# ⚠️ Catalogue d'erreurs

## 🎯 Objectif

Centraliser les erreurs qu'une API peut renvoyer, avec **un format unique** et
**des codes stables**, pour que les clients puissent les traiter automatiquement
et que les équipes parlent le même langage.

---

## 📦 Format de réponse

Toutes les erreurs de l'API ont la même structure :

```json
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "La commande demandée est introuvable.",
    "details": [],
    "request_id": "a1b2c3d4"
  }
}
```

| Champ | Obligatoire | Rôle |
|-------|-------------|------|
| `code` | ✅ | identifiant **stable**, en majuscules, que le client peut tester |
| `message` | ✅ | explication lisible par un humain ; peut évoluer, ne pas le tester côté client |
| `details` | ❌ | précisions, par exemple les champs invalides |
| `request_id` | ❌ | identifiant de la requête, à fournir au support ; retrouvé dans les logs |

Exemple d'erreur de validation avec `details` :

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Certains champs sont invalides.",
    "details": [
      { "field": "email", "code": "INVALID_FORMAT", "message": "Adresse e-mail invalide." },
      { "field": "quantity", "code": "OUT_OF_RANGE", "message": "Doit être comprise entre 1 et 100." }
    ]
  }
}
```

💡 Alternative normalisée : le format **Problem Details** ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457),
`application/problem+json`). L'important est d'en choisir **un seul** pour toute l'API.

---

## 🏷️ Convention de nommage des codes

- `MAJUSCULES_AVEC_TIRETS_BAS`
- Forme **`RESSOURCE_PROBLÈME`** : `ORDER_NOT_FOUND`, `USER_ALREADY_EXISTS`
- Codes **génériques** pour les cas transverses : `VALIDATION_ERROR`, `INTERNAL_ERROR`
- Un code publié **ne change plus** de sens ; on en crée un nouveau si besoin

---

## 📋 Catalogue

### Erreurs génériques

| Code | HTTP | Signification | Action côté client |
|------|------|---------------|--------------------|
| `VALIDATION_ERROR` | 400 / 422 | données d'entrée invalides (voir `details`) | corriger la saisie |
| `MALFORMED_REQUEST` | 400 | JSON invalide ou illisible | corriger la requête |
| `RESOURCE_NOT_FOUND` | 404 | ressource ou route inexistante | vérifier l'identifiant |
| `METHOD_NOT_ALLOWED` | 405 | méthode HTTP non prise en charge | utiliser la bonne méthode |
| `CONFLICT` | 409 | état incompatible avec l'opération | relire la ressource puis réessayer |
| `RATE_LIMITED` | 429 | trop de requêtes | attendre `Retry-After` |
| `INTERNAL_ERROR` | 500 | erreur imprévue côté serveur | réessayer plus tard, contacter le support avec `request_id` |
| `SERVICE_UNAVAILABLE` | 503 | service temporairement indisponible | réessayer avec délai croissant |

### Authentification et autorisation

| Code | HTTP | Signification | Action côté client |
|------|------|---------------|--------------------|
| `AUTHENTICATION_REQUIRED` | 401 | aucun jeton fourni | s'authentifier |
| `INVALID_TOKEN` | 401 | jeton invalide ou mal formé | s'authentifier à nouveau |
| `TOKEN_EXPIRED` | 401 | jeton expiré | rafraîchir le jeton |
| `ACCESS_DENIED` | 403 | droits insuffisants pour cette opération | ne pas réessayer |

### Erreurs métier (exemples)

| Code | HTTP | Signification |
|------|------|---------------|
| `USER_NOT_FOUND` | 404 | utilisateur introuvable |
| `USER_ALREADY_EXISTS` | 409 | adresse e-mail déjà utilisée |
| `ORDER_NOT_FOUND` | 404 | commande introuvable |
| `ORDER_ALREADY_PAID` | 409 | la commande est déjà payée |
| `INSUFFICIENT_STOCK` | 422 | stock insuffisant pour la quantité demandée |

Chaque projet complète cette section avec ses propres erreurs métier.

---

## 🔐 Sécurité des messages d'erreur

- ❌ Pas de trace d'exécution, de requête SQL, de chemin de fichier ni de version de logiciel dans la réponse.
- ❌ Pas d'information qui aide à l'énumération : sur un échec de connexion, répondre
  `INVALID_CREDENTIALS` sans préciser si c'est l'identifiant ou le mot de passe qui est faux.
- Pour une ressource appartenant à un autre utilisateur, `404` peut être préférable à `403`
  pour ne pas confirmer son existence.
- ✅ Le détail technique va **dans les logs**, retrouvable grâce au `request_id`.

---

## 🛠️ Mise en œuvre

- Une **classe d'erreur unique** dans le code, convertie en réponse par un gestionnaire global.
- Les erreurs imprévues sont interceptées et transformées en `INTERNAL_ERROR` (et journalisées).
- Les codes sont documentés dans le contrat [OpenAPI](openapi-guide.md) (schéma `Error` partagé).

Exemple complet en FastAPI : [exemple FastAPI](../../examples/fastapi-example.md).

---

## ✅ Checklist

- [ ] Un format d'erreur unique pour toute l'API
- [ ] Des codes stables, documentés dans ce catalogue
- [ ] Le bon code HTTP pour chaque erreur
- [ ] Aucun détail technique exposé
- [ ] `request_id` présent dans la réponse et dans les logs
- [ ] Le catalogue est mis à jour à chaque nouvelle erreur
