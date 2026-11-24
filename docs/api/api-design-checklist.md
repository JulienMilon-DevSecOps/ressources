# 📡 Checklist de conception d'API

## 🎯 Objectif

Vérifier la qualité d'une API REST **avant sa publication** : cohérence, sécurité,
documentation et capacité à évoluer sans casser les clients.

À utiliser en revue de conception, puis en revue de pull request.

---

## 🔌 Ressources et URLs

- [ ] Les URLs désignent des **ressources** (noms), pas des actions (verbes)
  - ✅ `POST /orders` ❌ `POST /createOrder`
- [ ] Noms au **pluriel** et en minuscules, mots séparés par des tirets : `/order-items`
- [ ] Hiérarchie limitée à un niveau d'imbrication : `/users/42/orders`, pas plus profond
- [ ] Les identifiants ne révèlent pas d'information exploitable (UUID plutôt qu'un compteur si l'énumération est un risque)
- [ ] Une même convention de nommage des champs dans toute l'API (`snake_case` ou `camelCase`, pas les deux)

## 🔁 Méthodes HTTP

| Méthode | Usage | Idempotente |
|---------|-------|-------------|
| `GET` | lire | ✅ |
| `POST` | créer, ou déclencher un traitement | ❌ |
| `PUT` | remplacer entièrement | ✅ |
| `PATCH` | modifier partiellement | ❌ (en général) |
| `DELETE` | supprimer | ✅ |

- [ ] `GET` ne modifie jamais rien
- [ ] Les opérations non idempotentes sensibles (paiement) acceptent une clé d'idempotence (`Idempotency-Key`)

## 🚦 Codes de statut

- [ ] `200` lecture ou modification réussie, `201` création (avec en-tête `Location`), `204` sans contenu
- [ ] `400` requête invalide, `422` validation métier échouée
- [ ] `401` non authentifié, `403` authentifié mais non autorisé
- [ ] `404` ressource introuvable, `409` conflit (doublon, version)
- [ ] `429` trop de requêtes (avec `Retry-After`)
- [ ] `500` réservé aux erreurs imprévues, jamais pour une erreur du client

## 📦 Données

- [ ] JSON en entrée et en sortie, `Content-Type: application/json`
- [ ] Dates au format **ISO 8601** en UTC : `2026-11-22T10:15:32Z`
- [ ] Montants en entier (centimes) ou en chaîne décimale, jamais en flottant
- [ ] **Pagination** sur toutes les listes (`limit` / `offset` ou curseur), avec une taille maximale
- [ ] Filtres et tri par paramètres de requête : `GET /orders?status=paid&sort=-created_at`
- [ ] Les réponses n'exposent que les champs utiles (pas de mot de passe haché, pas de champ interne)

```json
{
  "items": [{ "id": "a1b2", "status": "paid" }],
  "total": 134,
  "limit": 20,
  "offset": 40
}
```

## ⚠️ Erreurs

- [ ] Un **format d'erreur unique** pour toute l'API (voir le [catalogue d'erreurs](error-catalog.md))
- [ ] Un code d'erreur stable et lisible par une machine, en plus du message
- [ ] Pas de trace d'exécution ni de détail technique dans la réponse

## 🔐 Sécurité

- [ ] HTTPS uniquement
- [ ] Authentification standard (OAuth 2.0 / OpenID Connect, jetons JWT à durée courte)
- [ ] **Autorisation vérifiée sur chaque ressource** : l'utilisateur 42 ne lit pas la commande de l'utilisateur 43
  (première faille du [OWASP API Security Top 10](https://owasp.org/API-Security/))
- [ ] Validation stricte des entrées (types, longueurs, formats), champs inconnus refusés
- [ ] Limitation de débit (rate limiting) et taille maximale des requêtes
- [ ] CORS limité aux origines connues
- [ ] Aucun secret ni donnée sensible dans les URLs (elles finissent dans les logs)

Voir aussi les [bonnes pratiques de sécurité](../security/security-best-practices.md).

## 🧬 Versioning et évolution

- [ ] Version majeure dans l'URL (`/api/v1`) ou dans un en-tête, choisie une fois pour toutes
- [ ] Changements **compatibles** sans nouvelle version : ajout d'un champ, d'un endpoint, d'un paramètre optionnel
- [ ] Changements **cassants** = nouvelle version majeure : suppression ou renommage d'un champ, changement de type
- [ ] Ancienne version dépréciée avec un délai annoncé (en-têtes `Deprecation` et `Sunset`)

## 📚 Documentation

- [ ] Contrat **OpenAPI** à jour, généré ou vérifié en CI (voir le [guide OpenAPI](openapi-guide.md))
- [ ] Chaque endpoint documente ses paramètres, ses réponses **et ses erreurs**
- [ ] Exemples de requêtes et de réponses
- [ ] Méthode d'authentification expliquée

## 🩺 Exploitation

- [ ] Endpoint de santé (`/health`) pour les sondes et le load balancer
- [ ] Identifiant de requête (`X-Request-ID`) renvoyé et présent dans les logs
- [ ] Métriques exposées (voir [monitoring](../devops/monitoring-logging.md))

## 🧪 Tests

- [ ] Tests du cas nominal **et** des cas d'erreur de chaque endpoint
- [ ] Tests des règles d'autorisation (accès refusé à la ressource d'un autre utilisateur)
- [ ] Tests de contrat contre le schéma OpenAPI

Voir la [stratégie de tests](../testing/testing-strategy.md) et
l'[exemple FastAPI](../../examples/fastapi-example.md).
