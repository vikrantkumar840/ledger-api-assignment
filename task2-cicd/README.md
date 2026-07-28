# Task 2 — Secure CI/CD Pipeline & Supply Chain

## Pipeline stages
`secrets-scan (gitleaks)` → `sast (Semgrep)` + `dependency-scan (Trivy fs)` in parallel → `build-scan-sign` (Trivy image scan, Cosign keyless sign, SLSA provenance, in-pipeline verify) → `update-gitops-manifest` (bumps image tag in a separate GitOps repo; ArgoCD auto-syncs).

## Gate policy — what blocks, what warns

| Gate | Tool | Hard block | Warn only |
|---|---|---|---|
| Secrets | gitleaks | Any verified secret match | — (no warn tier; secrets are always a hard stop) |
| SAST | Semgrep (`p/owasp-top-ten`) | High/Critical findings | Medium/Low uploaded to Security tab |
| Dependency/CVE | Trivy (fs) | Critical/High **with a fix available** | Critical/High **with no fix**, if a reviewed waiver exists in `.trivyignore` (CVE id + justification + compensating control + re-scan date). No waiver = still blocks. |
| Image scan | Trivy (image) | Critical/High in final image | Medium/Low |
| Signing | Cosign keyless + Rekor transparency log | Build fails if signing fails | n/a |
| Provenance | SLSA generator (GitHub-hosted, non-forgeable) | n/a — attaches attestation | n/a |

## Why keyless Cosign
No private key material to store or rotate in CI secrets — identity is bound to the GitHub Actions OIDC token and logged to the public Rekor transparency log. `cosign verify` in Kyverno (Task 1) checks the image was signed specifically by *this* repo's workflow (`certificate-identity-regexp`), not just "signed by someone."

## GitOps: why a separate repo
Keeping app manifests in a dedicated `ledger-gitops` repo (rather than the app source repo) means ArgoCD's sync target is never touched by `kubectl apply` from CI directly — CI only commits a tag bump, ArgoCD does the actual apply. This is what makes `selfHeal: true` meaningful: a manual `kubectl edit` in the cluster gets detected as drift and reverted, proving git is the actual source of truth.

## Evidence to capture
- Screenshot: Security tab showing Semgrep + Trivy SARIF findings
- Terminal: `cosign verify` output showing the signing identity
- Terminal: `kubectl scale ... --replicas=10` followed by ArgoCD reverting it (selfHeal)
- Pipeline run link with all gates green
