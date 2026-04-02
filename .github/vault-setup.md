# Vault Setup for Dynamic OC Token Generation

## Prerequisites
- HashiCorp Vault reachable from the self-hosted runner
- OpenShift cluster kubeconfig with enough permissions to create service accounts

---

## 1. Enable the Kubernetes secrets engine

```bash
vault secrets enable -path=kubernetes kubernetes
```

## 2. Configure the engine to talk to your local OC cluster

```bash
vault write kubernetes/config \
  kubernetes_host="https://api.crc.testing:6443" \
  kubernetes_ca_cert=@/path/to/oc-ca.crt \
  service_account_jwt="$(oc serviceaccounts get-token vault-sa -n vault-auth)"
```

## 3. Create a Vault role that generates short-lived OC service account tokens

```bash
vault write kubernetes/roles/github-actions-deployer \
  allowed_kubernetes_namespaces="my-project" \
  service_account_name="github-actions-sa" \
  token_ttl="15m" \
  token_max_ttl="1h"
```

## 4. Create the service account in OC with least-privilege permissions

```bash
oc create serviceaccount github-actions-sa -n my-project
oc create rolebinding github-actions-deployer \
  --clusterrole=edit \
  --serviceaccount=my-project:github-actions-sa \
  -n my-project
```

## 5. Configure Vault JWT auth for GitHub OIDC

```bash
vault auth enable jwt

vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"

vault write auth/jwt/role/github-actions-oc \
  role_type="jwt" \
  bound_audiences="https://vault.internal" \
  user_claim="sub" \
  bound_claims_type="glob" \
  bound_claims='{"sub":"repo:YOUR_ORG/VaultSecretMgnt:*"}' \
  policies="oc-dynamic-token" \
  ttl="15m"
```

## 6. Vault policy

```hcl
# oc-dynamic-token.hcl
path "kubernetes/creds/github-actions-deployer" {
  capabilities = ["create", "update"]
}
```

```bash
vault policy write oc-dynamic-token oc-dynamic-token.hcl
```

---

## GitHub repository variables / secrets required

| Name | Type | Value |
|---|---|---|
| `VAULT_ADDR` | Secret | `https://vault.internal:8200` |
| `OC_API_URL` | Secret | `https://api.crc.testing:6443` |
| `VAULT_ROLE_ID` | Secret | AppRole role ID (fallback only) |
| `VAULT_SECRET_ID` | Secret | AppRole secret ID (fallback only) |
| `VAULT_ROLE` | Variable | `github-actions-oc` |
| `VAULT_JWT_AUDIENCE` | Variable | `https://vault.internal` |
| `OC_K8S_ROLE` | Variable | `github-actions-deployer` |
| `OC_NAMESPACE` | Variable | `my-project` |
| `USE_APPROLE` | Variable | `true` only for non-OIDC local dev |
