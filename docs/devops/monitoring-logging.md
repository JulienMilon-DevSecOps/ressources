# 📊 Monitoring et logging

## 🎯 Objectif

Savoir **à tout moment** si un système fonctionne, être **alerté avant les utilisateurs**
quand il ne fonctionne plus, et disposer des informations pour **comprendre pourquoi**.

---

## 🔭 Les trois piliers de l'observabilité

| Pilier | Répond à | Exemple | Outils courants |
|--------|----------|---------|-----------------|
| **Métriques** | *Est-ce que ça va ?* Valeurs chiffrées dans le temps | requêtes/s, taux d'erreur, latence p95, CPU | Prometheus, Grafana |
| **Logs** | *Que s'est-il passé ?* Événements horodatés | `ERROR paiement refusé order_id=42` | Loki, ELK / OpenSearch |
| **Traces** | *Où est passé le temps ?* Parcours d'une requête entre services | API → service → base : 1,2 s dont 1 s en base | OpenTelemetry, Tempo, Jaeger |

Les métriques **détectent**, les logs et les traces **expliquent**.

---

## 📈 Quelles métriques suivre

### Pour un service (méthode RED)

| Métrique | Question |
|----------|----------|
| **R**ate | combien de requêtes par seconde ? |
| **E**rrors | quelle part de ces requêtes échoue ? |
| **D**uration | combien de temps prennent-elles (p50, p95, p99) ? |

### Pour une ressource (méthode USE)

| Métrique | Question |
|----------|----------|
| **U**tilization | à quel point la ressource est-elle occupée (CPU 80 %) ? |
| **S**aturation | du travail attend-il (file d'attente, swap, load) ? |
| **E**rrors | la ressource produit-elle des erreurs (disque, réseau) ? |

💡 Suivre les **percentiles** (p95, p99) plutôt que la moyenne : une moyenne de 200 ms
peut cacher 5 % de requêtes à 3 s.

---

## 🧪 Instrumenter une application

Exemple : exposer les métriques d'une API FastAPI au format Prometheus.

```python
from fastapi import FastAPI
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()
Instrumentator().instrument(app).expose(app)   # expose /metrics
```

Prometheus vient ensuite **lire** (scrape) `/metrics` à intervalle régulier :

```yaml
# prometheus.yml
scrape_configs:
  - job_name: api
    scrape_interval: 15s
    static_configs:
      - targets: ["api:8000"]
```

Sur Kubernetes, l'opérateur Prometheus découvre les cibles via des `ServiceMonitor`.

---

## 📝 Bien écrire ses logs

### Logs structurés

Un log JSON se filtre et s'agrège ; une phrase libre se lit seulement.

```json
{"timestamp": "2026-11-22T10:15:32Z", "level": "ERROR", "service": "api",
 "message": "paiement refusé", "order_id": 42, "request_id": "a1b2c3", "duration_ms": 845}
```

### Niveaux

| Niveau | Usage |
|--------|-------|
| `DEBUG` | détail pour le développement, désactivé en production |
| `INFO` | événements normaux et utiles (démarrage, commande créée) |
| `WARNING` | anomalie gérée (nouvelle tentative, valeur par défaut utilisée) |
| `ERROR` | échec d'une opération, nécessite une analyse |
| `CRITICAL` | le service ne peut plus fonctionner |

### Règles

- Écrire sur **stdout / stderr** : le conteneur ou l'orchestrateur collecte (pas de fichiers dans le conteneur).
- Ajouter un **identifiant de corrélation** (`request_id`, `trace_id`) propagé entre services.
- ❌ **Jamais** de mot de passe, jeton, donnée personnelle ou de santé dans les logs
  (voir [sécurité](../security/security-best-practices.md)).
- Définir une **durée de rétention** adaptée (coût, obligations légales).

---

## 🚨 Alerter

Une alerte doit être **actionnable** : si personne n'a rien à faire en la recevant,
c'est une alerte de trop.

- Alerter sur les **symptômes vus par l'utilisateur** (taux d'erreur, latence, indisponibilité)
  plutôt que sur les causes (CPU à 90 % n'est pas forcément un problème).
- Une durée minimale (`for`) pour éviter les alertes sur un pic ponctuel.
- Deux niveaux : **critique** (intervention immédiate) et **avertissement** (à traiter en heures ouvrées).
- Chaque alerte renvoie vers une **procédure** (runbook).

```yaml
# Règle d'alerte Prometheus
groups:
  - name: api
    rules:
      - alert: TauxErreurEleve
        expr: |
          sum(rate(http_requests_total{job="api", status=~"5.."}[5m]))
            / sum(rate(http_requests_total{job="api"}[5m])) > 0.05
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Plus de 5 % d'erreurs 5xx sur l'API depuis 10 minutes"
          runbook_url: "https://wiki.example.com/runbooks/api-erreurs"
```

---

## 🎯 SLI, SLO, SLA

| Terme | Définition | Exemple |
|-------|------------|---------|
| **SLI** (indicateur) | ce qu'on mesure | part des requêtes réussies en moins de 500 ms |
| **SLO** (objectif) | la cible interne | 99,5 % sur 30 jours |
| **SLA** (accord) | l'engagement contractuel, avec pénalités | 99 % de disponibilité mensuelle |

Le SLO fixe un **budget d'erreur** (0,5 % ici) : tant qu'il reste du budget, on peut
déployer et prendre des risques ; quand il est épuisé, on priorise la fiabilité.

---

## 🏗️ Une pile type

```text
Applications ──/metrics──► Prometheus ──► Alertmanager ──► Teams / Slack / e-mail
     │                          │
     └──stdout──► collecteur ──► Loki ──┐
                 (Promtail,             ├──► Grafana (tableaux de bord, exploration)
                  Alloy, Fluent Bit)    │
     └──OTLP───► OpenTelemetry ─► Tempo ┘
```

Sur Kubernetes, `kube-prometheus-stack` (chart Helm) installe Prometheus, Alertmanager,
Grafana et les exporters de base (nœuds, kubelet, kube-state-metrics) en une fois.

---

## ✅ Checklist

- [ ] Métriques RED exposées par chaque service
- [ ] Métriques système (nœuds, conteneurs) collectées
- [ ] Logs structurés sur stdout, centralisés, sans données sensibles
- [ ] Identifiant de corrélation propagé
- [ ] Un tableau de bord par service (trafic, erreurs, latence, saturation)
- [ ] Alertes sur les symptômes, actionnables, avec runbook
- [ ] SLO définis pour les services critiques
- [ ] Rétention des logs et métriques définie
