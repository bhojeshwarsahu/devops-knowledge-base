# 06 — Raft Snapshot Backup

> *"A backup you have never tested is not a backup — it is a hope. Test your restore before you need it."*

---

## What is a Raft Snapshot?

A Raft snapshot captures the **complete state of your Vault cluster** into a single portable file. This includes everything:

| What is captured | Details |
|---|---|
| All secrets | Every KV secret and all versions |
| Policies | admin, readonly, and any custom policies |
| Auth methods | userpass, AppRole, and any others enabled |
| Users | All userpass users and their policy attachments |
| Secrets engines | All enabled engines and their configurations |
| Audit devices | Audit log configuration |
| Encryption keys | Vault's internal keyring |
| Cluster config | Raft cluster membership and state |

If your cluster is corrupted, a node is lost, or you accidentally delete something critical — this snapshot is what brings everything back.

---

## Prerequisites

- Vault cluster initialized, unsealed, and fully configured
- Root token or token with `sys/storage/raft/snapshot` read capability
- SSH access to vault-node-1 (the active leader)
- Snapshots must always be taken from the **leader node**

---

## Step 1 — Create snapshot directory

```bash
sudo mkdir -p /opt/vault/snapshots
sudo chown ubuntu:ubuntu /opt/vault/snapshots
ls -lad /opt/vault/snapshots/
```

Expected output:

```
drwxr-xr-x 2 ubuntu ubuntu 4096 Mar 17 08:00 /opt/vault/snapshots/
```

---

## Step 2 — Set environment variables

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<your-root-token>"
```

---

## Step 3 — Take a manual snapshot

```bash
vault operator raft snapshot save \
  /opt/vault/snapshots/vault-snapshot-$(date +%Y%m%d-%H%M%S).snap
```

No output means success. Verify the file was created:

```bash
ls -lh /opt/vault/snapshots/
```

Expected output:

```
-rw------- 1 ubuntu ubuntu 30K Mar 17 08:09 vault-snapshot-20260317-080942.snap
```

---

## Step 4 — Inspect the snapshot

Verify the snapshot is valid and see what it contains:

```bash
vault operator raft snapshot inspect \
  /opt/vault/snapshots/vault-snapshot-20260317-080942.snap
```

Expected output:

```
 ID           bolt-snapshot
 Size         30549
 Index        93
 Term         4
 Version      1

 Key Name                                          Count      Size
 ----                                              ----       ----
 sys/token                                         9          3.6KB
 sys/policy                                        5          5.1KB
 sys/expire                                        3          3.9KB
 core/auth                                         1          425B
 core/audit                                        1          281B
 core/mounts                                       1          592B
 core/keyring                                      1          337B
 core/seal-config                                  1          146B
 core/raft                                         2          1.9KB
 ----                                              ----
 Total Size                                                29.7KB
```

### What each key means

| Key | What it contains |
|---|---|
| `sys/token` | All tokens and their metadata |
| `sys/policy` | All policies (admin, readonly, default) |
| `sys/expire` | Lease expiration data |
| `core/auth` | Auth method configurations |
| `core/audit` | Audit device configuration |
| `core/mounts` | Secrets engine mount configurations |
| `core/keyring` | Vault's internal encryption keys |
| `core/seal-config` | Unseal configuration |
| `core/raft` | Raft cluster membership data |
| `logical/*` | Actual secret data from KV engines |

---

## Step 5 — Automate with a cron job

Taking snapshots manually is error-prone — you will forget. Set up a daily automated snapshot.

### Create the snapshot script

```bash
sudo tee /opt/vault/snapshots/snapshot.sh <<'EOF'
#!/bin/bash

export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<your-root-token>"

SNAPSHOT_DIR="/opt/vault/snapshots"
SNAPSHOT_FILE="$SNAPSHOT_DIR/vault-snapshot-$(date +%Y%m%d-%H%M%S).snap"
LOG_FILE="/var/log/vault/snapshot.log"

# Take snapshot
vault operator raft snapshot save "$SNAPSHOT_FILE"

if [ $? -eq 0 ]; then
  echo "$(date '+%Y-%m-%d %H:%M:%S') SUCCESS: Snapshot saved to $SNAPSHOT_FILE" >> "$LOG_FILE"
else
  echo "$(date '+%Y-%m-%d %H:%M:%S') ERROR: Snapshot failed" >> "$LOG_FILE"
  exit 1
fi

# Keep only last 7 snapshots — delete older ones
ls -t $SNAPSHOT_DIR/*.snap | tail -n +8 | xargs -r rm

# Log how many snapshots exist
COUNT=$(ls $SNAPSHOT_DIR/*.snap 2>/dev/null | wc -l)
echo "$(date '+%Y-%m-%d %H:%M:%S') INFO: $COUNT snapshots retained in $SNAPSHOT_DIR" >> "$LOG_FILE"
EOF

# Set correct permissions
sudo chmod +x /opt/vault/snapshots/snapshot.sh
sudo chown ubuntu:ubuntu /opt/vault/snapshots/snapshot.sh
```

### Test the script manually first

```bash
/opt/vault/snapshots/snapshot.sh

# Verify it worked
ls -lh /opt/vault/snapshots/
cat /var/log/vault/snapshot.log
```

Expected log output:

```
2026-03-17 08:30:00 SUCCESS: Snapshot saved to /opt/vault/snapshots/vault-snapshot-20260317-083000.snap
2026-03-17 08:30:00 INFO: 2 snapshots retained in /opt/vault/snapshots
```

### Add to crontab

```bash
crontab -e
```

Add this line at the bottom:

```
0 2 * * * /opt/vault/snapshots/snapshot.sh
```

This runs the snapshot every day at 2:00 AM.

### Verify crontab is set

```bash
crontab -l
```

---

## Step 6 — Copy snapshot off the server

Never rely on a single copy of your backup on the same server. If the server is lost, the backup is lost with it.

### Copy to your local machine

```bash
# Run this from your LOCAL machine
mkdir -p ~/vault-backups

scp -i ~/.ssh/pvt-key.pem \
  ubuntu@<Node-1-Public-IP>:/opt/vault/snapshots/*.snap \
  ~/vault-backups/

ls -lh ~/vault-backups/
```

### Copy to AWS S3 (production-grade)

```bash
# Install AWS CLI on Node 1
sudo apt install awscli -y

# Configure AWS credentials
aws configure

# Copy snapshot to S3
aws s3 cp /opt/vault/snapshots/vault-snapshot-$(date +%Y%m%d)*.snap \
  s3://your-bucket-name/vault-snapshots/

# Verify it uploaded
aws s3 ls s3://your-bucket-name/vault-snapshots/
```

### Automate S3 upload in the snapshot script

Add this to the end of `snapshot.sh` after the snapshot is taken:

```bash
# Upload to S3
aws s3 cp "$SNAPSHOT_FILE" s3://your-bucket-name/vault-snapshots/
echo "$(date '+%Y-%m-%d %H:%M:%S') INFO: Snapshot uploaded to S3" >> "$LOG_FILE"
```

---

## Snapshot file naming convention

The naming pattern `vault-snapshot-YYYYMMDD-HHMMSS.snap` makes it easy to identify when each snapshot was taken:

```
vault-snapshot-20260317-080942.snap   ← 2026 Mar 17 at 08:09:42
vault-snapshot-20260318-020000.snap   ← 2026 Mar 18 at 02:00:00 (cron)
vault-snapshot-20260319-020000.snap   ← 2026 Mar 19 at 02:00:00 (cron)
```

Always keep at least 7 days of snapshots so you can roll back to any point in the past week.

---

## Checking snapshot health

```bash
# List all snapshots with sizes and dates
ls -lh /opt/vault/snapshots/

# Check the most recent snapshot
vault operator raft snapshot inspect \
  $(ls -t /opt/vault/snapshots/*.snap | head -1)

# Check snapshot log
cat /var/log/vault/snapshot.log
```

---

## Common issues

| Issue | Cause | Fix |
|---|---|---|
| `permission denied` saving snapshot | Wrong directory ownership | `sudo chown ubuntu:ubuntu /opt/vault/snapshots` |
| `Error taking the snapshot` | Token lacks snapshot permission | Use root token or add `sys/storage/raft/snapshot` read to policy |
| Snapshot file is 0 bytes | Vault was sealed during snapshot | Unseal Vault first, then retry |
| Cron job not running | Wrong crontab user | Run `crontab -e` as ubuntu user, not root |
| Old snapshots not deleted | Script not executable | `chmod +x /opt/vault/snapshots/snapshot.sh` |

---

## Quick reference

```bash
# Take a snapshot
vault operator raft snapshot save /opt/vault/snapshots/vault-snapshot-$(date +%Y%m%d-%H%M%S).snap

# Inspect a snapshot
vault operator raft snapshot inspect /opt/vault/snapshots/<filename>.snap

# List snapshots
ls -lh /opt/vault/snapshots/

# Copy snapshot to local machine (run from local)
scp -i ~/.ssh/pvt-key.pem ubuntu@<public-ip>:/opt/vault/snapshots/*.snap ~/vault-backups/

# Copy to S3
aws s3 cp /opt/vault/snapshots/<filename>.snap s3://your-bucket/vault-snapshots/

# View snapshot log
cat /var/log/vault/snapshot.log

# Check crontab
crontab -l
```

---

## Snapshot best practices

- Always take a snapshot **before and after** any major change (enabling a new engine, changing policies, adding users)
- Keep snapshots in **at least 2 locations** — local disk and S3 or another server
- Retain at least **7 daily snapshots** so you can roll back to any point in the past week
- Always take snapshots from the **leader node** — follower snapshots may be slightly behind
- Test your restore procedure at least once so you know it works before you actually need it
- Store the snapshot token securely — the snapshot script contains your root token, set file permissions to `700`

---

## Next step

Snapshots are configured and automated. Now learn how to restore from a snapshot if disaster strikes.

Proceed to **[07-restore.md](./07-restore.md)** — restoring Vault from a Raft snapshot.

---

*Part of the [Vault HA Production Setup Guide](./Vault-HA-Production-Setup-Guide.md)*