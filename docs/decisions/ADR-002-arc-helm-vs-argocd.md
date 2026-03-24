# ADR-002 — ARC : Helm direct vs ArgoCD

**Date** : 2026-03-23  
**Statut** : Accepté

## Contexte

Actions Runner Controller (ARC) nécessite des ressources cluster-scoped (CRDs, ClusterRoles) pour fonctionner. L'instance ArgoCD de ce projet tourne en mode **namespaced** dans `openshift-operators`.

## Problème

ArgoCD en mode namespaced ne peut pas créer de ressources cluster-scoped :

```
Error: Failed to load live state: cluster level ClusterRole
"actions-runner-controller-proxy" can not be managed when
in namespaced mode
```

Tentatives de fix ArgoCD échouées :
- Patch `clusterResourceWhitelist` → champs non reconnus par l'opérateur
- `add-cluster-role-to-user cluster-admin` → insuffisant
- Le problème est architectural : l'opérateur ArgoCD Community force le mode namespaced

## Décision

Installer ARC via **Helm direct depuis WSL2** — pas via ArgoCD.

```bash
helm upgrade --install actions-runner-controller \
  actions-runner-controller/actions-runner-controller \
  --namespace actions-runner-system \
  --version 0.23.7 \
  ...
```

## Justification

```
Approche           Avantages              Inconvénients
──────────────────────────────────────────────────────
Helm direct        ✅ Fonctionne          ⚠️ Pas dans GitOps
                   ✅ Simple              ⚠️ Déploiement manuel
                   ✅ Droits cluster-admin

ArgoCD namespaced  ✅ GitOps              ❌ ClusterRole impossible
                   ✅ Cohérent            ❌ Erreur namespaced mode
```

## Conséquences

- ARC Controller géré hors GitOps (Helm direct)
- Le RunnerDeployment reste dans le repo Git (`manifests/actions-runner/`)
- ArgoCD continue de gérer toutes les apps namespace-scoped (Keycloak, Vault, Grafana...)

## Note architecture

```
┌──────────────────────────────────────────────────────┐
│  Géré par ArgoCD (namespace-scoped)                  │
│  keycloak, vault, grafana, loki, eso, kyverno        │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  Géré par Helm direct (cluster-scoped)               │
│  ARC Controller (CRDs + ClusterRoles)                │
└──────────────────────────────────────────────────────┘
```
