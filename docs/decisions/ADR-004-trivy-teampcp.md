# ADR-004 — Trivy Version Pinning post-attaque TeamPCP

**Date** : 2026-03-23  
**Statut** : Accepté  
**Criticité** : 🔴 Haute — Supply Chain Attack réelle

## Contexte

Le 19 mars 2026, le groupe `TeamPCP` a compromis l'écosystème Trivy d'Aqua Security dans une attaque supply chain multi-vecteurs.

## Chronologie de l'attaque

```
Fin février 2026
└── Exploitation misconfiguration pull_request_target workflow Trivy
    → Vol PAT GitHub via bot "hackerbot-claw"
    → Prise de contrôle partielle du repo

1 mars 2026
└── Aqua Security divulgue l'incident
    → Rotation credentials (incomplète — attaquant garde un accès résiduel)

19 mars 2026 ~17h43 UTC
├── Force-push 76/77 tags aquasecurity/trivy-action
│   → Tous redirigés vers commits malveillants (infostealer Python)
├── Force-push 7/7 tags aquasecurity/setup-trivy
│   → Même payload malveillant
└── Publication binaire v0.69.4 malveillant
    → Distribué via GitHub Releases, Docker Hub, GHCR, ECR

19 mars 2026 ~20h38 UTC
└── Aqua Security détecte et nettoie
    → Artifacts malveillants supprimés
    → v0.69.3 déclarée dernière version sûre (immutable release)

20 mars 2026
└── Publication advisory GHSA-69fq-xp46-6x23 + IOCs

22 mars 2026
├── Images Docker Hub v0.69.5 et v0.69.6 aussi compromises
├── Worm "CanisterWorm" via tokens npm volés
└── Défacement 44 repos internes Aqua Security ("aquasec-com" org)
```

## Payload malveillant

Le binaire/action compromise :
1. Exécute Trivy légitimement en parallèle (pour ne pas éveiller les soupçons)
2. Scanne les variables d'environnement et fichiers de credentials
3. Chiffre et exfiltre vers `https://scan.aquasecurtiy[.]org` (typosquatting)
4. Fallback : crée repo `tpcp-docs` dans l'org GitHub avec les secrets volés
5. Sur machine développeur : installe persistence via systemd (`sysmon.py`)

## Impact potentiel

Tout pipeline CI/CD utilisant entre le 19-20 mars 2026 :
- `aquasecurity/trivy-action` (tags 0.0.1 → 0.34.2)
- `aquasecurity/setup-trivy`
- Binaire `trivy v0.69.4`

**Doit être considéré comme compromis** — rotation immédiate de tous les secrets.

## Décision

Pinner Trivy à **v0.69.3** — dernière version sûre protégée par immutable releases GitHub.

```yaml
# Dans .github/workflows/supply-chain.yml
- name: Install Trivy v0.69.3 (pinned - last safe version pre-TeamPCP)
  run: |
    curl -sL https://github.com/aquasecurity/trivy/releases/download/v0.69.3/trivy_0.69.3_Linux-64bit.tar.gz | \
    tar -xz -C $HOME/bin trivy
```

## Mesures de sécurité appliquées

```
✅ Version pinnée v0.69.3 (immutable, pré-attaque)
✅ Téléchargement direct depuis GitHub Releases (pas via setup-trivy)
✅ Pas d'utilisation de aquasecurity/trivy-action
✅ Pas d'utilisation de aquasecurity/setup-trivy
✅ Installation dans $HOME/bin (pas de droits root requis)
```

## Règles générales supply chain CI/CD

```yaml
# ❌ UNSAFE — tag mutable
uses: aquasecurity/trivy-action@v0.69.4

# ❌ UNSAFE — latest
uses: aquasecurity/trivy-action@latest

# ✅ SAFE — SHA commit immutable
uses: aquasecurity/trivy-action@57a97c7e7821a5776cebc9bb87c984fa69cba8f1

# ✅ SAFE — download direct version pinnée
curl -sL .../trivy/releases/download/v0.69.3/trivy_0.69.3_Linux-64bit.tar.gz
```

## Références

- [GitHub Advisory GHSA-69fq-xp46-6x23](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23)
- [Wiz Blog — Trivy Compromised by TeamPCP](https://www.wiz.io/blog/trivy-compromised-teampcp-supply-chain-attack)
- [The Hacker News — Trivy Supply Chain Attack](https://thehackernews.com/2026/03/trivy-security-scanner-github-actions.html)
- [StepSecurity — Analysis](https://www.stepsecurity.io/blog/trivy-compromised-a-second-time)
