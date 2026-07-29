# Dodo Payments — Security & DevOps Technical Assessment

**Candidate:** Vikrant Kumar
**Repo:** https://github.com/vikrantkumar840/ledger-api-assignment
**Environment:** Local `kind` cluster + GitHub Actions + GHCR. No cloud account used (AWS EC2 was used only as a personal dev workstation to run `kind`/`kubectl`/Docker — no cloud-managed services were part of the assessed architecture).

## Repo layout

| Folder | Task | Status |
|---|---|---|
| `app/` | The `ledger-api` Flask application (starter code, hardened Dockerfile) | Working, deployed |
| `task1-hardening/` | Deploy & harden `ledger-api` | Done — see [README](task1-hardening/README.md) |
| `task2-cicd/` | Secure CI/CD + supply chain | Done — see [README](task2-cicd/README.md) |
| `task3-mesh/` | Istio zero-trust mesh | Done — see [README](task3-mesh/README.md) |
| `task4-recon-pentest/` | Recon + pentest | Done — [attack-surface-report.md](task4-recon-pentest/attack-surface-report.md), [pentest-report.md](task4-recon-pentest/pentest-report.md) |

## What's actually running, end to end

- `ledger-api` and `notification-svc` deployed in the `ledger` namespace, both fully hardened (non-root, read-only rootfs, all capabilities dropped, seccomp `RuntimeDefault`, resource limits, liveness/readiness probes) and both inside the Istio mesh (`2/2` pods, sidecar injected via CNI).
- Pod Security Standards (`restricted`) enforced at the namespace level, plus 5 Kyverno ClusterPolicies (reject root, `:latest` tags, missing cap-drop, writable rootfs, unsigned images) — all independently verified to actually block a real insecure deployment (see Task 1 README for the exact denial output).
- Secrets managed via Sealed Secrets — a real encrypted `SealedSecret` is committed to git; the plaintext value only ever exists inside the cluster after the controller decrypts it, and this was verified by round-tripping the value through `kubectl get secret`.
- A full GitHub Actions pipeline: `gitleaks` (secrets) → `Semgrep` (SAST) → `Trivy` (dependency + image CVE scan, with a documented, dated, per-CVE waiver for unfixed Debian base-image packages) → Docker build/push to GHCR → Cosign keyless signing → in-pipeline signature verification → automated commit of the new image digest back into this same repo (single-repo GitOps, per assessment scope decision).
- ArgoCD watching this repo, with `prune: true` / `selfHeal: true` — genuinely exercised during the assessment, not just staged: a real image-reference bug in `notification-svc` was caught by Kyverno's admission webhook mid-sync, diagnosed from ArgoCD's own event log, fixed, and re-synced successfully. That failure-and-recovery cycle is itself part of the evidence for how GitOps drift detection behaves under real conditions.
- Istio installed with **CNI-based sidecar injection** — the default (init-container-based) injection mode was tried first and failed outright, because Istio's default `istio-init` container requires `NET_ADMIN`/`NET_RAW` capabilities and root, which directly violates the `restricted` PSS enforced on the namespace in Task 1. This was diagnosed from the actual PodSecurity admission error, not assumed, and resolved by switching to the CNI plugin, which performs traffic redirection via a privileged DaemonSet in `kube-system` instead of inside each application pod — keeping every workload in `ledger` fully PSS-`restricted`-compliant with zero exceptions.
- mTLS `STRICT` confirmed via `istioctl x describe`. A default-deny `AuthorizationPolicy` plus explicit SPIFFE-identity-based allow rules confirmed working with a live test: an unauthorized ServiceAccount identity received `403`, the explicitly allowed `notification-svc` identity received `200`.
- A Kubernetes `NetworkPolicy` (default-deny + explicit allows) layered underneath Istio for defense-in-depth — see Task 3 README for exactly what each layer catches that the other doesn't.
- Task 4: three real, independently-reproduced findings against the actual `ledger-api` application (not simulated) — an unauthenticated cardholder-data-exposure endpoint, a working SSRF (proven against external targets; proven *blocked* against internal mesh targets specifically by the Task 3 AuthorizationPolicy, and blocked against cloud metadata by AWS's own IMDSv2 — neither block is due to the application itself), and an unsafe YAML deserialization pattern whose real-world exploitability was checked directly against server logs rather than assumed from the code alone (it turned out to be non-exploitable to RCE on the currently pinned PyYAML version, which is a more accurate and more defensible finding than assuming worst-case severity).
- Passive recon against `dodopayments.tech` via crt.sh enumerated 113 subdomains; a status-code-only liveness pass (no active scanning) surfaced a genuinely notable finding: a `-prod`-named ClickHouse analytics database instance responding `200` on a public hostname, which is a known real-world breach pattern for unauthenticated ClickHouse deployments.

## Design decisions & trade-offs

- **Secrets**: Sealed Secrets over SOPS+age/External Secrets — no external KMS/vault dependency, fits the "fully local" constraint.
- **Admission control**: Kyverno over Gatekeeper — YAML-native policies, faster to write and review correctly under time pressure.
- **GitOps repo layout**: single-repo (app source + manifests + CI + GitOps target all in one repo) rather than a separate `ledger-gitops` repo, as a deliberate scope trade-off to move faster within the assessment's timeframe. The trade-off: CI's `update-manifest` job commits back into the same repo it was triggered from, which requires careful `git pull --rebase` discipline to avoid push races — this was hit repeatedly during development and is a known sharp edge of this layout, noted here rather than hidden.
- **Sidecar injection mode**: CNI plugin over default init-container injection — not a stylistic choice but a hard requirement once `restricted` PSS was already enforced in Task 1; documented above and in the Task 3 README as a genuine cross-task architectural conflict that had to be resolved, not avoided.
- **CVE-with-no-fix policy**: implemented exactly as designed — Debian base-image CVEs with no upstream fix (confirmed via Trivy's empty "Fixed Version" column) are waived via a dated, justified, per-CVE `.trivyignore` entry with a documented re-scan date and listed compensating controls, rather than a blanket `ignore-unfixed`. Any *future* CVE with a real fix available still hard-blocks the pipeline.

## What I'd do with more time

- **SSRF egress hardening**: the current `NetworkPolicy` only restricts *ingress*. An egress rule explicitly blocking the cloud metadata address range (`169.254.169.254/32`) and RFC1918 private ranges from the `ledger-api` pod would close the SSRF finding's internal-pivot risk at the network layer too, as defense-in-depth alongside the application-level allowlist fix.
- **`/import` and `/transactions` app-code fixes**: this assessment is infra-focused, so the three Task 4 findings were deliberately left unpatched in the running application to preserve them as genuine, live, reproducible findings for the pentest report — in a real engagement these would be fixed immediately (one-line `yaml.safe_load()` swap; add auth + PAN masking).
- **`notification-svc` production-readiness**: swapped to `nginx-unprivileged` mid-assessment to satisfy the `restricted` securityContext; a real production neighbour service would need its own equivalent hardening pass (this was a stand-in service to satisfy the "at least one neighbour" requirement, not a real business service).
- **Gatekeeper alongside Kyverno**: for direct policy-engine comparison, as originally considered.
- **Argo Rollouts** for real canary/blue-green traffic shifting, instead of the static weighted `VirtualService` config currently in `task3-mesh/istio/gateway-and-canary.yaml`.
- **Chained Task 4 finding, fully weaponized**: the SSRF-to-future-auth-bypass chain is described conceptually in the pentest report but wasn't executed against a real second auth layer, since `/transactions` currently has no auth to bypass in the first place.
- **Broader Part A recon**: subfinder/amass/httpx tooling wasn't installed in this environment; crt.sh + manual `curl` status checks covered the same ground faster given time constraints, but dedicated tooling would surface additional passive signal (technology fingerprinting, TLS posture via testssl.sh) not captured here.

## Submission notes

Per assignment instructions: this README summarizes and links each task; the Task 4 pentest report is also provided as a standalone file (`task4-recon-pentest/pentest-report.md`) as required. Screenshots/terminal evidence for each control listed above were captured during the live assessment session and are available on request if not already included in the submission package.
