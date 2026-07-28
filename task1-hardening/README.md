# Task 1 — Deploy & Harden `ledger-api`

## What's here
- `kind-config.yaml` — local cluster with ingress port mappings
- `manifests/namespace.yaml` — `ledger` namespace with **Pod Security Standards: restricted** enforced (bonus)
- `manifests/serviceaccount-rbac.yaml` — dedicated SAs (no default SA, token automount off) + least-privilege Role/RoleBinding + **bonus** developer/operator/admin persona Roles
- `manifests/configmap.yaml`, `sealed-secret-example.yaml` — app config; secrets sealed, plaintext never in git
- `manifests/deployment-ledger-api.yaml`, `deployment-neighbour.yaml` — hardened workloads (non-root, RO rootfs, all caps dropped, seccomp RuntimeDefault, resource requests/limits, liveness+readiness probes)
- `manifests/service.yaml`, `ingress.yaml` — networking
- `manifests/kyverno-policies.yaml` — admission guardrails: reject root, `:latest`, unsigned images, missing cap-drop, writable rootfs
- `manifests/original-insecure-deployment.yaml` — the "before" state, used only to **prove** the Kyverno policies reject it (bonus)

## Secrets: why Sealed Secrets
No cloud KMS needed — the controller lives in-cluster, `kubeseal` encrypts client-side, and only ciphertext is committed. Rotation = reseal + re-apply. (SOPS+age was the runner-up; External Secrets was overkill without a cloud secret backend.)

## Proving the guardrails work (bonus)
```bash
# Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno-system --create-namespace
kubectl apply -f manifests/kyverno-policies.yaml

# Attempt the insecure deployment — expect denial
kubectl apply -f manifests/original-insecure-deployment.yaml
# -> Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:
#    policy disallow-root-user/check-runasnonroot ... (screenshot this output)

# Apply the hardened version — expect success
kubectl apply -f manifests/deployment-ledger-api.yaml
```

## RBAC scoping rationale
`ledger-api-sa` can only `get` its own named ConfigMap — nothing else, no token automount since the app never talks to the K8s API. This limits blast radius if the container is ever compromised: an attacker inside the pod cannot list secrets, other configmaps, or any cluster resource.

