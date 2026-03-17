# 07 — Vault Restore from Raft Snapshot

> *"The true value of a backup is not measured when you take it — it is measured when you need it. Know your restore procedure before disaster strikes."*

---

## What does a restore do?

A Raft snapshot restore **replaces the entire Vault state** with the state captured in the snapshot file. After a restore:

- All secrets are exactly as they were at snapshot time
- All policies, auth methods, and users are restored
- All secrets engine configurations are restored
- Any changes made **after** the snapshot was taken are lost

> This is why regular snapshots and off-server backups are critical — the snapshot is your point-in-time recovery target.

---

## When to restore

| Scenario | Action |
|---|---|
| Accidentally deleted a secret | Restore from last snapshot |
| Accidentally deleted a policy or user | Restore from last snapshot |
| Cluster data corruption | Restore from last snapshot |
| All nodes lost (catastrophic failure) | Rebuild cluster, then restore |
| Testing restore procedure | Follow Scenario 2 below |

---

## Prerequisites

- A valid snapshot file (from `06-raft-snapshot.md`)
- Unseal keys (all 5, you need any 3)
- Root token
- SSH access to all 3 nodes

---

## Scenario 1 — Cluster is running, restore to fix bad state

Use this when your cluster is still running but you need to roll back to a previous state — for example, you accidentally deleted secrets or policies.

### Step 1 — Confirm you are on the leader node

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<your-root-token>"

vault status | grep "HA Mode"
```

Expected output:

```
HA Mode        active
```

If it says `standby`, SSH into the correct leader node first.

### Step 2 — Take a fresh snapshot before restoring

Always snapshot the current state before overwriting it — gives you a fallback:

```bash
vault operator raft snapshot save \
  /opt/vault/snapshots/vault-pre-restore-$(date +%Y%m%d-%H%M%S).snap

ls -lh /opt/vault/snapshots/
```

### Step 3 — Stop Vault on all 3 nodes

```bash
# On vault-node-1, vault-node-2, vault-node-3
sudo systemctl stop vault
```

### Step 4 — Restore on vault-node-1 ONLY

```bash
vault operator raft snapshot restore \
  /opt/vault/snapshots/vault-snapshot-YYYYMMDD-HHMMSS.snap
```

Expected output:

```
Success! Vault data has been restored from the snapshot.
```

> Vault automatically seals itself after a restore. This is expected and correct.

### Step 5 — Start Vault on all 3 nodes

```bash
# On vault-node-1, vault-node-2, vault-node-3
sudo systemctl start vault
```

### Step 6 — Unseal all 3 nodes

Run on **each node** — provide any 3 of your 5 unseal keys:

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"

vault operator unseal <Unseal-Key-1>
vault operator unseal <Unseal-Key-2>
vault operator unseal <Unseal-Key-3>
```

### Step 7 — Verify cluster recovered

```bash
export VAULT_TOKEN="<your-root-token>"

vault status
vault operator raft list-peers
vault policy list
vault auth list
vault secrets list
vault kv get secret/myapp/config
```

---

## Scenario 2 — Complete disaster recovery (fresh servers)

Use this when all servers are lost and you are rebuilding from scratch.

### Step 1 — Rebuild all 3 servers

Follow `01-cluster-setup.md` to launch 3 new EC2 instances, install Vault, copy TLS certificates, and configure `vault.hcl`.

### Step 2 — Copy snapshot to new Node 1

From your local machine:

```bash
scp -i ~/.ssh/pvt-key.pem \
  ~/vault-backups/vault-snapshot-YYYYMMDD-HHMMSS.snap \
  ubuntu@<New-Node-1-Public-IP>:/opt/vault/snapshots/
```

Or pull from S3 directly on the server:

```bash
aws s3 cp \
  s3://your-bucket-name/vault-snapshots/vault-snapshot-YYYYMMDD-HHMMSS.snap \
  /opt/vault/snapshots/
```

### Step 3 — Start Vault on all 3 nodes

```bash
sudo systemctl start vault
```

### Step 4 — Initialize vault-node-1 ONLY

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"

vault operator init \
  -key-shares=5 \
  -key-threshold=3
```

> Save the new unseal keys and root token securely. These are different from your original ones.

### Step 5 — Unseal all 3 nodes with new unseal keys

```bash
vault operator unseal <New-Unseal-Key-1>
vault operator unseal <New-Unseal-Key-2>
vault operator unseal <New-Unseal-Key-3>
```

### Step 6 — Restore snapshot on vault-node-1

```bash
export VAULT_TOKEN="<new-root-token>"

vault operator raft snapshot restore \
  /opt/vault/snapshots/vault-snapshot-YYYYMMDD-HHMMSS.snap
```

### Step 7 — Restart and unseal all nodes again

Vault seals itself after restore — restart and unseal:

```bash
# Restart on all 3 nodes
sudo systemctl restart vault

# Unseal all 3 nodes again with new keys
vault operator unseal <New-Unseal-Key-1>
vault operator unseal <New-Unseal-Key-2>
vault operator unseal <New-Unseal-Key-3>
```

### Step 8 — Verify full recovery

```bash
export VAULT_TOKEN="<new-root-token>"

vault status
vault operator raft list-peers
vault policy list
vault auth list
vault secrets list
vault kv get secret/myapp/config
```

---

## Full verification checklist after restore

Run this after any restore to confirm everything is back:

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<your-root-token>"

echo "=== Cluster status ==="
vault status

echo "=== Raft peers ==="
vault operator raft list-peers

echo "=== Policies ==="
vault policy list

echo "=== Auth methods ==="
vault auth list

echo "=== Secrets engines ==="
vault secrets list

echo "=== Users ==="
vault list auth/userpass/users

echo "=== Test secret read ==="
vault kv get secret/myapp/config

echo "=== Audit devices ==="
vault audit list
```

---

## Testing restore in practice (do this now)

Test a full restore before you need it in an emergency:

```bash
# Step 1 — Take snapshot of current state
vault operator raft snapshot save \
  /opt/vault/snapshots/vault-test-restore-$(date +%Y%m%d-%H%M%S).snap

# Step 2 — Make a change you will undo
vault kv put secret/restore-test value="this-should-disappear"
vault kv get secret/restore-test

# Step 3 — Stop all nodes
sudo systemctl stop vault   # on all 3 nodes

# Step 4 — Restore the snapshot on Node 1
vault operator raft snapshot restore \
  /opt/vault/snapshots/vault-test-restore-*.snap

# Step 5 — Start all nodes
sudo systemctl start vault  # on all 3 nodes

# Step 6 — Unseal all nodes
vault operator unseal <key-1>
vault operator unseal <key-2>
vault operator unseal <key-3>

# Step 7 — Verify test secret is gone
vault kv get secret/restore-test
# Expected: No value found at secret/data/restore-test

echo "Restore test PASSED"
```

---

## Important rules about restore

| Rule | Why |
|---|---|
| Always restore on the **leader node only** | Raft replicates the restored state to followers automatically |
| Always **stop all nodes** before restoring | Prevents split-brain where nodes have conflicting state |
| Always **unseal after restore** | Vault seals itself after a restore by design |
| Take a **pre-restore snapshot** before restoring | Gives you a fallback if the restore makes things worse |
| Restore only from a **verified snapshot** | Use `vault operator raft snapshot inspect` to confirm validity first |

---

## Common issues

| Issue | Cause | Fix |
|---|---|---|
| `permission denied` on restore | Token lacks restore permission | Use root token |
| Vault stays sealed after restore | Expected behavior | Manually unseal with 3 keys |
| Followers not rejoining after restore | Nodes started before leader finished restore | Restart follower nodes after leader is unsealed |
| Restored data is missing secrets | Snapshot was taken before those secrets existed | Use a more recent snapshot |
| `snapshot file not found` | Wrong path or filename | Run `ls /opt/vault/snapshots/` to confirm |

---

## Quick reference

```bash
# Inspect snapshot before restoring
vault operator raft snapshot inspect /opt/vault/snapshots/<filename>.snap

# Take pre-restore safety snapshot
vault operator raft snapshot save /opt/vault/snapshots/vault-pre-restore-$(date +%Y%m%d-%H%M%S).snap

# Stop Vault on all nodes
sudo systemctl stop vault

# Restore (on leader only)
vault operator raft snapshot restore /opt/vault/snapshots/<filename>.snap

# Start Vault on all nodes
sudo systemctl start vault

# Unseal all nodes
vault operator unseal <key-1>
vault operator unseal <key-2>
vault operator unseal <key-3>

# Verify
vault status
vault operator raft list-peers
vault policy list
```

---

## Restore best practices

- Practice a full restore **at least once** so you are confident before an emergency
- Keep unseal keys and root token in a location completely separate from the servers
- After a disaster restore, immediately take a new snapshot and upload it off-server
- Document the restore procedure and share it with your team
- If restoring from an old snapshot, check what changed since that snapshot and manually re-apply anything critical that is missing

---

## Related documents

- **[06-raft-snapshot.md](./06-raft-snapshot.md)** — How to take and automate snapshots
- **[01-cluster-setup.md](./01-cluster-setup.md)** — Full cluster setup for disaster recovery rebuilds
- **[02-audit-log.md](./02-audit-log.md)** — Check audit logs to understand what happened before a restore

---

*Part of the [Vault HA Production Setup Guide](./Vault-HA-Production-Setup-Guide.md)*