# HashiCorp Vault — AWS IAM Auth Method

> **Vault Server:** `vault-node-1`  
> **AWS Account ID:** `xxxxxxxxxxxx`  
> **Auth Method:** AWS IAM — zero standing credentials, no passwords stored anywhere.

---

## Table of Contents

1. [Overview](#1-overview)
2. [How It Works](#2-how-it-works)
3. [AWS IAM Setup](#3-aws-iam-setup)
4. [Vault Configuration](#4-vault-configuration)
5. [Testing the Auth Flow](#5-testing-the-auth-flow)
6. [Real-World App Usage](#6-real-world-app-usage)
7. [bound_iam_principal_arn Patterns](#7-bound_iam_principal_arn-patterns)
8. [Troubleshooting](#8-troubleshooting)
9. [Production Checklist](#9-production-checklist)

---

## 1. Overview

### Why AWS auth method?

Every EC2 instance already has a built-in identity — its IAM role. The AWS auth method lets your EC2
instances prove who they are using AWS's own cryptographic infrastructure. No passwords, no tokens,
nothing stored on the instance.

| | Static Token | AWS IAM Auth |
|---|---|---|
| **Credential stored on EC2** | Yes — risky | No — nothing to leak |
| **Rotation** | Manual | Automatic (STS temp creds) |
| **If instance is compromised** | Token leaked permanently | Attacker still needs valid IAM role |
| **Audit trail** | Token only | Account ID, role, instance ID |

### When to use it

| Scenario | Use AWS Auth |
|---|---|
| EC2 app server fetching DB password at boot | ✅ |
| Lambda reading API keys per invocation | ✅ |
| ECS / EKS container fetching secrets at runtime | ✅ |
| GitHub Actions CI/CD pipeline | ❌ use OIDC instead |
| On-prem server (not on AWS) | ❌ use AppRole instead |

---

## 2. How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  EC2 Instance                                                   │
│  (has IAM role attached)                                        │
│       │                                                         │
│       │  ① fetch temp credentials from instance metadata       │
│       │     http://169.254.169.254/latest/meta-data/iam/...    │
│       ▼                                                         │
│  Signs a request using IAM role credentials                     │
│       │                                                         │
│       │  ② POST /v1/auth/aws/login                             │
│       │     { iam_request_headers, iam_request_body, role }    │
│       ▼                                                         │
│  HashiCorp Vault                                                │
│       │                                                         │
│       │  ③ sts:GetCallerIdentity (verify the signed request)   │
│       ▼                                                         │
│  AWS STS                                                        │
│       │                                                         │
│       │  ④ returns: AccountID, ARN, UserID                     │
│       ▼                                                         │
│  Vault checks bound_iam_principal_arn matches role              │
│       │                                                         │
│       │  ⑤ issues Vault token (1h TTL, readonly policy)        │
│       ▼                                                         │
│  EC2 reads secrets from Vault                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Key insight:** The EC2 instance never sends a password.
> It sends a **signed request** — AWS's own signature proves the identity.
> Vault asks AWS "is this signature real?" and AWS confirms it.

---

## 3. AWS IAM Setup

### Step 1 — Create the IAM policy

Go to: `AWS Console → IAM → Policies → Create policy`

**Name:** `vault-sts-policy`

**JSON:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity",
        "iam:GetRole"
      ],
      "Resource": "*"
    }
  ]
}
```

| Permission | Why needed |
|---|---|
| `sts:GetCallerIdentity` | Vault verifies the EC2's signed request against STS |
| `iam:GetRole` | Vault resolves the role ARN to a permanent role ID (prevents ARN-reuse attacks) |

### Step 2 — Create the IAM role

Go to: `AWS Console → IAM → Roles → Create role`

```
Trusted entity type  →  AWS service
Use case             →  EC2

Name: vault-server-role

Attach policy: vault-sts-policy
```

### Step 3 — Attach the role to vault-node-1

```
EC2 → Instances → vault-node-1
→ Actions → Security → Modify IAM role
→ Select: vault-server-role
→ Save
```

### Summary of AWS resources

| Resource | Name | Purpose |
|---|---|---|
| IAM Policy | `vault-sts-policy` | Grants `sts:GetCallerIdentity` + `iam:GetRole` |
| IAM Role | `vault-server-role` | Identity worn by `vault-node-1` EC2 |

---

## 4. Vault Configuration

### Step 1 — Enable AWS auth method

```bash
vault auth enable aws
```

Expected output:
```
Success! Enabled aws auth method at: aws/
```

### Step 2 — Configure AWS client

```bash
vault write auth/aws/config/client region="us-east-1"
```

Expected output:
```
Success! Data written to: auth/aws/config/client
```

> Vault automatically uses `vault-server-role` (attached to vault-node-1) to call STS.
> No access keys needed — the IAM role handles authentication.

### Step 3 — Create the Vault role

```bash
vault write auth/aws/role/my-app-server \
  auth_type="iam" \
  policies="readonly" \
  bound_iam_principal_arn="arn:aws:iam::xxxxxxxxxxxx:role/vault-server-role" \
  ttl="1h" \
  max_ttl="4h"
```

Expected output:
```
Success! Data written to: auth/aws/role/my-app-server
```

**Role fields explained:**

| Field | Value | Purpose |
|---|---|---|
| `auth_type` | `iam` | IAM-based auth (recommended over `ec2`) |
| `policies` | `readonly` | Vault policy attached to the issued token |
| `bound_iam_principal_arn` | `arn:aws:iam::xxxxxxxxxxxx:role/vault-server-role` | Only EC2s with this IAM role can authenticate |
| `ttl` | `1h` | Token expires after 1 hour |
| `max_ttl` | `4h` | Token cannot be renewed beyond 4 hours |

### Verify the role was saved

```bash
vault read auth/aws/role/my-app-server
```

---

## 5. Testing the Auth Flow

### Login using AWS IAM identity

```bash
vault login -method=aws role="my-app-server"
```

Successful output:
```
Success! You are now authenticated.

Key                      Value
---                      -----
token                    hvs.CAESIE3D-xxxxxxxxxxxx
token_accessor           c6zrZZas5wwvnk0v38Br4ki0
token_duration           1h
token_renewable          true
token_policies           ["default" "readonly"]
token_meta_account_id    xxxxxxxxxxxx
token_meta_auth_type     iam
token_meta_role_id       00f9e628-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### Verify secrets are readable

```bash
vault kv get secret/myapp/config
```

Output:
```
====== Secret Path ======
secret/data/myapp/config

======= Data =======
Key            Value
---            -----
api_key        abc-xyz-789
db_password    newpassword456
```

> **What just happened:**
> No password was used. `vault-node-1` proved its identity using the
> `vault-server-role` IAM role alone. Vault issued a 1h token with `readonly` policy.

---

## 6. Real-World App Usage

This is how a real application fetches secrets at startup:

```bash
#!/bin/bash
# app-startup.sh — runs on your EC2 when the app boots

# Step 1 — authenticate using AWS IAM identity
VAULT_TOKEN=$(vault login \
  -method=aws \
  -format=json \
  role="my-app-server" | jq -r '.auth.client_token')

# Step 2 — export token for subsequent commands
export VAULT_TOKEN

# Step 3 — fetch individual secret fields
API_KEY=$(vault kv get -field=api_key secret/myapp/config)
DB_PASSWORD=$(vault kv get -field=db_password secret/myapp/config)

# Step 4 — confirm loaded (print length, never the value)
echo "API_KEY length    : ${#API_KEY}"
echo "DB_PASSWORD length: ${#DB_PASSWORD}"
echo "Auth method       : AWS IAM"

# Step 5 — use secrets in your app
export API_KEY
export DB_PASSWORD
# start your application here
```

---

## 7. bound_iam_principal_arn Patterns

`bound_iam_principal_arn` is your trust boundary — only EC2s with matching IAM roles can authenticate.

### Pattern 1 — Exact role (most restrictive — recommended)

```bash
bound_iam_principal_arn="arn:aws:iam::xxxxxxxxxxxx:role/my-app-role"
```

### Pattern 2 — Any role in account (less strict)

```bash
bound_iam_principal_arn="arn:aws:iam::xxxxxxxxxxxx:role/*"
```

### Pattern 3 — Specific instance profile

```bash
bound_iam_principal_arn="arn:aws:iam::xxxxxxxxxxxx:instance-profile/my-app-profile"
```

### Pattern 4 — Multiple roles (comma separated)

```bash
bound_iam_principal_arn="arn:aws:iam::xxxxxxxxxxxx:role/app-role,arn:aws:iam::xxxxxxxxxxxx:role/worker-role"
```

### Tighten further with additional bindings

```bash
vault write auth/aws/role/my-app-server \
  auth_type="iam" \
  policies="readonly" \
  bound_iam_principal_arn="arn:aws:iam::xxxxxxxxxxxx:role/vault-server-role" \
  bound_account_id="xxxxxxxxxxxx" \
  ttl="1h" \
  max_ttl="4h"
```

| Binding | Restricts to |
|---|---|
| `bound_iam_principal_arn` | Specific IAM role |
| `bound_account_id` | Specific AWS account |
| `bound_region` | Specific AWS region |
| `bound_vpc_id` | Specific VPC |
| `bound_subnet_id` | Specific subnet |
| `bound_ami_id` | Specific AMI (EC2 type only) |

---

## 8. Troubleshooting

| Error | Root Cause | Fix |
|---|---|---|
| `AccessDenied: iam:GetRole` | `vault-sts-policy` missing `iam:GetRole` | Add `iam:GetRole` to policy JSON |
| `AccessDenied: sts:GetCallerIdentity` | IAM policy not attached to role | Attach `vault-sts-policy` to `vault-server-role` |
| `instance not found` | EC2 has no IAM role attached | Attach `vault-server-role` to EC2 via Security → Modify IAM role |
| `role not found` | Wrong role name in `bound_iam_principal_arn` | Verify with `aws sts get-caller-identity` |
| `entry for role not found` | Vault role name mismatch | Check `vault list auth/aws/role` |

### Useful debug commands

```bash
# check what identity vault-node-1 is using
aws sts get-caller-identity

# list all AWS auth roles in Vault
vault list auth/aws/role

# read a specific role config
vault read auth/aws/role/my-app-server

# check AWS auth config
vault read auth/aws/config/client
```

---

## 9. Production Checklist

- [ ] Create a **separate IAM role per application** — not shared `vault-server-role`
- [ ] Add `bound_account_id` to restrict to your specific AWS account
- [ ] Add `bound_vpc_id` to restrict to instances inside your VPC only
- [ ] Use a **dedicated Vault policy per app** — not shared `readonly`
- [ ] Set `max_ttl` to the minimum your app can tolerate
- [ ] Enable token renewal in your app so it refreshes before expiry
- [ ] Enable Vault audit logging to track all AWS auth logins
- [ ] Rotate the IAM role periodically using AWS IAM Access Analyzer

---

## Auth Methods Completed

| # | Method | Actor | Status |
|---|---|---|---|
| 1 | Token | Human / admin (root) | ✅ |
| 2 | GitHub Actions OIDC | CI/CD pipeline | ✅ |
| 3 | AWS IAM | EC2 / Lambda / ECS | ✅ |
| 4 | AppRole | On-prem / Docker / non-AWS | 🔜 |

---

*Practice setup — vault-node-1 — AWS account: xxxxxxxxxxxx*