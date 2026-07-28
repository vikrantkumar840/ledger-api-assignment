# Task 3 — Service Mesh & Zero-Trust (Istio)

## Setup
```bash
istioctl install --set profile=default -y
kubectl label namespace ledger istio-injection=enabled
kubectl rollout restart deployment -n ledger    # inject sidecars
kubectl apply -f istio/
```

## Certificate issuance & rotation (write this up, don't just paste)
Istiod runs a built-in CA (or can delegate to an external one via `cacerts`). On sidecar
startup, `istio-agent` generates a key pair and sends a CSR over the SDS API to Istiod,
which signs a short-lived (default 24h) X.509 cert encoding the workload's **SPIFFE
identity** (`spiffe://cluster.local/ns/ledger/sa/ledger-api-sa`) as the SAN. The agent
auto-rotates the cert before expiry with no pod restart. The root of trust is Istiod's
self-signed root CA (or your org's intermediate CA if you plug one in via `cacerts`);
every workload cert chains up to that single root, which is what lets any two sidecars
mutually verify identity without a shared secret.

## Evidence to capture
1. `istioctl authn tls-check ledger-api.ledger.svc.cluster.local` — all STRICT
2. Plaintext curl from a non-mesh pod → refused
3. `notification-svc` → `ledger-api` → 200 OK (explicit allow)
4. An `unauthorised-svc` (own SA, no AuthorizationPolicy rule) → `ledger-api` → RBAC 403
5. `kubectl apply --dry-run` showing NetworkPolicy blocking a pod with no matching label even with mTLS off (defense-in-depth proof)

## PCI CDE scope note (bonus)
The `ledger` namespace *is* the cardholder data environment boundary. Anything with an mTLS-verified SPIFFE identity issued by the in-cluster root and an explicit AuthorizationPolicy allow is in-scope and audited; anything outside the mesh (no sidecar, no cert) cannot reach `ledger-api` at all thanks to the NetworkPolicy default-deny — meaning it never enters CDE scope in the first place. This mesh boundary is the technical control you'd point a PCI QSA to as evidence of network segmentation (Req 1 / Req 4 of PCI DSS v4.0).
