# 🔐 OKD SNO Supply Chain Security Platform

> **Enterprise-grade CI/CD supply chain security on a Single Node OpenShift cluster**  
> GitHub Actions · Trivy · Cosign · Harbor · ArgoCD · Kyverno

[![OKD](https://img.shields.io/badge/OKD-4.15-red?logo=redhat)](https://www.okd.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-orange?logo=argo)](https://argoproj.github.io/cd/)
[![Harbor](https://img.shields.io/badge/Harbor-Registry-blue?logo=harbor)](https://goharbor.io/)
[![Cosign](https://img.shields.io/badge/Cosign-Signed-green?logo=sigstore)](https://docs.sigstore.dev/)
[![Trivy](https://img.shields.io/badge/Trivy-Scanned-purple?logo=aquasecurity)](https://trivy.dev/)
[![Kyverno](https://img.shields.io/badge/Kyverno-Policy-yellow)](https://kyverno.io/)

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GitHub.com (SaaS)                            │
│   Z3ROX-lab/okd-sno-supply-chain                                   │
│              │  git push                                            │
│              ▼                                                      │
│   .github/workflows/supply-chain.yml                               │
└──────────────┬──────────────────────────────────────────────────────┘
               │  job dispatch
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│              OKD SNO 4.15 (192.168.241.10)                         │
│                                                                     │
│  ┌──────────────────────┐     ┌──────────────────────────────────┐ │
│  │  GitHub Actions      │     │           ArgoCD                 │ │
│  │  Self-hosted Runner  │     │  root-app (App of Apps)          │ │
│  │  (pod ARC)           │     │  ├── keycloak    ✅              │ │
│  └────────┬─────────────┘     │  ├── vault       ✅              │ │
│           │                   │  ├── grafana     ✅              │ │
│      Pipeline:                │  ├── loki        ✅              │ │
│      1. docker build          │  ├── eso         ✅              │ │
│      2. trivy scan ──gate──▶  │  ├── kyverno     ✅              │ │
│         CRITICAL=0 ?          │  └── app-demo    🚀              │ │
│      3. cosign sign           └──────────────────────────────────┘ │
│      4. harbor push                                                 │
│      5. argocd sync                                                 │
└───────────┬─────────────────────────────────────────────────────────┘
            │ push signed image
            ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Harbor Registry (192.168.241.20)                      │
│              harbor.okd.lab                                         │
│                                                                     │
│  Projects:                                                          │
│  ├── library/         ← images mirrorées (nginx, httpd, caddy...)  │
│  └── supply-chain/    ← images buildées + signées Cosign           │
│      └── app-demo:<sha>                                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🛡️ Security Pipeline

```
Code Push (git push master)
         │
         ▼
┌─────────────────┐
│  docker build   │  ← FROM harbor.okd.lab/library/httpd:alpine
│  (runner pod)   │    + apk upgrade libexpat (fix CVE-2026-32767)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  trivy scan     │  ← v0.69.3 pinné (pré-attaque TeamPCP)
│  CRITICAL gate  │
└────────┬────────┘
         │
    ┌────┴────┐
    │  CVE ?  │
    └────┬────┘
    CRITICAL │ NON → continue
    → FAIL   ▼
    pipeline ┌─────────────────┐
             │  docker push    │  → harbor.okd.lab/supply-chain/app-demo
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │  cosign sign    │  ← keyless Sigstore
             │  (ephemeral key)│
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │  argocd sync    │  → déploie app-demo dans OKD
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │  Kyverno verify │  ← vérifie signature Cosign
             │  image policy   │
             └─────────────────┘
```

---

## 📦 Stack

| Composant | Version | Rôle |
|-----------|---------|------|
| **OKD SNO** | 4.15 | Single Node OpenShift — cluster de démo |
| **GitHub Actions** | - | CI/CD orchestrator (SaaS) |
| **ARC** | 0.23.7 | Actions Runner Controller — gère le runner pod |
| **Self-hosted Runner** | summerwind | Runner pod dans OKD (2 containers) |
| **Harbor** | 2.x | Registry OCI privée |
| **Trivy** | 0.69.3 | Scan CVE (pinné — pré-attaque TeamPCP) |
| **Cosign** | 2.4.1 | Signature keyless via Sigstore |
| **ArgoCD** | Community 0.17.0 | GitOps — déploiement continu |
| **Kyverno** | 1.12.0 | Policy-as-Code — vérification signature |
| **HashiCorp Vault** | 0.28.0 | Secrets management |
| **Keycloak** | 24.x | IAM / OIDC SSO |
| **Loki + Grafana** | latest | Observabilité |

---

## 🗂️ Repository Structure

```
okd-sno-supply-chain/
├── .github/
│   └── workflows/
│       └── supply-chain.yml       # Pipeline principal
├── app/
│   ├── Dockerfile                 # httpd:alpine + libexpat fix
│   └── src/
│       └── index.html             # Page de démo supply chain
├── argocd/
│   └── applications/
│       └── actions-runner.yaml    # RunnerDeployment
├── manifests/
│   ├── actions-runner/
│   │   └── values.yaml            # ARC Helm values
│   └── argocd/
│       └── 03-repo-helm-arc.yaml  # Repo Helm ARC
├── docs/
│   ├── phase5-supply-chain.md     # Documentation Phase 5
│   ├── decisions/
│   │   ├── ADR-001-github-vs-gitlab.md
│   │   ├── ADR-002-arc-helm-vs-argocd.md
│   │   ├── ADR-003-base-image-selection.md
│   │   └── ADR-004-trivy-teampcp.md
│   └── screenshots/               # 15 screenshots documentés
└── README.md
```

---

## 🚀 Phases du Projet

### ✅ Phase 1 — Bootstrap OKD SNO
- Installation OKD 4.15 agent-based sur GEEKOM A6 (32GB DDR5)
- ArgoCD Community Operator v0.17.0
- App of Apps pattern (`root-app`)
- Proxy tinyproxy (`10.128.0.2:8888`) pour accès internet cluster

### ✅ Phase 2a — IAM : Keycloak OIDC SSO
- Keycloak déployé via ArgoCD (Helm)
- OIDC SSO configuré pour OKD console
- Secrets migrés vers SealedSecrets

### ✅ Phase 2b — Secrets : HashiCorp Vault
- Vault v0.28.0 dev mode via ArgoCD
- External Secrets Operator (ESO) intégré

### ✅ Phase 3 — Observabilité
- Loki + Grafana déployés via ArgoCD
- Dashboards OKD + pipeline métriques

### ✅ Phase 4 — Policy-as-Code : Kyverno
- Kyverno v1.12.0 installé (images Harbor)
- `ClusterPolicy verify-image-signature` (mode Audit)
- Cosign keyless verification configuré

### 🚀 Phase 5 — CI/CD Supply Chain (en cours)
- GitHub Actions self-hosted runner dans OKD via ARC
- Trivy gate : 3 images bloquées (nginx, caddy, httpd) → fix libexpat
- Trivy pinné v0.69.3 (post-attaque TeamPCP du 19 mars 2026)
- Pipeline : build → Trivy✅ → push Harbor✅ → Cosign (en cours)

---

## ⚠️ Incident Supply Chain : TeamPCP (19 mars 2026)

> Le 19 mars 2026, le groupe TeamPCP a compromis l'écosystème Trivy d'Aqua Security,
> publiant un binaire malveillant v0.69.4 capable d'exfiltrer les secrets CI/CD.

**Notre pipeline n'a pas été affecté** car :
- Trivy v0.69.3 pinnée (immutable, pré-attaque)
- Téléchargement direct depuis GitHub Releases (pas via `setup-trivy`)

Voir [ADR-004](docs/decisions/ADR-004-trivy-teampcp.md) pour l'analyse complète.

---

## 🔒 Security Controls

| Contrôle | Outil | Status |
|----------|-------|--------|
| Scan CVE images (gate) | Trivy v0.69.3 | ✅ CRITICAL=0 requis |
| Signature images | Cosign 2.4.1 (keyless) | 🔄 En cours |
| Vérification signature | Kyverno ClusterPolicy | ✅ Audit mode |
| Secrets management | HashiCorp Vault + ESO | ✅ |
| OIDC SSO | Keycloak | ✅ |
| Registry privée | Harbor | ✅ |
| GitOps immutable | ArgoCD | ✅ |
| Base image mirrorée | Harbor library/ | ✅ |
| Runner isolé | Pod OKD (namespaced) | ✅ |

---

## 📋 Compliance Mapping

| Framework | Article | Contrôle démontré |
|-----------|---------|-------------------|
| **NIS2** | Art. 21 | Sécurité chaîne d'approvisionnement |
| **DORA** | RTS Art. 8 | Intégrité logicielle + gestion vulnérabilités |
| **SLSA Level 2** | - | Provenance build + signature artefacts |
| **OWASP CI/CD Top 10** | CICD-SEC-3 | Dependency chain abuse |
| **OWASP CI/CD Top 10** | CICD-SEC-8 | Ungoverned 3rd party services |

---

## 🏗️ Infrastructure

| Composant | IP | Description |
|-----------|-----|-------------|
| OKD SNO Master | `192.168.241.10` | Single Node OpenShift |
| Harbor Registry | `192.168.241.20` | Registry + Trivy intégré |
| Tinyproxy | `10.128.0.2:8888` | Proxy internet pour pods OKD |
| Harbor FQDN | `harbor.okd.lab` | DNS interne |

---

## ⚡ Quick Start

```bash
# Accès cluster
export KUBECONFIG=~/work/okd-sno-install/auth/kubeconfig

# Vérifier état ArgoCD
oc get applications -n openshift-operators

# Vérifier runner
oc get runners -n actions-runner-system

# SSH SNO
ssh sno-master
```

---

## 📄 License

MIT — Portfolio project by [Stéphane Seloi / Z3ROX-lab](https://github.com/Z3ROX-lab)

---

> **Note** : Ce projet est un lab de démonstration portfolio illustrant les patterns
> de supply chain security en environnement Cloud Native / OpenShift.
> Il démontre des cas réels rencontrés en production : CVE CRITICAL dans les images de base,
> attaques supply chain sur les outils de sécurité eux-mêmes (TeamPCP/Trivy 2026),
> et les décisions d'architecture qui en découlent.
