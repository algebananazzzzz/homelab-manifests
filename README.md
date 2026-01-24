# Homelab GitOps (ArgoCD + Vault + External Secrets)

## Overview
This repo uses ArgoCD for GitOps, Vault for secret storage, and External Secrets Operator (ESO) for secret delivery. ArgoCD never sees plaintext secrets; ESO is the only component that creates Kubernetes Secrets.

## Prerequisites
- A running Kubernetes cluster.
- ArgoCD installed.

## Deployment (Phase 1 + Phase 2)

Phase 1 bootstrap (creates only Vault + External Secrets apps):

```bash
kubectl apply -k app-of-apps/init/phase_1
```

Wait for ArgoCD to sync those two apps before proceeding.

## Repo Layout
- `app-of-apps/` - root app-of-apps entrypoint (`app-of-apps/root.yaml`)
- `app-of-apps/init/phase_1/` - bootstrap apps (Vault + External Secrets)
- `app-of-apps/apps/` - root app-of-apps (actual apps)
- `apps/secrets/vault/` - Vault Helm app
- `apps/secrets/external-secrets/` - ESO Helm app + ClusterSecretStore
- `apps/secrets/secrets/` - all ExternalSecrets (single place to view created secrets)
- `apps/networking/tailscale/` - Tailscale Operator Helm app
- `apps/observability/prometheus/` - kube-prometheus-stack Helm app
- `apps/observability/loki/` - Loki Helm app
- `apps/observability/fluent-bit/` - Fluent Bit Helm app

## Phased Deployment
Phase 1 (bootstrap infrastructure):
1. Vault setup (runtime + service)
2. External Secrets Operator + ClusterSecretStore

--- MANUAL INTERVENTION REQUIRED ---
3. Human manually writes secrets into Vault using the Vault CLI (outside ArgoCD)

Phase 2 (dependent resources):
4. ExternalSecret resources
5. Tailscale setup (consumes secrets)
6. Rest of applications (Prometheus, Grafana, etc.)

## Sync Order (ArgoCD Sync Waves)
- `vault` app: wave `10`
- `external-secrets` app + ClusterSecretStore: wave `20`
- `secrets` app (ExternalSecrets): wave `30`
- `tailscale` app: wave `40`
- `prometheus` + `loki` apps: wave `50`
- `fluent-bit` app: wave `60`

## Manual Step (Vault CLI)
After Vault is running, initialize/unseal it and write secrets before syncing Phase 2 apps.

Example:

```bash
vault operator init
vault operator unseal
vault login <root-token>
vault secrets enable -path=secret kv-v2
vault kv put secret/tailscale-credentials clientId="..." clientSecret="..."
```

Vault CLI is not installed automatically. Use your local Vault CLI, or exec into the Vault pod:

```bash
kubectl -n vault exec -it vault-0 -- vault operator init
```

Create the Vault token Secret for ESO (namespace `external-secrets`):

```bash
kubectl -n external-secrets create secret generic vault-token \
  --from-literal=token='<vault-token-with-read-access>'
```

Root app-of-apps (actual apps):

```bash
argocd app create homelab-root \
  --repo https://github.com/algebananazzzzz/homelab-manifests \
  --path app-of-apps/apps \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

The declarative form of this app lives at `app-of-apps/root.yaml`.
`app-of-apps/apps` contains the Phase 2 app list; Phase 1 stays manual to allow Vault initialization.

Phase 2 apps have automated sync disabled to enforce the manual intervention point.
Use the ArgoCD UI to sync in order: `secrets` → `tailscale` → observability apps.

Notes on Vault root token:
- The root token is printed by `vault operator init`. Store it securely.
- If you did not save it, you must re-initialize Vault (data loss) or use recovery procedures if configured.

## Accessing Services (Tailscale-only)
These services are exposed privately via Tailscale Ingress:
- ArgoCD: Ingress in `argocd` namespace (`argocd-tailscale`).
- Prometheus: Ingress in `observability` namespace (`prometheus-tailscale`).
- Grafana: Ingress in `observability` namespace (`grafana-tailscale`).
- Vault UI: Ingress in `vault` namespace (`vault-tailscale`).

Use the Tailscale MagicDNS hostname for each ingress (typically `<ingress-name>.<tailnet-domain>`).

## Where Secrets Are Defined
All ExternalSecret manifests live under:
- `apps/secrets/secrets/manifests/`

This is the single folder to audit which Kubernetes Secrets will be created by ESO.
