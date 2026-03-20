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
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub.com                                │
│   Z3ROX-lab/okd-sno-supply-chain                                │
│              │  push / PR                                        │
│              ▼                                                   │
│   .github/workflows/supply-chain.yml                            │
└──────────────┬──────────────────────────────────────────────────┘
               │  trigger
               ▼
┌─────────────────────────────────────────────────────────────────┐
│              OKD SNO 4.15 (192.168.241.10)                      │
│                                                                  │
│  ┌─────────────────┐     ┌──────────────────────────────────┐  │
│  │  GitHub Actions │     │           ArgoCD                 │  │
│  │  Self-hosted    │────▶│  root-app (App of Apps)          │  │
│  │  Runner (pod)   │     │  ├── keycloak    ✅               │  │
│  └────────┬────────┘     │  ├── vault       ✅               │  │
│           │              │  ├── grafana     ✅               │  │
│      Pipeline:           │  ├── loki        ✅               │  │
│      1. docker build     │  ├── eso         ✅               │  │
│      2. trivy scan       │  ├── kyverno     ✅               │  │
│      3. cosign sign      │  └── app-demo    🚀               │  │
│      4. harbor push      └──────────────────────────────────┘  │
│           │                                                      │
└───────────┼──────────────────────────────────────────────────────┘
            │ push signed image
            ▼
┌─────────────────────────────────────────────────────────────────┐
│              Harbor Registry (192.168.241.20)                   │
│              harbor.okd.lab                                      │
│                                                                  │
│  Projects:                                                       │
│  ├── library/         ← images mirrorées (charts, base images)  │
│  ├── supply-chain/    ← images buildées + signées Cosign        │
│  └── signed/          ← images validées par Kyverno             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛡️ Security Pipeline

```
Code Push
    │
    ▼
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│ docker build │───▶│ trivy scan   │───▶│ cosign sign  │
│             │    │ CVE check    │    │ keyless sig  │
│ Dockerfile  │    │ CRITICAL=0   │    │ Sigstore     │
└─────────────┘    └──────────────┘    └──────┬───────┘
                                              │
                                              ▼
                                    ┌──────────────────┐
                                    │  harbor push     │
                                    │  harbor.okd.lab  │
                                    │  /supply-chain   │
                                    └────────┬─────────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
                                    │  ArgoCD sync     │
                                    │  app-demo        │
                                    │  auto-deploy     │
                                    └────────┬─────────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
                                    │  Kyverno policy  │
                                    │  verify-image-   │
                                    │  signature ✅    │
                                    └──────────────────┘
```

---

## 📦 Stack

| Composant | Version | Rôle |
|-----------|---------|------|
| **OKD SNO** | 4.15 | Single Node OpenShift — cluster de démo |
| **GitHub Actions** | - | CI/CD orchestrator (SaaS) |
| **Self-hosted Runner** | latest | Runner pod dans OKD |
| **Harbor** | 2.x | Registry OCI privée + scan Trivy intégré |
| **Trivy** | latest | Scan CVE des images (CRITICAL gate) |
| **Cosign** | 2.x | Signature keyless via Sigstore |
| **ArgoCD** | Community 0.17.0 | GitOps — déploiement continu |
| **Kyverno** | 1.12.0 | Policy-as-Code — vérification signature |
| **HashiCorp Vault** | 0.28.0 | Secrets management |
| **Keycloak** | 24.x | IAM / OIDC SSO |
| **Loki + Grafana** | latest | Observabilité stack |

---

## 🗂️ Repository Structure

```
okd-sno-supply-chain/
├── .github/
│   └── workflows/
│       └── supply-chain.yml       # Pipeline principal
├── argocd/
│   └── applications/
│       ├── gitlab-runner.yaml     # (remplacé par actions-runner)
│       ├── actions-runner.yaml    # Self-hosted runner deployment
│       └── app-demo.yaml          # Application de démo
├── manifests/
│   ├── argocd/
│   │   ├── 01-repo-helm-hashicorp.yaml
│   │   ├── 02-repo-helm-actions-runner.yaml
│   │   └── 03-repo-helm-harbor.yaml
│   ├── actions-runner/
│   │   └── values.yaml            # Helm values runner
│   ├── kyverno/
│   │   └── verify-image-policy.yaml
│   └── vault/
│       └── values.yaml
├── app/
│   ├── Dockerfile                 # Application de démo
│   └── src/
│       └── index.html
├── docs/
│   ├── architecture.md
│   ├── supply-chain-security.md
│   └── screenshots/
└── README.md
```

---

## 🚀 Phases du Projet

### ✅ Phase 1 — Bootstrap OKD SNO
- Installation OKD 4.15 agent-based sur GEEKOM A6 (32GB DDR5)
- ArgoCD Community Operator v0.17.0
- App of Apps pattern (`root-app`)
- Proxy tinyproxy (`10.128.0.2:8888`) pour accès internet

### ✅ Phase 2a — IAM : Keycloak OIDC SSO
- Keycloak déployé via ArgoCD (Helm)
- OIDC SSO configuré pour OKD console
- Secrets migrés vers SealedSecrets

### ✅ Phase 2b — Secrets : HashiCorp Vault
- Vault v0.28.0 dev mode via ArgoCD
- External Secrets Operator (ESO) intégré
- Vault Agent Injector configuré

### ✅ Phase 3 — Observabilité
- Loki + Grafana déployés via ArgoCD
- Dashboards OKD + pipeline métriques

### ✅ Phase 4 — Policy-as-Code : Kyverno
- Kyverno v1.12.0 installé (images Harbor)
- `ClusterPolicy verify-image-signature` (mode Audit)
- Cosign keyless verification configuré

### 🚀 Phase 5 — CI/CD Supply Chain (en cours)
- GitHub Actions self-hosted runner dans OKD
- Pipeline : build → Trivy → Cosign → Harbor → ArgoCD
- Kyverno enforcement mode

---

## ⚡ Quick Start

### Prérequis

```bash
# Accès cluster
export KUBECONFIG=~/work/okd-sno-install/auth/kubeconfig

# Vérifier l'état ArgoCD
oc get applications -n argocd

# SSH SNO master
ssh sno-master  # 192.168.241.10
```

### Déployer le Runner

```bash
# 1. Créer le secret GitHub Runner token
oc create secret generic github-runner-secret \
  --from-literal=github_token=<RUNNER_TOKEN> \
  -n actions-runner-system

# 2. Appliquer via ArgoCD
oc apply -f argocd/applications/actions-runner.yaml

# 3. Vérifier
oc get pods -n actions-runner-system
```

### Déclencher le pipeline

```bash
git add .
git commit -m "feat: trigger supply chain pipeline"
git push origin main
# → GitHub Actions déclenche automatiquement
```

---

## 🔒 Security Controls

| Contrôle | Outil | Status |
|----------|-------|--------|
| Scan CVE images | Trivy | ✅ Gate CRITICAL=0 |
| Signature images | Cosign (keyless) | ✅ Sigstore |
| Vérification signature | Kyverno ClusterPolicy | ✅ Audit → Enforce |
| Secrets management | HashiCorp Vault + ESO | ✅ |
| OIDC SSO | Keycloak | ✅ |
| Registry privée | Harbor | ✅ |
| GitOps immutable | ArgoCD | ✅ |
| Network policies | OKD NetworkPolicy | 🔄 |

---

## 📋 Compliance Mapping

Ce projet démontre des contrôles alignés sur :

- **NIS2** — Article 21 : sécurité de la chaîne d'approvisionnement
- **DORA** — RTS : intégrité des logiciels et gestion des vulnérabilités  
- **SLSA Level 2** — Provenance build + signature artefacts
- **OWASP Top 10 CI/CD** — CICD-SEC-8 : Ungoverned usage of 3rd party services

---

## 🏗️ Infrastructure

| Composant | IP | Description |
|-----------|-----|-------------|
| OKD SNO Master | `192.168.241.10` | Single Node OpenShift |
| Harbor Registry | `192.168.241.20` | Registry + Trivy |
| Tinyproxy | `10.128.0.2:8888` | Proxy internet cluster |
| Harbor FQDN | `harbor.okd.lab` | DNS interne |

---

## 📄 License

MIT — Portfolio project by [Stéphane Seloi](https://github.com/Z3ROX-lab)

---

> **Note** : Ce projet est un lab de démonstration portfolio illustrant les patterns  
> de supply chain security en environnement Cloud Native / OpenShift.
