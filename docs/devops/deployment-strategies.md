# 🚀 Stratégies de déploiement

## 🎯 Objectif

Mettre une nouvelle version en production **sans interruption de service**,
en **limitant l'impact** d'un éventuel problème et en gardant la possibilité
de **revenir en arrière** rapidement.

---

## 🗺️ Vue d'ensemble

| Stratégie | Principe | Interruption | Retour arrière | Coût en ressources | Complexité |
|-----------|----------|--------------|----------------|--------------------|------------|
| **Recreate** | on arrête l'ancienne version, on démarre la nouvelle | ⚠️ oui | redéploiement | × 1 | faible |
| **Rolling update** | remplacement progressif des instances | non | progressif | × 1 + marge | faible |
| **Blue/Green** | deux environnements complets, bascule du trafic | non | instantané | × 2 | moyenne |
| **Canary** | une petite part du trafic sur la nouvelle version, puis augmentation | non | instantané | × 1 + marge | élevée |
| **Feature flags** | le code est déployé, la fonctionnalité est activée à part | non | désactivation du flag | × 1 | moyenne (dans le code) |

---

## 🔁 Recreate

```text
v1 v1 v1  →  (arrêt)  →  v2 v2 v2
```

- **Quand** : environnements de développement, applications internes tolérant une coupure,
  ou quand deux versions **ne peuvent pas coexister** (migration de base incompatible).
- **Limite** : coupure de service le temps du redémarrage.

---

## 🔄 Rolling update

```text
v1 v1 v1  →  v2 v1 v1  →  v2 v2 v1  →  v2 v2 v2
```

Les instances sont remplacées une par une (ou par lots). C'est la stratégie
**par défaut de Kubernetes** (`Deployment`).

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # jamais moins de 3 instances disponibles
      maxSurge: 1            # au plus 1 instance supplémentaire pendant le déploiement
  template:
    spec:
      containers:
        - name: api
          image: registry.example.com/api:1.4.2
          readinessProbe:    # le trafic n'arrive que quand la nouvelle instance est prête
            httpGet:
              path: /health
              port: 8000
```

- **Prérequis** : les versions N et N+1 doivent **coexister** quelques minutes
  (API et schéma de base compatibles).
- **Indispensable** : une `readinessProbe`, sinon le trafic arrive sur une instance qui démarre.
- **Retour arrière** : `kubectl rollout undo deployment/api`.

---

## 🔵🟢 Blue/Green

```text
                 ┌──────────────┐
utilisateurs ──► │ load balancer│──► Blue  (v1)  ← en production
                 └──────────────┘    Green (v2)  ← déployé, testé, en attente
                       │
                 bascule du trafic
```

1. La nouvelle version est déployée sur l'environnement inactif (Green).
2. Elle est testée (smoke tests) sans recevoir de trafic utilisateur.
3. Le trafic est basculé de Blue vers Green en une opération.
4. Blue reste disponible pour un **retour arrière instantané**, puis devient la cible du prochain déploiement.

- **Avantage** : bascule et retour arrière immédiats, test en conditions réelles avant bascule.
- **Limites** : double infrastructure pendant le déploiement, gestion délicate des données
  (une base partagée doit rester compatible avec les deux versions).

Avec Kubernetes, la bascule peut se faire en changeant le sélecteur d'un `Service` :

```bash
kubectl patch service api -p '{"spec":{"selector":{"app":"api","version":"green"}}}'
```

---

## 🐤 Canary

```text
trafic : 95 % → v1        puis   75 % → v1        puis   0 % → v1
          5 % → v2                25 % → v2              100 % → v2
```

La nouvelle version reçoit d'abord une **petite part du trafic**. Si les indicateurs
(taux d'erreur, latence) restent bons, la part augmente par paliers ; sinon, on revient à 0 %.

- **Avantage** : un problème ne touche qu'une faible part des utilisateurs, validation sur du trafic réel.
- **Prérequis** : un routage pondéré (Ingress, Gateway API, service mesh) et
  un **monitoring fiable** pour décider (voir [monitoring et logging](monitoring-logging.md)).
- **Outils** : Argo Rollouts, Flagger, ou annotations canary d'Ingress NGINX.

---

## 🚩 Feature flags

Le code de la nouvelle fonctionnalité est **déployé mais désactivé**, puis activé
indépendamment du déploiement (pour un groupe d'utilisateurs, un pourcentage, un client).

- **Avantage** : séparer **déployer** et **livrer une fonctionnalité** ; désactivation en quelques secondes.
- **Limite** : des flags oubliés complexifient le code ; prévoir leur suppression.
- **Outils** : Unleash, OpenFeature, LaunchDarkly, GrowthBook.

---

## 🗄️ Le piège : les migrations de base de données

Avec rolling, blue/green ou canary, **deux versions tournent en même temps** sur la même base.
Une migration doit donc rester compatible avec l'ancienne version : c'est le principe
**expand / contract**.

| Étape | Exemple : renommer `nom` en `last_name` |
|-------|----------------------------------------|
| 1. **Expand** | ajouter `last_name`, écrire dans les deux colonnes |
| 2. Migrer les données | recopier `nom` dans `last_name` |
| 3. Basculer la lecture | la nouvelle version lit `last_name` |
| 4. **Contract** | une fois l'ancienne version retirée, supprimer `nom` |

❌ Renommer la colonne en une seule migration casse l'ancienne version encore en service.

---

## 🧭 Choisir sa stratégie

| Contexte | Stratégie conseillée |
|----------|----------------------|
| Environnement de dev ou recette | Recreate ou rolling |
| Application Kubernetes standard | Rolling update avec probes |
| Retour arrière instantané exigé, infrastructure doublable | Blue/Green |
| Fort trafic, risque élevé, bon monitoring | Canary |
| Fonctionnalité à activer progressivement ou par client | Feature flags (avec n'importe quelle stratégie) |

---

## 🔄 Dans la pipeline CI/CD

- Déployer **une image immuable** (tag de version ou digest), jamais `latest`.
- Même artefact promu d'environnement en environnement (recette → préproduction → production).
- **Smoke tests automatiques** après déploiement ; échec = retour arrière.
- **Validation manuelle** et environnement protégé pour la production :

```yaml
deploy-production:
  if: startsWith(github.ref, 'refs/tags/v')
  runs-on: ubuntu-latest
  # Environment protégé : "Required reviewers" impose une approbation
  environment:
    name: production
    url: https://api.example.com
  # Jamais deux déploiements de production en parallèle
  concurrency:
    group: deploy-production
    cancel-in-progress: false
  steps:
    - uses: actions/checkout@v4
    - env:
        KUBECONFIG_CONTENT: ${{ secrets.KUBECONFIG_CONTENT }}
      run: |
        export KUBECONFIG="$RUNNER_TEMP/kubeconfig"
        (umask 077 && printf '%s\n' "$KUBECONFIG_CONTENT" > "$KUBECONFIG")
        helm upgrade --install api ./chart --set image.tag="$GITHUB_REF_NAME" --wait --atomic
```

`--atomic` annule automatiquement le déploiement Helm s'il échoue. L'environment
GitHub porte les secrets propres à la production, l'approbation obligatoire et
l'historique des déploiements ; relancer le workflow d'un tag précédent redéploie
cette version. Exemple complet (Compose ou Helm, au choix) :
[pipeline-python-centralisee-portfolio](https://github.com/JulienMilon-DevSecOps/pipeline-python-centralisee-portfolio).

---

## ✅ Bonnes pratiques

- [ ] Sondes de santé (`readinessProbe`, `livenessProbe`) sur toutes les applications
- [ ] Migrations de base compatibles avec la version précédente
- [ ] Retour arrière documenté **et testé**
- [ ] Indicateurs surveillés pendant et après le déploiement
- [ ] Arrêt propre de l'application (`SIGTERM`, fin des requêtes en cours)
- [ ] Déploiements fréquents et petits plutôt que rares et volumineux
