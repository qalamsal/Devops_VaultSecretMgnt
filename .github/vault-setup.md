# Vault Setup for Dynamic OC Token Generation

## Environment (verified working)

| Component | Value |
|---|---|
| Vault version | 1.21.4 (dev mode) |
| Vault address | `http://127.0.0.1:8200` |
| Vault root token | `root` |
| OC cluster | CRC v4.21.0 |
| OC API | `https://api.crc.testing:6443` |
| OC namespace | `vault-auth` |
| OC service account | `github-actions-sa` |
| GitHub repo | `qalamsal/Devops_VaultSecretMgnt` |

> **Note:** Dev mode Vault is in-memory only. All data is lost on restart — re-run steps 1–7 after each restart. For production use `brew services start hashicorp/tap/vault` with a persistent config file.

---

## Prerequisites

- Vault installed: `brew install vault`
- OC CLI available via CRC: `eval $(crc oc-env)`
- CRC running: `crc start`
- `KUBECONFIG` set: `export KUBECONFIG=~/.crc/machines/crc/kubeconfig`

---

## Step 1 — Start Vault in dev mode

```bash
vault server -dev -dev-root-token-id="root" -dev-listen-address="0.0.0.0:8200" > /tmp/vault-dev.log 2>&1 &

export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

vault status   # should show Sealed: false
```

---

## Step 2 — Create OC namespace and service accounts

```bash
export KUBECONFIG=~/.crc/machines/crc/kubeconfig
export PATH="/Users/sandeshlamsal/.crc/bin/oc:$PATH"

oc new-project vault-auth

# vault-sa: used by Vault to issue tokens on behalf of github-actions-sa
oc create serviceaccount vault-sa -n vault-auth

# github-actions-sa: identity injected into CI/CD jobs
oc create serviceaccount github-actions-sa -n vault-auth
```

---

## Step 3 — Bind RBAC roles

```bash
# vault-sa needs cluster-wide token review (for Vault k8s auth)
oc create clusterrolebinding vault-tokenreview \
  --clusterrole=system:auth-delegator \
  --serviceaccount=vault-auth:vault-sa

# vault-sa needs to create SA tokens inside vault-auth namespace
oc create role vault-token-creator \
  --verb=create \
  --resource=serviceaccounts/token \
  -n vault-auth

oc create rolebinding vault-token-creator \
  --role=vault-token-creator \
  --serviceaccount=vault-auth:vault-sa \
  -n vault-auth

# github-actions-sa gets edit rights in the working namespace
oc create rolebinding github-actions-deployer \
  --clusterrole=edit \
  --serviceaccount=vault-auth:github-actions-sa \
  -n vault-auth
```

---

## Step 4 — Enable and configure Vault Kubernetes secrets engine

```bash
vault secrets enable -path=kubernetes kubernetes

# Get vault-sa token (1-year TTL for Vault config use)
SA_TOKEN=$(oc create token vault-sa -n vault-auth --duration=8760h)

# Get OC cluster CA cert
OC_CA=$(oc config view --raw \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 --decode)

vault write kubernetes/config \
  kubernetes_host="https://api.crc.testing:6443" \
  kubernetes_ca_cert="$OC_CA" \
  service_account_jwt="$SA_TOKEN"
```

---

## Step 5 — Create Vault role for dynamic token generation

```bash
vault write kubernetes/roles/github-actions-deployer \
  allowed_kubernetes_namespaces="vault-auth" \
  service_account_name="github-actions-sa" \
  token_ttl="15m" \
  token_max_ttl="1h"
```

---

## Step 6 — Configure Vault JWT auth for GitHub OIDC

```bash
vault auth enable jwt

vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"

# bound_claims must be passed as JSON via stdin (-<<EOF syntax)
vault write auth/jwt/role/github-actions-oc \
  role_type="jwt" \
  bound_audiences="https://vault.internal" \
  user_claim="sub" \
  bound_claims_type="glob" \
  policies="oc-dynamic-token" \
  ttl="15m" \
  -<<EOF
{
  "bound_claims": {
    "sub": "repo:qalamsal/Devops_VaultSecretMgnt:*"
  }
}
EOF
```

> **Important:** Passing `bound_claims` as a CLI string (`bound_claims='{"sub":"..."}'`) silently drops the value. Always use the JSON stdin form above.

---

## Step 7 — Write Vault policy

```bash
vault policy write oc-dynamic-token - <<'EOF'
path "kubernetes/creds/github-actions-deployer" {
  capabilities = ["create", "update"]
}
EOF
```

---

## Verify end-to-end

```bash
# Generate a dynamic OC token from Vault
DYNAMIC_TOKEN=$(vault write -force -field=service_account_token \
  kubernetes/creds/github-actions-deployer)

# Login to OC with that token
oc login https://api.crc.testing:6443 --token="$DYNAMIC_TOKEN"

# Should print: system:serviceaccount:vault-auth:github-actions-sa
oc whoami
```

---

## GitHub repository variables / secrets required

| Name | Type | Value |
|---|---|---|
| `VAULT_ADDR` | Secret | `http://127.0.0.1:8200` (dev) |
| `OC_API_URL` | Secret | `https://api.crc.testing:6443` |
| `VAULT_ROLE_ID` | Secret | AppRole role ID (fallback only) |
| `VAULT_SECRET_ID` | Secret | AppRole secret ID (fallback only) |
| `VAULT_ROLE` | Variable | `github-actions-oc` |
| `VAULT_JWT_AUDIENCE` | Variable | `https://vault.internal` |
| `OC_K8S_ROLE` | Variable | `github-actions-deployer` |
| `OC_NAMESPACE` | Variable | `vault-auth` |
| `USE_APPROLE` | Variable | `true` only for non-OIDC local dev |

---

## Known issues & fixes

| Issue | Fix |
|---|---|
| `vault write kubernetes/creds/...` → `Must supply data` | Use `-force` flag: `vault write -force ...` |
| `bound_claims` shows `nil` after write | Pass via JSON stdin, not as CLI string arg |
| `vault-sa cannot create resource serviceaccounts/token` | Add `vault-token-creator` role + rolebinding (Step 3) |
| `oc login --insecure-skip-tls-verify` fails | Use `KUBECONFIG=~/.crc/machines/crc/kubeconfig` directly instead |
| `oc login` → `Internal error 500` on fresh CRC start | Wait ~2 min for OAuth operator to fully start; use kubeconfig in the meantime |
