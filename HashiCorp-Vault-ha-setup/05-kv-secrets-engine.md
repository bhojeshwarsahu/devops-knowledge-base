# 05 — KV Secrets Engine

> *"A secret that is never rotated, never audited, and never expired is not a secret — it is a liability waiting to be discovered."*

---

## What is the KV Secrets Engine?

The KV (Key-Value) secrets engine is the most commonly used secrets engine in Vault. It stores arbitrary secrets as key-value pairs — database passwords, API keys, TLS certificates, configuration values, or any sensitive string your applications need.

### KV v1 vs KV v2

| Feature | KV v1 | KV v2 |
|---|---|---|
| Versioning | No | Yes — stores up to 10 versions |
| Soft delete | No | Yes — deleted secrets are recoverable |
| Metadata | No | Yes — created time, updated time, custom metadata |
| Check-and-set | No | Yes — prevent accidental overwrites |
| Path | `secret/foo` | `secret/data/foo` (internally) |

Always use **KV v2** — it is strictly better and has no downsides for practice or production.

---

## How KV v2 stores data internally

When you mount KV v2 at `secret/`, Vault creates two internal sub-paths:

```
secret/data/<path>      ← actual secret data lives here
secret/metadata/<path>  ← version history and metadata lives here
```

This is why the readonly policy uses `+/data/*` instead of `+/*` — to target the actual secret data specifically.

---

## Prerequisites

- Vault cluster initialized and unsealed
- Audit log enabled (see `02-audit-log.md`)
- Policies created (see `03-policies.md`)
- Auth methods enabled (see `04-auth-methods.md`)
- Root or admin token

---

## Step 1 — Set environment variables

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<your-root-token>"
```

---

## Step 2 — Enable KV v2 secrets engine

```bash
vault secrets enable -path=secret kv-v2
```

Expected output:

```
Success! Enabled the kv-v2 secrets engine at: secret/
```

---

## Step 3 — Verify it is enabled

```bash
vault secrets list
```

Expected output:

```
Path          Type         Accessor               Description
----          ----         --------               -----------
cubbyhole/    cubbyhole    cubbyhole_xxxxxxxx     per-token private secret storage
identity/     identity     identity_xxxxxxxx      identity store
secret/       kv           kv_xxxxxxxx            n/a
sys/          system       system_xxxxxxxx        system endpoints used for control
```

---

## Step 4 — Write your first secret

```bash
vault kv put secret/myapp/config \
  db_password="supersecret123" \
  api_key="abc-xyz-789"
```

Expected output:

```
======= Secret Path =======
secret/data/myapp/config

======= Metadata =======
Key              Value
---              -----
created_time     2026-03-17T08:00:00.000000000Z
custom_metadata  <nil>
deletion_time    n/a
destroyed        false
version          1
```

---

## Step 5 — Read the secret back

```bash
vault kv get secret/myapp/config
```

Expected output:

```
====== Secret Path ======
secret/data/myapp/config

======= Metadata =======
Key              Value
---              -----
created_time     2026-03-17T08:00:00Z
version          1

====== Data ======
Key             Value
---             -----
api_key         abc-xyz-789
db_password     supersecret123
```

### Read a specific field only

```bash
vault kv get -field=db_password secret/myapp/config
```

Output:

```
supersecret123
```

### Read as JSON

```bash
vault kv get -format=json secret/myapp/config
```

---

## Step 6 — Update a secret (creates a new version)

```bash
vault kv put secret/myapp/config \
  db_password="newpassword456" \
  api_key="abc-xyz-789"
```

Every `vault kv put` creates a new version. The old version is preserved and recoverable.

---

## Step 7 — Check version history

```bash
vault kv metadata get secret/myapp/config
```

Expected output:

```
======== Metadata ========
Key                     Value
---                     -----
cas_required            false
created_time            2026-03-17T08:00:00Z
current_version         2
delete_version_after    0s
max_versions            0
oldest_version          0
updated_time            2026-03-17T08:05:00Z

====== Version 1 ======
Key              Value
---              -----
created_time     2026-03-17T08:00:00Z
deletion_time    n/a
destroyed        false

====== Version 2 ======
Key              Value
---              -----
created_time     2026-03-17T08:05:00Z
deletion_time    n/a
destroyed        false
```

---

## Step 8 — Read a specific old version

```bash
# Read version 1 (the original)
vault kv get -version=1 secret/myapp/config
```

This is extremely useful when you need to roll back to a previous secret value after a rotation.

---

## Step 9 — Patch a secret (update only specific fields)

Unlike `kv put` which replaces the entire secret, `kv patch` updates only the fields you specify:

```bash
# Only update db_password, keep api_key unchanged
vault kv patch secret/myapp/config \
  db_password="patchedpassword789"

# Verify — api_key should still be abc-xyz-789
vault kv get secret/myapp/config
```

---

## Working with secret paths

Organize secrets by application, environment, or team:

```bash
# Application secrets
vault kv put secret/myapp/database \
  host="db.internal" \
  port="5432" \
  username="appuser" \
  password="dbpass123"

vault kv put secret/myapp/redis \
  host="redis.internal" \
  port="6379" \
  password="redispass456"

# Environment-specific secrets
vault kv put secret/production/database \
  password="prod-super-secret"

vault kv put secret/staging/database \
  password="staging-secret"

# List all secrets under a path
vault kv list secret/myapp/
vault kv list secret/
```

---

## Deleting secrets

### Soft delete (recoverable)

```bash
# Delete latest version — data is hidden but recoverable
vault kv delete secret/myapp/config

# Verify — shows deletion_time but data is not gone
vault kv metadata get secret/myapp/config

# Recover the deleted version
vault kv undelete -versions=2 secret/myapp/config
```

### Hard delete (permanent)

```bash
# Permanently destroy a specific version — cannot be recovered
vault kv destroy -versions=1 secret/myapp/config

# Delete all versions and metadata permanently
vault kv metadata delete secret/myapp/config
```

---

## Set max versions per secret

By default KV v2 keeps all versions. Set a limit to control storage:

```bash
# Keep only last 5 versions for all secrets under this engine
vault write secret/config max_versions=5

# Set max versions for a specific secret path only
vault kv metadata put -max-versions=5 secret/myapp/config
```

---

## Enable Check-and-Set (CAS)

CAS prevents accidental overwrites. With CAS enabled, you must provide the current version number when writing:

```bash
# Enable CAS for a specific secret
vault kv metadata put -cas-required=true secret/myapp/config

# Now writes require -cas flag with current version
vault kv put -cas=2 secret/myapp/config db_password="newvalue"

# Without -cas it will fail:
# Error: check-and-set parameter required for this call
```

---

## Enabling multiple KV engines

You can mount multiple KV engines at different paths:

```bash
# Mount a second KV engine for infrastructure secrets
vault secrets enable -path=infra kv-v2

# Mount a third for team-specific secrets
vault secrets enable -path=team kv-v2

# List all mounted engines
vault secrets list
```

---

## Common issues

| Issue | Cause | Fix |
|---|---|---|
| `no handler for route` | Wrong path format | Use `secret/myapp/key` not `secret/data/myapp/key` in CLI |
| `permission denied` | Policy does not cover the path | Check policy has `+/data/*` or `secret/*` |
| Secret shows as deleted | `kv delete` was run | Use `kv undelete` to recover |
| Version not found | Version was destroyed | Destroyed versions cannot be recovered |
| `cas_required` error | CAS enabled but no `-cas` flag | Provide `-cas=<version>` in the write command |

---

## Quick reference

```bash
# Enable KV v2
vault secrets enable -path=secret kv-v2

# Write a secret
vault kv put secret/myapp/config key="value"

# Read a secret
vault kv get secret/myapp/config

# Read specific field
vault kv get -field=key secret/myapp/config

# Read as JSON
vault kv get -format=json secret/myapp/config

# Read a specific version
vault kv get -version=1 secret/myapp/config

# Patch (partial update)
vault kv patch secret/myapp/config key="newvalue"

# List secrets
vault kv list secret/myapp/

# View version history
vault kv metadata get secret/myapp/config

# Soft delete (recoverable)
vault kv delete secret/myapp/config

# Undelete
vault kv undelete -versions=1 secret/myapp/config

# Hard delete (permanent)
vault kv destroy -versions=1 secret/myapp/config

# Delete all versions and metadata
vault kv metadata delete secret/myapp/config
```

---

## KV v2 best practices

- Organize secrets by application and environment: `secret/<app>/<environment>/<type>`
- Use `kv patch` instead of `kv put` when updating a single field — avoids accidentally overwriting other fields
- Set `max_versions` to control storage growth in production
- Enable CAS for critical secrets like database passwords to prevent accidental overwrites
- Never store secrets in environment variables or config files — always fetch from Vault at runtime
- Rotate secrets regularly and use versioning to track rotation history

---

## Next step

KV secrets engine is configured and tested. Now protect your cluster state with regular backups.

Proceed to **[06-raft-snapshot.md](./06-raft-snapshot.md)** — taking and automating Raft snapshots.

---

*Part of the [Vault HA Production Setup Guide](./Vault-HA-Production-Setup-Guide.md)*