# HashiCorp Vault — GitHub Auth Method

> **Vault Server:** `vault-node-1`  
> **GitHub Org:** `bhojeshwarsahu-org`  
> **GitHub User:** `bhojeshwarsahu`  
> Authenticate to Vault using your GitHub Personal Access Token — no separate credential system needed.

---

## Table of Contents

1. [Overview](#1-overview)
2. [How It Works](#2-how-it-works)
3. [Prerequisites](#3-prerequisites)
4. [GitHub Personal Access Token](#4-github-personal-access-token)
5. [Vault Configuration](#5-vault-configuration)
6. [Login and Verify](#6-login-and-verify)
7. [Team vs User Mapping](#7-team-vs-user-mapping)
8. [Managing GitHub Auth](#8-managing-github-auth)
9. [Troubleshooting](#9-troubleshooting)
10. [Production Checklist](#10-production-checklist)

---

## 1. Overview

### What is GitHub auth?

GitHub auth allows developers to authenticate to Vault using their **GitHub Personal Access Token (PAT)**. Vault maps GitHub organizations, teams, or individual users to Vault policies.

No separate credential management needed — your GitHub identity IS your Vault identity.

### When to use it

| Scenario | Use GitHub Auth |
|---|---|
| Developer authenticating from laptop/terminal | ✅ |
| Team members needing Vault access via GitHub identity | ✅ |
| CI/CD pipeline | ❌ use OIDC instead |
| EC2 / Lambda authenticating at runtime | ❌ use AWS auth instead |
| On-prem server or Docker container | ❌ use AppRole instead |

### GitHub auth vs other methods

| Method | Who uses it | Identity source |
|---|---|---|
| Token | Root admin | Vault itself |
| GitHub auth | Developers on laptop | GitHub PAT |
| AWS IAM | EC2 / Lambda / ECS | AWS IAM role |
| GitHub Actions OIDC | CI/CD pipeline | GitHub JWT |
| AppRole | On-prem / Docker | RoleID + SecretID |

---

## 2. How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Developer (local machine)                                  │
│       │                                                     │
│       │  ① vault login -method=github token="ghp_xxx"      │
│       ▼                                                     │
│  HashiCorp Vault                                            │
│       │                                                     │
│       │  ② GET https://api.github.com/user                 │
│       │     (verify token, get username)                    │
│       ▼                                                     │
│  GitHub API                                                 │
│       │                                                     │
│       │  ③ returns: username, org membership, teams        │
│       ▼                                                     │
│  Vault checks:                                              │
│    - Is user in configured organization?                    │
│    - Does user/team have a policy mapping?                  │
│       │                                                     │
│       │  ④ issues Vault token with mapped policy           │
│       ▼                                                     │
│  Developer can now read/write secrets                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Prerequisites

### GitHub Organization required

> **Important:** GitHub auth requires a **GitHub Organization** account.
> Personal accounts return `404` from GitHub's `/orgs/{username}` API.

```
Personal account (bhojeshwarsahu)   →  404 Not Found — does NOT work
GitHub Organization (bhojeshwarsahu-org) →  works ✅
```

### Create a free GitHub Organization

```
GitHub → your avatar (top right)
→ Organizations → New organization
→ Choose "Free" plan
→ Name: bhojeshwarsahu-org
→ Skip adding members for now
```

---

## 4. GitHub Personal Access Token

### Generate the token

```
GitHub → Settings → Developer Settings
→ Personal Access Tokens → Tokens (classic)
→ Generate new token (classic)

Name:    vault-login
Expiry:  7-30 days (your preference)
Scope:   read:org  ← this is the ONLY scope needed
```

> **Why `read:org` only?**  
> Vault only needs to verify your organization membership and team assignments.  
> It does not need access to your repositories, code, or any other GitHub data.

### Token security rules

```
✅  Use it interactively only — never store in scripts
✅  Set short expiry — 30 days max
✅  Regenerate regularly
❌  Never put in .bashrc, .zshrc, or any config file
❌  Never commit to a repository
❌  Never share with anyone
```

---

## 5. Vault Configuration

### Step 1 — Enable GitHub auth method

```bash
vault auth enable github
```

Expected output:
```
Success! Enabled github auth method at: github/
```

### Step 2 — Configure GitHub organization

```bash
vault write auth/github/config \
  organization="bhojeshwarsahu-org"
```

Expected output:
```
Success! Data written to: auth/github/config
```

> Replace `bhojeshwarsahu-org` with your actual GitHub Organization name.
> Using a personal username here returns `404 Not Found`.

### Step 3 — Map GitHub user to Vault policy

```bash
# map specific GitHub user → admin policy
vault write auth/github/map/users/bhojeshwarsahu \
  value="admin"
```

Expected output:
```
Success! Data written to: auth/github/map/users/bhojeshwarsahu
```

### Step 4 — Map GitHub team to Vault policy (optional)

```bash
# map entire team → readonly policy
vault write auth/github/map/teams/developers \
  value="readonly"

# map devops team → admin policy
vault write auth/github/map/teams/devops \
  value="admin"
```

> Team mapping only works if your GitHub Organization has teams created.
> Individual user mapping works without teams.

---

## 6. Login and Verify

### Login using GitHub PAT

```bash
vault login -method=github token="ghp_xxxxxxxxxxxxxxxxxxxx"
```

Successful output:
```
Success! You are now authenticated.

Key                    Value
---                    -----
token                  hvs.xxxxxxxxxxxxxxxxxxxx
token_accessor         SZjkjdjdjljldldjldj
token_duration         768h
token_renewable        true
token_policies         ["admin" "default"]
identity_policies      []
policies               ["admin" "default"]
token_meta_org         bhojeshwarsahu-org
token_meta_username    bhojeshwarsahu
```

**What each field means:**

| Field | Value | Meaning |
|---|---|---|
| `token_duration` | `768h` | Token valid for 32 days |
| `token_policies` | `["admin" "default"]` | Policies from user mapping |
| `token_meta_org` | `bhojeshwarsahu-org` | GitHub org Vault verified |
| `token_meta_username` | `bhojeshwarsahu` | GitHub username confirmed |

### Verify secrets are accessible

```bash
vault kv get secret/myapp/config
```

Output:
```
====== Secret Path ======
secret/data/myapp/config

======= Metadata =======
Key                Value
---                -----
created_time       2026-03-17T07:53:12.886443724Z
version            2

======= Data =======
Key            Value
---            -----
api_key        abc-xyz-789
db_password    newpassword456
```

---

## 7. Team vs User Mapping

### Priority order

When a user logs in, Vault applies policies in this order:

```
1. User mapping    (auth/github/map/users/username)   ← highest priority
2. Team mapping    (auth/github/map/teams/teamname)
3. Default policy  (always applied)
```

If a user has both a user mapping and is in a mapped team, **both policies are applied**.

### Mapping examples

```bash
# give one specific user admin access
vault write auth/github/map/users/bhojeshwarsahu \
  value="admin"

# give entire developers team readonly access
vault write auth/github/map/teams/developers \
  value="readonly"

# give devops team admin access
vault write auth/github/map/teams/devops \
  value="admin"

# give multiple policies to a team (comma separated)
vault write auth/github/map/teams/sre \
  value="admin,readonly"
```

### Real-world org example

```
GitHub Org: my-company
├── Team: devops      → Vault policy: admin
├── Team: developers  → Vault policy: readonly
├── Team: qa          → Vault policy: readonly
└── User: cto         → Vault policy: admin (individual override)
```

---

## 8. Managing GitHub Auth

### List all mappings

```bash
# list team mappings
vault list auth/github/map/teams

# list user mappings
vault list auth/github/map/users
```

### Read a specific mapping

```bash
vault read auth/github/map/users/bhojeshwarsahu
vault read auth/github/map/teams/developers
```

### Update a mapping

```bash
# change user policy
vault write auth/github/map/users/bhojeshwarsahu \
  value="readonly"
```

### Delete a mapping

```bash
# remove user mapping (user loses Vault access)
vault delete auth/github/map/users/bhojeshwarsahu

# remove team mapping
vault delete auth/github/map/teams/developers
```

### Read current GitHub config

```bash
vault read auth/github/config
```

### Disable GitHub auth entirely

```bash
vault auth disable github
```

---

## 9. Troubleshooting

| Error | Root Cause | Fix |
|---|---|---|
| `404 Not Found` on org | Using personal username not org name | Create a GitHub Org and use org name |
| `403 Forbidden` | PAT missing `read:org` scope | Regenerate token with `read:org` checked |
| `user is not part of required org` | User not a member of the configured org | Add user to the GitHub Organization |
| `no mapping found` | No user or team mapping exists | Run `vault write auth/github/map/users/USERNAME value="POLICY"` |
| `token expired` | PAT has expired | Generate a new PAT and login again |
| `invalid token` | Wrong PAT or typo | Regenerate and copy carefully |

### Debug commands

```bash
# check github auth config
vault read auth/github/config

# list all auth methods enabled
vault auth list

# check what policies a token has
vault token lookup
```

---

## 10. Production Checklist

- [ ] Use GitHub Organization — not personal accounts
- [ ] Set PAT expiry to maximum 30 days — rotate regularly
- [ ] Use team mappings — not individual user mappings (easier to manage at scale)
- [ ] Apply least-privilege policies — developers get `readonly`, only devops get `admin`
- [ ] Enable Vault audit logging to track all GitHub auth logins
- [ ] Revoke PATs when team members leave — removes Vault access immediately
- [ ] Set `token_ttl` on the GitHub auth config to limit issued token lifetime:

```bash
vault write auth/github/config \
  organization="bhojeshwarsahu-org" \
  token_ttl="8h" \
  token_max_ttl="24h"
```

---

## Auth Methods Completed

| # | Method | Actor | Status |
|---|---|---|---|
| 1 | Token | Human / admin (root) | ✅ |
| 2 | GitHub Actions OIDC | CI/CD pipeline | ✅ |
| 3 | AWS IAM | EC2 / Lambda / ECS | ✅ |
| 4 | GitHub Auth | Developers on laptop | ✅ |
| 5 | AppRole | On-prem / Docker / non-AWS | 🔜 |

---

*Practice setup — vault-node-1 — GitHub Org: bhojeshwarsahu-org*