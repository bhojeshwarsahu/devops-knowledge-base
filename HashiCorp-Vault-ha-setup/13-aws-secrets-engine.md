# HashiCorp Vault — AWS Secrets Engine

> **Vault Server:** `vault-node-1`  
> **AWS Account:** `xxxxxxxxxxxx`  
> **Engine path:** `aws/`  
> Vault generates temporary AWS credentials on demand — no static access keys stored anywhere.

---

## Table of Contents

1. [Overview](#1-overview)
2. [AWS Auth Method vs AWS Secrets Engine](#2-aws-auth-method-vs-aws-secrets-engine)
3. [How It Works](#3-how-it-works)
4. [AWS IAM Prerequisites](#4-aws-iam-prerequisites)
5. [Vault Configuration](#5-vault-configuration)
6. [Generating Credentials](#6-generating-credentials)
7. [How Other Applications Use This](#7-how-other-applications-use-this)
8. [Credential Types](#8-credential-types)
9. [Managing Leases](#9-managing-leases)
10. [Troubleshooting](#10-troubleshooting)
11. [Production Checklist](#11-production-checklist)

---

## 1. Overview

### What is the AWS Secrets Engine?

The AWS Secrets Engine generates **temporary, auto-expiring AWS credentials** on demand.
Instead of creating long-lived IAM access keys and storing them in `.env` files,
your application asks Vault for credentials that expire automatically.

### Why use it?

| | Static IAM Keys (old way) | Vault AWS Secrets Engine |
|---|---|---|
| **Where stored** | `.env` files, GitHub Secrets, config | Nowhere — generated on demand |
| **Lifetime** | Permanent until manually deleted | 15 min to 12h, auto-expire |
| **If leaked** | Permanent access until rotated | Useless after TTL |
| **Rotation** | Manual, error-prone | Automatic |
| **Audit trail** | Hard to trace which app used a key | Full Vault audit log per request |
| **Per-app isolation** | One shared key for everything | Unique credentials per app per request |

---

## 2. AWS Auth Method vs AWS Secrets Engine

These are completely different things — a common source of confusion.

```
AWS Auth Method (done earlier)
  ├── Direction:  EC2 → Vault
  ├── Purpose:    EC2 proves its identity TO Vault
  └── Question:   "Who are you?" — Authentication

AWS Secrets Engine (this guide)
  ├── Direction:  Vault → AWS
  ├── Purpose:    Vault generates AWS credentials FOR your app
  └── Question:   "Here are temp AWS keys" — Secrets Generation
```

### Used together in a real app

```
Step 1: App authenticates to Vault using AWS Auth Method
        vault login -method=aws role="my-app-server"
              ↓
        Vault verifies EC2 identity via STS
              ↓
        Issues Vault token

Step 2: App requests AWS credentials from Secrets Engine
        vault read aws/creds/my-app-role
              ↓
        Vault assumes vault-server-role via STS
              ↓
        Returns temporary access_key + secret_key + session_token
```

---

## 3. How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Your App / EC2 Instance                                        │
│       │                                                         │
│       │  ① vault read aws/creds/my-app-role                    │
│       ▼                                                         │
│  HashiCorp Vault                                                │
│       │                                                         │
│       │  ② sts:AssumeRole (using vault-server-role)            │
│       ▼                                                         │
│  AWS STS                                                        │
│       │                                                         │
│       │  ③ returns temporary credentials                       │
│       │     access_key    (ASIA...)                             │
│       │     secret_key    (random)                              │
│       │     session_token (long token)                          │
│       │     expires_in    (TTL)                                 │
│       ▼                                                         │
│  Vault returns credentials to your app                          │
│       │                                                         │
│       │  ④ app uses credentials to call AWS APIs               │
│       ▼                                                         │
│  AWS API (S3, DynamoDB, etc.)                                   │
│       │                                                         │
│       │  ⑤ TTL expires → credentials auto-invalidated          │
│       ▼                                                         │
│  App requests fresh credentials from Vault (repeat from ①)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. AWS IAM Prerequisites

### Update vault-sts-policy

Vault needs `sts:AssumeRole` permission in addition to what was added for AWS Auth.

Go to: `AWS Console → IAM → Policies → vault-sts-policy → Edit`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity",
        "iam:GetRole",
        "sts:AssumeRole"
      ],
      "Resource": "*"
    }
  ]
}
```

### Update vault-server-role trust policy

Allow `vault-server-role` to assume itself (required for `assumed_role` credential type).

Go to: `AWS Console → IAM → Roles → vault-server-role → Trust relationships → Edit`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com",
        "AWS": "arn:aws:iam::xxxxxxxxxxxx:role/vault-server-role"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## 5. Vault Configuration

### Step 1 — Set environment variables

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
```

### Step 2 — Enable the AWS secrets engine

```bash
vault secrets enable aws
```

Expected output:
```
Success! Enabled the aws secrets engine at: aws/
```

### Step 3 — Configure the root credentials

```bash
vault write aws/config/root \
  region="us-east-1"
```

Expected output:
```
Success! Data written to: aws/config/root
```

> Vault automatically uses `vault-server-role` (attached to vault-node-1).
> No access keys needed — the IAM role handles it.

### Step 4 — Create a Vault role

```bash
vault write aws/roles/my-app-role \
  credential_type=assumed_role \
  role_arns="arn:aws:iam::xxxxxxxxxxxx:role/vault-server-role"
```

Expected output:
```
Success! Data written to: aws/roles/my-app-role
```

---

## 6. Generating Credentials

### Request temporary AWS credentials

```bash
vault read aws/creds/my-app-role
```

Actual output from this setup:
```
Key                Value
---                -----
lease_id           aws/creds/my-app-role/ImZROqPW5WNFsFWOKIuHz6q4
lease_duration     59m59s
lease_renewable    false
access_key         ASIA3JXEALQA5RK2YJNL
arn                arn:aws:sts::xxxxxxxxxxxx:assumed-role/vault-server-role/vault-github-...
secret_key         a3yzlXvSClFwO6ak***************************
security_token     IQoJb3JpZ2luX2Vj...  (long STS token)
session_token      IQoJb3JpZ2luX2Vj...  (same as security_token)
ttl                59m59s
```

**What each field means:**

| Field | Meaning |
|---|---|
| `lease_id` | Vault's internal ID for this credential — use to revoke early |
| `lease_duration` | How long the credentials are valid |
| `access_key` | `AWS_ACCESS_KEY_ID` — use in your app |
| `secret_key` | `AWS_SECRET_ACCESS_KEY` — use in your app |
| `security_token` | `AWS_SESSION_TOKEN` — required for assumed role creds |
| `arn` | The assumed role ARN — shows which role was assumed |
| `ttl` | Time remaining on the lease |

> **Important:** Assumed role credentials ALWAYS require `security_token` / `AWS_SESSION_TOKEN`.
> Without it, AWS will reject the request with `InvalidClientTokenId`.

---

## 7. How Other Applications Use This

### Any app on the same EC2 (shell script)

```bash
#!/bin/bash
# fetch-aws-creds.sh

# authenticate to Vault first (using AWS auth)
export VAULT_TOKEN=$(vault login \
  -method=aws \
  -format=json \
  role="my-app-server" | jq -r '.auth.client_token')

# request temporary AWS credentials
CREDS=$(vault read -format=json aws/creds/my-app-role)

# export as environment variables
export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.data.access_key')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.data.secret_key')
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.data.security_token')

echo "AWS credentials loaded. Expires in: $(echo $CREDS | jq -r '.lease_duration')"

# now use AWS CLI normally
aws s3 ls
aws ec2 describe-instances --region us-east-1
```

---

### Python application

```python
import hvac
import boto3

# Step 1 — authenticate to Vault using AWS IAM
client = hvac.Client(
    url='https://44.215.126.96:8200',
    verify=False  # use ca_cert path in production
)

client.auth.aws.iam_login(role='my-app-server')

# Step 2 — request temporary AWS credentials
aws_creds = client.secrets.aws.generate_credentials(
    name='my-app-role'
)

# Step 3 — use credentials with boto3
boto3_session = boto3.Session(
    aws_access_key_id=aws_creds['data']['access_key'],
    aws_secret_access_key=aws_creds['data']['secret_key'],
    aws_session_token=aws_creds['data']['security_token'],
    region_name='us-east-1'
)

# use any AWS service
s3 = boto3_session.client('s3')
buckets = s3.list_buckets()
print(buckets)
```

---

### Node.js application

```javascript
const vault = require('node-vault');
const AWS = require('aws-sdk');

async function getAWSCredentials() {
  // Step 1 — connect to Vault
  const client = vault({
    apiVersion: 'v1',
    endpoint: 'https://44.215.126.96:8200',
    token: process.env.VAULT_TOKEN
  });

  // Step 2 — get temporary AWS credentials
  const result = await client.read('aws/creds/my-app-role');
  const { access_key, secret_key, security_token } = result.data;

  // Step 3 — configure AWS SDK
  AWS.config.update({
    accessKeyId: access_key,
    secretAccessKey: secret_key,
    sessionToken: security_token,
    region: 'us-east-1'
  });

  // use any AWS service
  const s3 = new AWS.S3();
  const buckets = await s3.listBuckets().promise();
  console.log(buckets);
}

getAWSCredentials();
```

---

### GitHub Actions workflow

```yaml
name: Deploy with Vault AWS Creds

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Get Vault token via OIDC
        uses: hashicorp/vault-action@v3
        id: vault
        with:
          url: https://44.215.126.96:8200
          tlsSkipVerify: true
          method: jwt
          role: github-actions
          exportToken: true   # exports VAULT_TOKEN env var

      - name: Get temporary AWS credentials from Vault
        run: |
          CREDS=$(vault read -format=json aws/creds/my-app-role)
          echo "AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.data.access_key')" >> $GITHUB_ENV
          echo "AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.data.secret_key')" >> $GITHUB_ENV
          echo "AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.data.security_token')" >> $GITHUB_ENV

      - name: Use AWS credentials
        run: aws s3 ls --region us-east-1
```

---

### Docker container (via entrypoint script)

```dockerfile
FROM python:3.11-slim

# install vault CLI
RUN apt-get update && apt-get install -y curl unzip && \
    curl -fsSL https://releases.hashicorp.com/vault/1.21.4/vault_1.21.4_linux_amd64.zip \
    -o vault.zip && unzip vault.zip && mv vault /usr/local/bin/

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/bash
# entrypoint.sh — runs before your app starts

# authenticate to Vault using AppRole
VAULT_TOKEN=$(vault write -format=json auth/approle/login \
  role_id="$ROLE_ID" \
  secret_id="$SECRET_ID" | jq -r '.auth.client_token')

export VAULT_TOKEN

# get AWS credentials
CREDS=$(vault read -format=json aws/creds/my-app-role)
export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.data.access_key')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.data.secret_key')
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.data.security_token')

# start your application
exec python app.py
```

---

## 8. Credential Types

| Type | How it works | Best for |
|---|---|---|
| `assumed_role` | Vault assumes an existing IAM role via STS | Production — cleaner, uses existing roles |
| `iam_user` | Vault creates a real IAM user + access key | When you can't use assumed roles |
| `federation_token` | Vault calls GetFederationToken | Legacy, limited use |

### assumed_role (this setup)

```bash
vault write aws/roles/my-app-role \
  credential_type=assumed_role \
  role_arns="arn:aws:iam::xxxxxxxxxxxx:role/vault-server-role" \
  default_sts_ttl="1h" \
  max_sts_ttl="4h"
```

### iam_user (alternative)

```bash
vault write aws/roles/s3-reader \
  credential_type=iam_user \
  policy_document=-<<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": "*"
  }]
}
EOF
```

---

## 9. Managing Leases

### List active leases

```bash
vault list sys/leases/lookup/aws/creds/my-app-role
```

### Renew a lease

```bash
vault lease renew aws/creds/my-app-role/LEASE_ID
```

### Revoke a specific lease (immediately invalidate credentials)

```bash
vault lease revoke aws/creds/my-app-role/LEASE_ID
```

### Revoke ALL credentials for a role

```bash
vault lease revoke -prefix aws/creds/my-app-role
```

> This is your emergency kill switch — instantly invalidates all active
> credentials for a role if you suspect a leak.

---

## 10. Troubleshooting

| Error | Root Cause | Fix |
|---|---|---|
| `AccessDenied: sts:AssumeRole` | Role missing `sts:AssumeRole` permission | Add to `vault-sts-policy` |
| `not authorized to perform: sts:AssumeRole on resource` | Role cannot assume itself | Update trust policy to allow self-assumption |
| `InvalidClientTokenId` | Missing `AWS_SESSION_TOKEN` | Always export `security_token` for assumed role creds |
| `ExpiredTokenException` | Credential TTL expired | Request fresh credentials from Vault |
| `no handler for route` | AWS secrets engine not enabled | Run `vault secrets enable aws` |
| `role not found` | Wrong role name | Check `vault list aws/roles` |

---

## 11. Production Checklist

- [ ] Create a dedicated IAM role per application — not shared `vault-server-role`
- [ ] Set minimum required permissions on each role — not `Resource: "*"`
- [ ] Set `default_sts_ttl` to the minimum your app needs
- [ ] Implement credential renewal in your app before TTL expires
- [ ] Use `vault lease revoke -prefix` as emergency kill switch if credentials leak
- [ ] Enable Vault audit logging to track all credential generation
- [ ] Use separate Vault roles per environment (dev/staging/prod)
- [ ] Never log `security_token` — it's as sensitive as the secret key

---

## Secrets Engines Enabled

| Engine | Path | Status |
|---|---|---|
| KV v2 | `secret/` | ✅ configured |
| AWS | `aws/` | ✅ configured |
| PKI | `pki/` | 🔜 next |
| Database | `database/` | 🔜 next |
| SSH | `ssh/` | 🔜 next |
| Transit | `transit/` | 🔜 next |

---

*Practice setup — vault-node-1 — AWS account: xxxxxxxxxxxx*