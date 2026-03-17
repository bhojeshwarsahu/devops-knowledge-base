# 03 — Vault Policies

> *"In Vault, silence is denial. If a policy does not explicitly allow something, it is forbidden. Design your policies with the minimum access needed — nothing more."*

---

## What is a Vault Policy?

A policy is a set of rules that defines **what an authenticated identity can do** in Vault. Without a policy attached, a token has zero permissions — it cannot read, write, or list anything.

This is the **deny by default** model — every action is denied unless explicitly permitted by a policy.

### AWS equivalent

| Vault concept | AWS equivalent |
|---|---|
| Vault Policy | IAM Policy |
| Vault Token | IAM Role / User |
| Vault Path | AWS Resource ARN |
| Capabilities | IAM Actions (Allow/Deny) |

---

## Capabilities

Each policy rule defines a path and a list of capabilities (permissions):

| Capability | HTTP method | What it allows |
|---|---|---|
| `create` | POST/PUT | Create new secrets or config |
| `read` | GET | Read secrets or config |
| `update` | POST/PUT | Update existing secrets or config |
| `delete` | DELETE | Delete secrets or config |
| `list` | LIST | List keys at a path |
| `sudo` | any | Access root-protected paths |
| `deny` | any | Explicitly deny access (overrides all) |

---

## Policy Path Wildcards

| Wildcard | Meaning | Example |
|---|---|---|
| `*` | Matches everything including `/` | `secret/*` matches `secret/foo/bar` |
| `+` | Matches one path segment | `+/data/*` matches `secret/data/foo` |

---

## Prerequisites

- Vault cluster initialized and unsealed
- Audit log enabled (see `02-audit-log.md`)
- Root token or admin token

---

## Step 1 — Set environment variables

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<your-root-token>"
```

---

## Step 2 — Create the admin policy

```bash
vault policy write admin - <<'EOF'
# Manage all auth methods
path "auth/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Manage ALL secrets engines (any mount, current and future)
path "+/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Manage secrets engine mounts
path "sys/mounts/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# List secrets engines
path "sys/mounts" {
  capabilities = ["read", "list"]
}

# Manage policies
path "sys/policies/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# List policies
path "sys/policies/acl" {
  capabilities = ["read", "list"]
}

# Manage audit devices
path "sys/audit*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Take Raft snapshots
path "sys/storage/raft/snapshot" {
  capabilities = ["read"]
}

# Check cluster health
path "sys/health" {
  capabilities = ["read"]
}

# Manage leases
path "sys/leases/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Renew and revoke tokens
path "auth/token/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF
```

---

## Step 3 — Create the readonly policy

```bash
vault policy write readonly - <<'EOF'
# Read secrets from ANY secrets engine
path "+/data/*" {
  capabilities = ["read"]
}

# List secret paths in ANY secrets engine
path "+/metadata/*" {
  capabilities = ["list"]
}

# Check own token details
path "auth/token/lookup-self" {
  capabilities = ["read"]
}

# Renew own token
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Check own permissions
path "sys/capabilities-self" {
  capabilities = ["update"]
}
EOF
```

---

## Step 4 — Verify policies

```bash
vault policy list
```

Expected output:

```
admin
default
readonly
root
```

```bash
vault policy read admin
vault policy read readonly
```

---

## Understanding the built-in policies

| Policy | Description |
|---|---|
| `root` | Full access to everything. Use only for initial setup — never for apps or users. |
| `default` | Auto-attached to every token. Allows basic token operations. |
| `admin` | Your custom admin policy — full management access. |
| `readonly` | Your custom readonly policy — read secrets only. |

---

## Why `+/*` instead of `secret/*` for admin?

```hcl
# This only covers KV engine mounted at "secret/"
path "secret/*" { ... }

# This covers ALL engines at ANY mount point
path "+/*" { ... }
```

The `+` wildcard matches exactly one path segment — so `+/*` matches `secret/foo`, `aws/foo`, `database/foo`, `pki/foo`. Any engine you mount now or in the future is automatically covered. This is the correct way to write an admin policy.

---

## Testing policies

### Test admin user has write access

```bash
vault login -method=userpass username=admin-user password="StrongPassword@123"

# Should succeed
vault kv put secret/test/check value="hello"
vault kv get secret/test/check
```

### Test readonly user cannot write

```bash
vault login -method=userpass username=readonly-user password="StrongPassword@456"

# Should succeed
vault kv get secret/test/check

# Should FAIL with permission denied
vault kv put secret/test/check value="hacked"
```

### Check what a token can do on a path

```bash
vault token capabilities secret/data/myapp/config
```

---

## Updating a policy

Changes take effect immediately for all tokens using that policy.

```bash
vault policy write readonly - <<'EOF'
path "+/data/*" {
  capabilities = ["read"]
}
path "+/metadata/*" {
  capabilities = ["list"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "sys/capabilities-self" {
  capabilities = ["update"]
}
EOF
```

---

## Deleting a policy

```bash
vault policy delete readonly
```

> Warning: Deleting a policy does not revoke tokens that have it attached. Those tokens simply lose the permissions that policy granted.

---

## Common issues

| Issue | Cause | Fix |
|---|---|---|
| `permission denied` on a path | Policy does not cover that path | Add the path to the policy |
| Policy write fails | Token lacks `sys/policies/*` access | Use root or admin token |
| Changes not taking effect | Cached token | Re-login to get a fresh token |

---

## Quick reference

```bash
# Write policy from file
vault policy write my-policy ./my-policy.hcl

# Write policy inline
vault policy write my-policy - <<'EOF'
path "secret/*" { capabilities = ["read"] }
EOF

# List all policies
vault policy list

# Read a policy
vault policy read my-policy

# Delete a policy
vault policy delete my-policy

# Check token capabilities on a path
vault token capabilities secret/data/myapp/config
```

---

## Policy best practices

- Follow the **principle of least privilege** — grant only what is needed
- Never use the `root` policy for applications or team members
- Create one policy per role — do not share policies across different access levels
- Test every policy with both allowed and denied actions
- Keep policy files in version control (Git)
- Regularly audit which tokens have which policies with `vault token lookup`

---

## Next step

Policies are defined. Now you need a way for your team to authenticate and receive tokens with these policies attached.

Proceed to **[04-auth-methods.md](./04-auth-methods.md)** — enabling userpass authentication.

---

*Part of the [Vault HA Production Setup Guide](./Vault-HA-Production-Setup-Guide.md)*