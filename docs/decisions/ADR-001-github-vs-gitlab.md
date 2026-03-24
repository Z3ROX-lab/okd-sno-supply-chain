# ADR-001 — GitHub Actions vs GitLab CI

**Date** : 2026-03-23  
**Statut** : Accepté

## Contexte

Pour la Phase 5 (Supply Chain Security Pipeline), on avait besoin d'un orchestrateur CI/CD SaaS avec un runner self-hosted dans OKD SNO.

Le plan initial était GitLab.com (free tier) + GitLab Runner pod dans OKD.

## Problème

GitLab.com impose depuis 2023 une vérification par carte de crédit pour activer les pipelines CI/CD — même pour les runners self-hosted qui n'utilisent pas les shared runners GitLab.

![GitLab credit card requirement](../screenshots/01-gitlab-credit-card-why-github.png)

## Décision

Utiliser **GitHub Actions** avec un runner self-hosted dans OKD via **Actions Runner Controller (ARC)**.

## Justification

```
Critère              GitLab.com    GitHub Actions
─────────────────────────────────────────────────
Friction inscription CB requise   ✅ Aucune
Portfolio visibility ✅ Bon        ✅✅ Excellent (Z3ROX-lab)
Pattern enterprise   ✅ Courant    ✅ Très courant
Self-hosted runner   ✅ GitLab Rn  ✅ ARC (opérateur K8s)
Pipeline YAML        .gitlab-ci   .github/workflows
Chaîne supply chain  Identique    Identique
```

## Conséquences

- Pipeline en `.github/workflows/supply-chain.yml` au lieu de `.gitlab-ci.yml`
- Runner géré via ARC (Actions Runner Controller) au lieu de gitlab-runner Helm chart
- Tout le reste de la chaîne (Trivy, Cosign, Harbor, ArgoCD) est identique

## Note

Le pattern **SaaS Git + runners on-premise** est identique dans les deux cas — c'est exactement ce que font les entreprises en production.
