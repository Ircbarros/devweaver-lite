# rule_security_toolchain

> Security toolchain for CI/CD pipelines, pre-commit, and MLSecOps.  
> Covers: bandit, opengrep, gitleaks, pip-audit, trivy, cosign, CycloneDX SBOM.

---

## Table of Contents

1. [bandit — Python SAST](#1-bandit--python-sast)
2. [opengrep — Multi-Language SAST](#2-opengrep--multi-language-sast)
3. [gitleaks — Secret Scanning](#3-gitleaks--secret-scanning)
4. [pip-audit — Dependency CVE Scanning](#4-pip-audit--dependency-cve-scanning)
5. [trivy — Container Image Scanning](#5-trivy--container-image-scanning)
6. [cosign — Image & Artifact Signing](#6-cosign--image--artifact-signing)
7. [CycloneDX SBOM Generation](#7-cyclonedx-sbom-generation)
8. [MLSecOps Practices](#8-mlsecops-practices)
9. [GitHub Actions Integration](#9-github-actions-integration)
10. [Stable Versions](#10-stable-versions)

---

## 1. bandit — Python SAST

```toml
# pyproject.toml
[tool.bandit]
skips = ["B101"]          # B101 = assert used; acceptable in test code
tests = [
  "B105", "B106", "B107", # Hardcoded password/secret
  "B201", "B202",         # Flask/FastAPI debug mode on
  "B301", "B302",         # Pickle deserialization
  "B401", "B402",         # Import of unsafe modules
  "B501", "B502", "B503", # SSL/TLS issues
  "B601", "B602",         # Shell injection
  "B608",                 # SQL injection
]
```

**CI command**: `bandit -r src/ -c pyproject.toml --exit-zero-on-suppressed`  
**Block merge on**: HIGH severity or MEDIUM + HIGH confidence findings.

---

## 2. opengrep — Multi-Language SAST

```yaml
# opengrep config (run in CI)
rules:
  - python.lang.security.insecure-hash-algorithms.insecure-hash-algorithm
  - python.jwt.security.unverified-jwt-decode.python-jwt-decode-without-verify
  - python.requests.security.no-auth-over-http.no-auth-over-http
  - javascript.lang.security.detected-non-literal-require.detected-non-literal-require
```

**CI command**: `opengrep --config=opengrep.yaml --output=json src/`

---

## 3. gitleaks — Secret Scanning

**Pre-commit hook setup**:
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.20.1
    hooks:
      - id: gitleaks
```

**Developer onboarding command**: `pre-commit install`  
**CI scan**: `gitleaks detect --source . --report-format json --report-path gitleaks-report.json`  
Block merge on: any finding (zero-tolerance policy for secrets in git history).

---

## 4. pip-audit — Dependency CVE Scanning

```bash
pip-audit --requirement requirements.txt --output json --output-file pip-audit.json
```

- Run after every `requirements.txt` change and on weekly scheduled CI run.
- Block deployment if any CRITICAL CVE found; HIGH severity → create GitHub issue.

---

## 5. trivy — Container Image Scanning

```bash
# Scan built image before push
trivy image --exit-code 1 --severity CRITICAL,HIGH ghcr.io/myorg/myapp:$VERSION

# Scan IaC (Dockerfile, Compose, K8s)
trivy config --exit-code 1 .
```

**Stable version**: trivy `0.50.x`

---

## 6. cosign — Image & Artifact Signing

```bash
# Sign image after push (CI, keyless via OIDC)
cosign sign --yes ghcr.io/myorg/myapp:$VERSION

# Verify before deployment
cosign verify \
  --certificate-identity="https://github.com/myorg/myapp/.github/workflows/ci.yml@refs/heads/main" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/myapp:$VERSION
```

---

## 7. CycloneDX SBOM Generation

**Python (cyclonedx-bom)**:
```bash
pip install cyclonedx-bom
cyclonedx-py requirements requirements.txt -o sbom-python.json --format json
```

**Node.js (@cyclonedx/cyclonedx-npm)**:
```bash
npx @cyclonedx/cyclonedx-npm --output-file sbom-node.json --output-format JSON
```

**Upload to Garage after generation**:
```python
await s3.put_object(
    Bucket="aibuilder-prod-sbom",
    Key=f"sbom-{VERSION}-{datetime.utcnow().isoformat()}.json",
    Body=sbom_json,
    ServerSideEncryption="AES256"
)
```

---

## 8. MLSecOps Practices

- **Data poisoning detection**: Entropy check on documents before embedding (see `rule_owasp_llm_top10.md`).
- **Model artefact SBOM**: Include model name, version, hash (SHA-256), training data source in CycloneDX `components`.
- **Signed models**: Upload model weights to Garage with `cosign` attestation.
- **Drift monitoring**: Track embedding distribution shifts in Grafana; alert via ntfy on z-score > 3.
- **Red-teaming**: Run Garak against LLM endpoint monthly; log results to LangFuse.
- **ART (Adversarial Robustness Toolbox)**: Use for evasion/poisoning tests in CI pipeline.

---

## 9. GitHub Actions Integration

```yaml
# .github/workflows/security.yml (excerpt)
jobs:
  security:
    steps:
      - uses: actions/checkout@v4        # Fetch full history for gitleaks
        with: { fetch-depth: 0 }
      - run: pip install bandit pip-audit cyclonedx-bom
      - run: bandit -r src/ -c pyproject.toml
      - run: pip-audit -r requirements.txt
      - run: cyclonedx-py requirements requirements.txt -o sbom.json --format json
      - run: trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE
      - run: cosign sign --yes $IMAGE
```

---

## 10. Stable Versions

| Tool | Version | Notes |
|---|---|---|
| `bandit` | `1.7.x` | ✅ Stable |
| `opengrep` | `1.x` | ✅ Stable (fork of semgrep OSS) |
| `gitleaks` | `8.20.x` | ✅ Stable |
| `pip-audit` | `2.7.x` | ✅ Stable |
| `trivy` | `0.50.x` | ✅ Stable |
| `cosign` | `2.x` | ✅ Stable |
| `cyclonedx-bom` | `4.x` | ✅ Stable |
| `garak` | `0.9.x` | ⚠️ Beta |
