# Dodo Payments — Security & DevOps Technical Assessment

Candidate submission. Environment: local `kind` cluster + GitHub Actions + GHCR. No cloud account used.

## Repo layout
| Folder | Task | Status |
|---|---|---|
| `task1-hardening/` | Deploy & harden `ledger-api` | Manifests + Kyverno guardrails |
| `task2-cicd/` | Secure CI/CD + supply chain | GitHub Actions pipeline + ArgoCD app |
| `task3-mesh/` | Istio zero-trust mesh | mTLS STRICT + AuthorizationPolicy + NetworkPolicy |
| `task4-recon-pentest/` | Recon + pentest | Report templates + tool cheat-sheet |

## How to reproduce (quick path)
```bash
# 1. Spin up local cluster
kind create cluster --name dodo --config task1-hardening/kind-config.yaml

# 2. Namespace + PSS + guardrails
kubectl apply -f task1-hardening/manifests/namespace.yaml
kubectl apply -f task1-hardening/manifests/serviceaccount-rbac.yaml
kubectl apply -f task1-hardening/manifests/kyverno-policies.yaml   # requires Kyverno installed first

# 3. App
kubectl apply -f task1-hardening/manifests/configmap.yaml
kubectl apply -f task1-hardening/manifests/deployment-ledger-api.yaml
kubectl apply -f task1-hardening/manifests/deployment-neighbour.yaml
kubectl apply -f task1-hardening/manifests/service.yaml
kubectl apply -f task1-hardening/manifests/ingress.yaml

# 4. Mesh (after istioctl install)
kubectl label namespace ledger istio-injection=enabled
kubectl apply -f task3-mesh/istio/

# 5. GitOps
kubectl apply -f task2-cicd/argocd/application.yaml
```

## Design decisions & trade-offs (short version)
- **Secrets**: chose **Sealed Secrets** over SOPS+age/External Secrets because it needs zero external infra (no cloud KMS, no vault) — fits the "fully local" constraint, and the controller + kubeseal CLI are enough. Trade-off: secret is still tied to one cluster's private key, so cross-cluster/DR needs re-sealing.
- **Admission control**: chose **Kyverno** over Gatekeeper — YAML-native policies (no Rego) are faster to write and review correctly under time pressure, which matters for both velocity and auditability.
- **GitOps**: **ArgoCD** for the visual diff/drift view, which is easier to screenshot as evidence of self-heal than Flux's CLI-only output.
- **CVE-with-no-fix policy**: block on Critical/High with a fix available; High/Critical with *no* fix available get a time-boxed waiver (documented in `task2-cicd/README.md`) requiring compensating control (network policy / WAF rule) + a tracked re-scan date; Medium/Low warn only.

## What I'd do with more time
- Full Gatekeeper ConstraintTemplate library alongside Kyverno for comparison.
- Canary rollout automated via Argo Rollouts instead of manual VirtualService weight edits.
- Chain Task 4 findings into a full attack path + retest section (see `task4-recon-pentest/README.md` for the plan).
