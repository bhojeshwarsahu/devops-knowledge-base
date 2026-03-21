# GitHub Actions + HashiCorp Vault — OIDC Authentication

> **Repository:** `bhojeshwarsahu/my-pvt-project`  
> Zero standing credentials — no long-lived tokens stored anywhere.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Vault Configuration](#3-vault-configuration)
4. [bound_claims Patterns](#4-bound_claims-patterns)
5. [GitHub Actions Workflow](#5-github-actions-workflow)
6. [AWS Security Group](#6-aws-security-group)
7. [Troubleshooting](#7-troubleshooting)
8. [Production Checklist](#8-production-checklist)

---

## 1. Overview

### Why OIDC over static tokens?

| | Static Token (old way) | OIDC (this setup) |
|---|---|---|
| **Storage** | Stored in GitHub Secrets | No credentials stored anywhere |
| **Lifetime** | Long-lived, manual rotation | Auto-expires after 15 minutes |
| **If leaked** | Permanent access until rotated | Useless after TTL |
| **Audit** | No context on usage | Claims include repo, branch, environment |

### How it works in one line

> GitHub mints a signed JWT proving *who the job is* → Vault verifies it → issues a short-lived token → job reads secrets → token expires.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Channel 1 — GitHub internal (no internet)              │
│                                                         │
│  GH Runner ──── request JWT ────► GitHub OIDC Issuer    │
│           ◄─── signed JWT ───────                       │
│   JWT contains: sub, repo, branch, environment          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Channel 2 — network (HTTPS)                            │
│                                                         │
│  GH Runner ── POST /v1/auth/jwt/login ──► Vault         │
│                                           │             │
│                               fetch JWKS ▼             │
│                          GitHub public keys             │
│                          (token.actions.githubusercontent.com)
│                                           │             │
│           ◄── Vault token (15m TTL) ──────             │
│                                                         │
│  GH Runner ── GET /v1/secret/data/myapp ──► Vault       │
│           ◄── secrets ────────────────────              │
└─────────────────────────────────────────────────────────┘
```

> **Note:** Your repo being **private** has zero effect on this flow.  
> Private repo = your code is private.  
> JWKS = always public.  
> These are completely separate things. Vault never reads your code.

---

## 3. Vault Configuration

### 3.1 Enable JWT auth method

```bash
vault auth enable jwt
```

### 3.2 Configure JWT auth — trust GitHub as OIDC issuer

```bash
vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"
```

### 3.3 Create the role

> **Use the JSON file method** — avoids shell escaping issues with `bound_claims`.

```bash
cat > /tmp/role.json <<'EOF'
{
  "role_type": "jwt",
  "user_claim": "sub",
  "bound_audiences": ["vault"],
  "bound_claims_type": "glob",
  "bound_claims": {
    "sub": "repo:bhojeshwarsahu/my-pvt-project:ref:refs/heads/main"
  },
  "policies": "readonly",
  "ttl": "15m"
}
EOF

vault write auth/jwt/role/github-actions @/tmp/role.json
```

**Role fields explained:**

| Field | Value | Purpose |
|---|---|---|
| `role_type` | `jwt` | Plain JWT login — not full browser OIDC flow |
| `user_claim` | `sub` | Used as identity in Vault audit logs |
| `bound_audiences` | `["vault"]` | Must match `jwtGithubAudience` in workflow |
| `bound_claims_type` | `glob` | Enables `*` wildcard matching in claim values |
| `bound_claims.sub` | `repo:org/repo:ref:...` | Trust gate — only this repo + branch allowed |
| `policies` | `readonly` | Vault policy attached to the issued token |
| `ttl` | `15m` | Token auto-expires after 15 minutes |

**Verify the role was saved correctly:**

```bash
vault read auth/jwt/role/github-actions
```

### 3.4 Create the policy

```bash
vault policy write readonly - <<'EOF'
# Read secrets from ANY secrets engine
path "+/data/*" {
  capabilities = ["read"]
}

# List secret paths
path "+/metadata/*" {
  capabilities = ["list"]
}

# Check own token
path "auth/token/lookup-self" {
  capabilities = ["read"]
}

# Renew own token
path "auth/token/renew-self" {
  capabilities = ["update"]
}
EOF
```

### 3.5 Store secrets

```bash
# Enable KV v2 secrets engine
vault secrets enable -path=secret kv-v2

# Write secrets
vault kv put secret/myapp/config \
  api_key="your-api-key" \
  db_password="your-db-password"

# Verify
vault kv get secret/myapp/config
```

---

## 4. bound_claims Patterns

`bound_claims` is your entire security boundary. Multiple fields use **AND logic** — every claim listed must match.

### What the GitHub JWT actually contains

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "sub": "repo:bhojeshwarsahu/my-pvt-project:ref:refs/heads/main",
  "aud": "vault",
  "repository": "bhojeshwarsahu/my-pvt-project",
  "repository_owner": "bhojeshwarsahu",
  "ref": "refs/heads/main",
  "ref_type": "branch",
  "environment": "production",
  "job_workflow_ref": "bhojeshwarsahu/my-pvt-project/.github/workflows/deploy.yml@refs/heads/main",
  "event_name": "push"
}
```

### Pattern 1 — Single repo, main branch only ✅ (this setup)

```json
"bound_claims": {
  "sub": "repo:bhojeshwarsahu/my-pvt-project:ref:refs/heads/main"
}
```

### Pattern 2 — Production environment only

```json
"bound_claims": {
  "sub": "repo:bhojeshwarsahu/my-pvt-project:environment:production",
  "repository": "bhojeshwarsahu/my-pvt-project"
}
```

### Pattern 3 — All repos under org, any branch, never forks

```json
"bound_claims": {
  "sub": "repo:bhojeshwarsahu/*:ref:refs/heads/main",
  "repository_owner": "bhojeshwarsahu"
}
```

### Pattern 4 — Any branch (read-only secrets, less strict)

```json
"bound_claims": {
  "sub": "repo:bhojeshwarsahu/my-pvt-project:*"
}
```

> **Tip:** For a single repo, `sub` alone is sufficient — it already encodes repo name and branch.  
> Adding `repository` as a second claim only adds value when you use a glob on `sub`.

---

## 5. GitHub Actions Workflow

Create `.github/workflows/vault-deploy.yml`:

```yaml
name: Deploy (Vault OIDC)

on:
  push:
    branches: [main]
  workflow_dispatch:           # allows manual trigger from GitHub UI

permissions:
  id-token: write              # REQUIRED — allows JWT generation
  contents: read               # repo checkout
  actions: read                # audit tools

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true     # cancel older run if new push comes in

env:
  VAULT_URL: https://44.215.126.96:8200
  VAULT_ROLE: github-actions

jobs:
  read-secrets:
    name: Fetch secrets and deploy
    runs-on: ubuntu-latest
    timeout-minutes: 10        # kill job if it hangs

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 1       # shallow clone — faster, no history needed

      - name: Fetch secrets from Vault
        id: vault
        uses: hashicorp/vault-action@v3
        with:
          url: ${{ env.VAULT_URL }}
          tlsSkipVerify: true                # remove when proper TLS cert is added
          method: jwt
          role: ${{ env.VAULT_ROLE }}
          jwtGithubAudience: vault           # must match bound_audiences in Vault role
          exportEnv: true
          secrets: |
            secret/data/myapp/config api_key     | API_KEY ;
            secret/data/myapp/config db_password | DB_PASSWORD ;

      - name: Verify secrets loaded
        run: |
          if [[ -z "$API_KEY" ]]; then
            echo "ERROR: API_KEY is empty"
            exit 1
          fi
          if [[ -z "$DB_PASSWORD" ]]; then
            echo "ERROR: DB_PASSWORD is empty"
            exit 1
          fi
          echo "All secrets loaded successfully"
          echo "API_KEY length    : ${#API_KEY}"
          echo "DB_PASSWORD length: ${#DB_PASSWORD}"

      - name: Deploy
        run: |
          echo "Branch : ${{ github.ref_name }}"
          echo "Actor  : ${{ github.actor }}"
          echo "Commit : ${{ github.sha }}"
          # use $API_KEY and $DB_PASSWORD here
```

### Workflow best practices applied

| Practice | Why |
|---|---|
| `workflow_dispatch` | Re-run deploys without dummy commits |
| `concurrency` + `cancel-in-progress` | Prevents two deploys running simultaneously |
| `env: VAULT_URL` | Change IP in one place, not per-step |
| `timeout-minutes: 10` | Kills hung jobs — never wastes Actions minutes |
| `fetch-depth: 1` | Shallow clone — faster, no history needed |
| `id: vault` | Step outputs referenceable in later steps |
| `jwtGithubAudience: vault` | Explicit audience — prevents JWT replay attacks |
| Empty secret check | Fail fast with clear error, not silent failure |
| `${#SECRET}` | Print length not value — safe confirmation |

### Secrets format reference

```
secret/data/myapp/config   api_key   |  API_KEY
└─── KV v2 path ──────┘   └─ field ┘  └─ env var name ┘
```

Multiple secrets separated by `;`

---

## 6. AWS Security Group

Security group: `sg-017c87afd41894f41 — vault-security`

### Inbound rules

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| `8200` | TCP | `0.0.0.0/0` | Vault API — reachable by GitHub runners |
| `8201` | TCP | `172.31.0.0/16` | Vault HA cluster communication (VPC only) |
| `22` | TCP | `0.0.0.0/0` | SSH admin access |
| `80` | TCP | `0.0.0.0/0` | HTTP |

### Outbound rules

| Port | Protocol | Destination | Purpose |
|---|---|---|---|
| All | All | `0.0.0.0/0` | Vault fetches GitHub JWKS to verify JWTs |

> **Production hardening:** Replace `0.0.0.0/0` on port 8200 with GitHub Actions IP ranges.  
> Fetch current ranges from: `https://api.github.com/meta` → look for the `"actions"` key.

---

## 7. Troubleshooting

| Error | Root Cause | Fix |
|---|---|---|
| `ETIMEDOUT` | Port 8200 blocked by Security Group | Open port 8200 inbound in AWS SG |
| `ECONNRESET` | Vault serves HTTPS, workflow sends HTTP | Change `url: http://` → `url: https://` |
| `audience claim does not match` | `jwtGithubAudience: vault` set but role has no `bound_audiences` | Add `"bound_audiences": ["vault"]` to Vault role |
| `audience found but no audiences bound` | Role missing `bound_audiences` | Add `"bound_audiences": ["vault"]` to role OR remove `jwtGithubAudience` from workflow |
| `error converting bound_claims: expected map` | Shell escaping issue with inline JSON | Use `@/tmp/role.json` file method |
| `sub does not match bound_claims` | Trailing slash in repo name | Remove trailing `/` — use `my-pvt-project` not `my-pvt-project/` |
| `Vault sealed` | Vault restarted and needs unsealing | Run `vault operator unseal` (3 times for threshold=3) |

---

## 8. Production Checklist

- [ ] Replace `tlsSkipVerify: true` with `caCertificate: ${{ secrets.VAULT_CA_CERT }}`
- [ ] Restrict Security Group port 8200 source to GitHub Actions IP ranges only
- [ ] Add GitHub Environment protection rules (Settings → Environments → Required reviewers)
- [ ] Enable Vault audit logging: `vault audit enable file file_path=/var/log/vault/audit.log`
- [ ] Tighten `bound_claims` to environment level for production deploys
- [ ] Add `max_ttl` to the role in addition to `ttl`
- [ ] Create a dedicated policy per repo — not shared `readonly`
- [ ] Add proper TLS cert from Let's Encrypt or your org's CA

---

## Vault Cluster Info

| Node | API Address | Active |
|---|---|---|
| `vault-node-1` | `https://172.31.10.120:8200` | ✅ Yes |
| `vault-node-2` | `https://172.31.93.237:8200` | No |
| `vault-node-3` | `https://172.31.20.21:8200` | No |

- **Storage:** Raft (HA enabled)
- **Version:** 1.21.4
- **Cluster name:** `vault-ha-cluster`

---