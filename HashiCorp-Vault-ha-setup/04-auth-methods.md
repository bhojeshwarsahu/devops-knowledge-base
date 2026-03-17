# 04 — Vault Auth Methods

> *"Never give an application a username and password. Give it an identity — one that expires, one that can be revoked, and one that leaves a trail."*

---

## What is an Auth Method?

An auth method is how an **identity proves itself to Vault** in exchange for a token. That token carries attached policies which determine what the identity can do.

```
Identity (user / app / EC2)
        │
        │  presents credentials
        ▼
   Auth Method
        │
        │  validates credentials
        ▼
  Vault issues Token
        │
        │  token has policies attached
        ▼
  Access to secrets
```

Without an auth method, the only way to authenticate is with a manually created token — which is not scalable or secure for teams or applications.

---

## Auth methods covered in this document

| Method | Used by | How it authenticates |
|---|---|---|
| `userpass` | Team members | Username + password |
| `approle` | Applications / services | Role ID + Secret ID |
| `aws` | EC2 instances / Lambda | AWS IAM identity (no credentials needed) |
| `github` | Developers | GitHub personal access token |

---

## Prerequisites

- Vault cluster initialized and unsealed
- Audit log enabled (see `02-audit-log.md`)
- Policies created (see `03-policies.md`)
- Root token or admin token

---

## Set environment variables

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<your-root-token>"
```

---

## Auth Method 1 — userpass

### What is it?

The simplest auth method — users authenticate with a username and password. Best for team members accessing Vault via CLI or UI.

### Enable userpass

```bash
vault auth enable userpass
```

### Create users

```bash
# Admin user
vault write auth/userpass/users/admin-user \
  password="StrongPassword@123" \
  policies="admin"

# Readonly user
vault write auth/userpass/users/readonly-user \
  password="StrongPassword@456" \
  policies="readonly"
```

### Login via CLI

```bash
vault login -method=userpass \
  username=admin-user \
  password="StrongPassword@123"
```

### Login via UI

Open `http://<Node-1-Public-IP>/ui/` → select **Username** method → enter credentials.

### Manage users

```bash
# List all users
vault list auth/userpass/users

# Read a user
vault read auth/userpass/users/admin-user

# Update password
vault write auth/userpass/users/admin-user \
  password="NewPassword@789"

# Update policies
vault write auth/userpass/users/admin-user \
  policies="admin,readonly"

# Delete a user
vault delete auth/userpass/users/admin-user
```

### Set token TTL for userpass

```bash
# Tokens expire after 8 hours, renewable up to 24 hours
vault write auth/userpass/users/admin-user \
  password="StrongPassword@123" \
  policies="admin" \
  ttl="8h" \
  max_ttl="24h"
```

---

## Auth Method 2 — AppRole

### What is it?

AppRole is designed for **applications and services** — not humans. Instead of a username/password, an application authenticates with two pieces:

- **Role ID** — like a username, not secret, can be baked into app config
- **Secret ID** — like a password, secret, short-lived, fetched at runtime

This separation means even if someone finds your Role ID, they cannot authenticate without the Secret ID.

```
Application
    │
    ├── Role ID   (static, embedded in app)
    └── Secret ID (dynamic, fetched from CI/CD or secrets manager)
           │
           └──► Vault AppRole ──► Token with policies
```

### Enable AppRole

```bash
vault auth enable approle
```

### Create a role for your application

```bash
vault write auth/approle/role/myapp \
  policies="readonly" \
  secret_id_ttl="24h" \
  token_ttl="1h" \
  token_max_ttl="4h" \
  secret_id_num_uses=10
```

| Parameter | Value | Meaning |
|---|---|---|
| `policies` | readonly | Token gets readonly policy |
| `secret_id_ttl` | 24h | Secret ID expires after 24 hours |
| `token_ttl` | 1h | Issued token valid for 1 hour |
| `token_max_ttl` | 4h | Token can be renewed up to 4 hours total |
| `secret_id_num_uses` | 10 | Secret ID can only be used 10 times |

### Get the Role ID (embed this in your app config)

```bash
vault read auth/approle/role/myapp/role-id
```

Expected output:

```
Key        Value
---        -----
role_id    xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### Generate a Secret ID (fetch this from CI/CD at deploy time)

```bash
vault write -f auth/approle/role/myapp/secret-id
```

Expected output:

```
Key                   Value
---                   -----
secret_id             yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
secret_id_accessor    zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz
secret_id_ttl         24h
secret_id_num_uses    10
```

### Authenticate with Role ID + Secret ID

```bash
vault write auth/approle/login \
  role_id="<role-id>" \
  secret_id="<secret-id>"
```

Expected output:

```
Key                     Value
---                     -----
token                   hvs.xxxxxxxxxxxx
token_accessor          xxxxxxxxxxxx
token_duration          1h
token_renewable         true
token_policies          ["default", "readonly"]
```

### How an application uses AppRole in practice

In your application code:

```bash
# Step 1 — Get secret ID from environment or secrets manager
SECRET_ID=$(vault write -f -field=secret_id auth/approle/role/myapp/secret-id)

# Step 2 — Login and get token
VAULT_TOKEN=$(vault write -field=token auth/approle/login \
  role_id="<role-id>" \
  secret_id="$SECRET_ID")

# Step 3 — Use token to read secrets
VAULT_TOKEN=$VAULT_TOKEN vault kv get secret/myapp/config
```

### Manage AppRole roles

```bash
# List all AppRole roles
vault list auth/approle/role

# Read role config
vault read auth/approle/role/myapp

# Delete a role
vault delete auth/approle/role/myapp

# Revoke a specific secret ID
vault write auth/approle/role/myapp/secret-id/destroy \
  secret_id_accessor="<accessor>"
```

---

## Auth Method 3 — AWS

### What is it?

The AWS auth method allows **EC2 instances and Lambda functions** to authenticate to Vault using their AWS IAM identity — **no credentials or passwords needed**. The instance proves its identity using a cryptographically signed AWS request.

```
EC2 Instance
    │
    │  signs request using instance IAM role
    ▼
Vault AWS Auth Method
    │
    │  verifies signature with AWS STS
    ▼
Token issued with attached policies
```

This is the gold standard for cloud-native secrets management — your EC2 instances never need a Secret ID, password, or any credential stored on them.

### Enable AWS auth method

```bash
vault auth enable aws
```

### Configure AWS credentials for Vault to call STS

Vault needs permission to call `sts:GetCallerIdentity` to verify EC2 identity.

**Option A — Use EC2 instance role (recommended)**

If vault-node-1 already has an IAM role with `sts:GetCallerIdentity` permission, Vault will use it automatically:

```bash
vault write auth/aws/config/client \
  iam_server_id_header_value="vault.yourcompany.com"
```

**Option B — Provide AWS credentials explicitly**

```bash
vault write auth/aws/config/client \
  access_key="AKIAIOSFODNN7EXAMPLE" \
  secret_key="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" \
  region="us-east-1"
```

### Create a role for EC2 instances

```bash
# Allow any EC2 instance with this IAM role ARN to authenticate
vault write auth/aws/role/ec2-readonly \
  auth_type="iam" \
  policies="readonly" \
  bound_iam_principal_arn="arn:aws:iam::<ACCOUNT_ID>:role/<EC2_IAM_ROLE_NAME>" \
  ttl="1h" \
  max_ttl="4h"
```

| Parameter | Meaning |
|---|---|
| `auth_type` | `iam` for IAM-based auth (recommended), `ec2` for EC2 metadata-based |
| `bound_iam_principal_arn` | Only EC2 instances with this IAM role can authenticate |
| `policies` | Policies attached to the issued token |
| `ttl` | Token lifetime |

### Authenticate from an EC2 instance

Run this on the EC2 instance that needs to authenticate:

```bash
# Install vault CLI on the EC2 instance first
# Then authenticate using IAM method
vault login -method=aws role="ec2-readonly"
```

Vault calls AWS STS to verify the instance's identity — no credentials needed on the EC2 instance.

### Restrict to specific AWS accounts or VPCs

```bash
# Only allow instances in specific AWS account
vault write auth/aws/role/ec2-readonly \
  auth_type="iam" \
  policies="readonly" \
  bound_account_id="<AWS_ACCOUNT_ID>" \
  bound_iam_principal_arn="arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>" \
  ttl="1h"
```

### Manage AWS auth roles

```bash
# List all AWS auth roles
vault list auth/aws/role

# Read a role
vault read auth/aws/role/ec2-readonly

# Delete a role
vault delete auth/aws/role/ec2-readonly
```

---

## Auth Method 4 — GitHub

### What is it?

GitHub auth allows developers to authenticate to Vault using their **GitHub personal access token**. Vault maps GitHub organizations, teams, or individual users to Vault policies.

This is ideal for development teams who already use GitHub — no separate credential management needed.

### Enable GitHub auth

```bash
vault auth enable github
```

### Configure your GitHub organization

```bash
vault write auth/github/config \
  organization="your-github-org-name"
```

Replace `your-github-org-name` with your actual GitHub organization name (e.g. `hashicorp`, `mycompany`).

### Map a GitHub team to a Vault policy

```bash
# Members of the "devops" team get admin policy
vault write auth/github/map/teams/devops \
  value="admin"

# Members of the "developers" team get readonly policy
vault write auth/github/map/teams/developers \
  value="readonly"
```

### Map a specific GitHub user to a policy

```bash
# A specific user gets admin policy regardless of team
vault write auth/github/map/users/bhojeshwarsahu \
  value="admin"
```

### Authenticate using GitHub token

```bash
# Generate a GitHub Personal Access Token at:
# GitHub → Settings → Developer Settings → Personal Access Tokens
# Required scope: read:org

vault login -method=github token="ghp_xxxxxxxxxxxxxxxxxxxx"
```

Expected output:

```
Key                    Value
---                    -----
token                  hvs.xxxxxxxxxxxx
token_duration         768h
token_policies         ["admin", "default"]
token_meta_org         your-github-org-name
token_meta_username    bhojeshwarsahu
```

### Manage GitHub auth

```bash
# List team mappings
vault list auth/github/map/teams

# Read a team mapping
vault read auth/github/map/teams/devops

# List user mappings
vault list auth/github/map/users

# Delete a team mapping
vault delete auth/github/map/teams/devops
```

---

## Listing all enabled auth methods

```bash
vault auth list
```

Expected output after enabling all 4:

```
Path         Type        Accessor                  Description
----         ----        --------                  -----------
approle/     approle     auth_approle_xxxxxxxx     n/a
aws/         aws         auth_aws_xxxxxxxx          n/a
github/      github      auth_github_xxxxxxxx      n/a
token/       token       auth_token_xxxxxxxx        token based credentials
userpass/    userpass    auth_userpass_xxxxxxxx    n/a
```

---

## Disabling an auth method

```bash
# Disabling an auth method revokes ALL tokens issued by it
vault auth disable github/
```

> Warning: This immediately revokes all tokens that were issued by that auth method. Users will be logged out.

---

## Choosing the right auth method

| Use case | Recommended method |
|---|---|
| Team member logging into UI | `userpass` |
| Developer on your team | `github` |
| Application running on a server | `approle` |
| Application running on EC2 | `aws` |
| Application running in Kubernetes | `kubernetes` (future) |
| Automated CI/CD pipeline | `approle` |

---

## Common issues

| Issue | Cause | Fix |
|---|---|---|
| `invalid role name` on AppRole login | Role not created | Run `vault list auth/approle/role` to verify |
| `invalid secret id` on AppRole | Secret ID expired or used too many times | Generate a new Secret ID |
| GitHub auth `could not verify membership` | Token lacks `read:org` scope | Regenerate token with correct scope |
| AWS auth `failed to verify` | Vault cannot call STS | Verify IAM role has `sts:GetCallerIdentity` permission |
| `auth method already enabled` | Trying to enable twice | Run `vault auth list` to check |

---

## Quick reference

```bash
# List all auth methods
vault auth list

# Enable an auth method
vault auth enable <method>

# Disable an auth method
vault auth disable <method>/

# userpass — create user
vault write auth/userpass/users/<username> password="<pass>" policies="<policy>"

# userpass — login
vault login -method=userpass username=<user> password=<pass>

# AppRole — create role
vault write auth/approle/role/<name> policies="<policy>" token_ttl="1h"

# AppRole — get role ID
vault read auth/approle/role/<name>/role-id

# AppRole — generate secret ID
vault write -f auth/approle/role/<name>/secret-id

# AppRole — login
vault write auth/approle/login role_id="<id>" secret_id="<id>"

# GitHub — configure
vault write auth/github/config organization="<org>"

# GitHub — map team
vault write auth/github/map/teams/<team> value="<policy>"

# GitHub — login
vault login -method=github token="<github-pat>"

# AWS — enable and create role
vault auth enable aws
vault write auth/aws/role/<name> auth_type="iam" policies="<policy>" \
  bound_iam_principal_arn="arn:aws:iam::<id>:role/<role>"

# AWS — login from EC2
vault login -method=aws role="<role-name>"
```

---

## Auth method best practices

- Never use the root token for applications — always use AppRole or AWS auth
- Set short `token_ttl` values — 1 hour for applications, 8 hours for humans
- Always set `token_max_ttl` to prevent indefinitely renewable tokens
- Use `bound_iam_principal_arn` in AWS auth to restrict which instances can authenticate
- Rotate AppRole Secret IDs regularly — treat them like passwords
- Use GitHub teams (not individual users) for policy mapping — easier to manage at scale
- Monitor auth attempts in the audit log — failed logins are a security signal

---

## Next step

Auth methods are configured. Your team and applications can now authenticate to Vault securely.

Proceed to **[05-kv-secrets-engine.md](./05-kv-secrets-engine.md)** — enabling and using the KV secrets engine.

---

*Part of the [Vault HA Production Setup Guide](./Vault-HA-Production-Setup-Guide.md)*