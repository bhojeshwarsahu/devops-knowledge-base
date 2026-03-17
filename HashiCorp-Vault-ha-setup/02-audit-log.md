# 02 — Vault Audit Log

> *"You cannot secure what you cannot see. The audit log is your eyes inside Vault — every request, every response, every failure, recorded forever."*

---

## What is the Vault Audit Log?

The audit log is a detailed, tamper-evident record of **every request and response** that goes through Vault. Every login attempt, every secret read, every policy change, every token creation — all of it is written to the audit log in JSON format.

### Why enable it first?

Audit logging must be enabled **before** anything else because:

- It captures all actions from the moment it is turned on
- Actions performed before enabling the audit log are never recorded
- If something goes wrong later, you need the full history from day one
- It is a compliance requirement in most security frameworks (SOC2, ISO27001, PCI-DSS)

### What does it capture?

Every audit log entry contains:

| Field | Description |
|---|---|
| `time` | Exact timestamp of the request |
| `type` | request or response |
| `auth.display_name` | Who made the request |
| `auth.policies` | Policies attached to the token |
| `request.operation` | read, write, delete, list |
| `request.path` | Which Vault path was accessed |
| `response.data` | Response data (secrets are hashed) |
| `error` | Error message if request failed |

> Note: Vault never writes raw secret values to the audit log. Secret values are HMAC-hashed before logging, so the log is safe to store without exposing your secrets.

---

## Prerequisites

- Vault cluster initialized and unsealed
- Root token or token with `sudo` capability on `sys/audit*`
- SSH access to vault-node-1 (the active leader)

---

## Step 1 — Create audit log directory

```bash
sudo mkdir -p /var/log/vault
sudo chown vault:vault /var/log/vault
sudo chmod 750 /var/log/vault
```

---

## Step 2 — Set environment variables

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<your-root-token>"
```

Add these to `~/.bashrc` to persist across sessions:

```bash
echo 'export VAULT_ADDR="https://127.0.0.1:8200"' >> ~/.bashrc
echo 'export VAULT_CACERT="/etc/vault.d/tls/ca.crt"' >> ~/.bashrc
source ~/.bashrc
```

---

## Step 3 — Enable the file audit device

```bash
vault audit enable file file_path=/var/log/vault/audit.log
```

Expected output:

```
Success! Enabled the file audit device at: file/
```

---

## Step 4 — Verify audit log is enabled

```bash
vault audit list
```

Expected output:

```
Path     Type    Description
----     ----    -----------
file/    file    n/a
```

---

## Step 5 — Confirm logs are being written

```bash
# Perform any vault action to generate a log entry
vault status

# Check the log file
sudo tail -f /var/log/vault/audit.log
```

You will see JSON entries streaming in. Press `Ctrl+C` to stop following.

Sample log entry (formatted for readability):

```json
{
  "time": "2026-03-17T08:00:00Z",
  "type": "request",
  "auth": {
    "display_name": "root",
    "policies": ["root"],
    "token_type": "service"
  },
  "request": {
    "id": "abc-123",
    "operation": "read",
    "path": "sys/health"
  }
}
```

---

## Step 6 — Enable a second audit device (recommended)

Always enable at least two audit devices. If Vault cannot write to any audit device, it **stops serving requests entirely** — this is by design for security. A second device acts as a failover.

```bash
# Enable syslog as a second audit device
vault audit enable -path=syslog syslog
```

Verify both are active:

```bash
vault audit list
```

Expected output:

```
Path      Type    Description
----      ----    -----------
file/     file    n/a
syslog/   syslog  n/a
```

---

## Important: What happens if audit log fills up disk?

If `/var/log/vault/audit.log` fills up the disk and Vault cannot write to it, **Vault will stop serving all requests**. This is intentional — Vault refuses to operate without a working audit trail.

To prevent this, set up log rotation.

---

## Step 7 — Configure log rotation

```bash
sudo tee /etc/logrotate.d/vault <<'EOF'
/var/log/vault/audit.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
    copytruncate
    dateext
    dateformat -%Y%m%d
}
EOF
```

Test log rotation config:

```bash
sudo logrotate -d /etc/logrotate.d/vault
```

---

## Querying the audit log

The audit log is newline-delimited JSON. You can query it with `jq`:

```bash
# Install jq if not present
sudo apt install jq -y

# Show all write operations
sudo cat /var/log/vault/audit.log | jq '. | select(.request.operation == "create")'

# Show all failed requests
sudo cat /var/log/vault/audit.log | jq '. | select(.error != null)'

# Show all logins
sudo cat /var/log/vault/audit.log | jq '. | select(.request.path | startswith("auth/"))'

# Show requests by a specific user
sudo cat /var/log/vault/audit.log | jq '. | select(.auth.display_name == "admin-user")'

# Count requests per path
sudo cat /var/log/vault/audit.log | jq -r '.request.path' | sort | uniq -c | sort -rn
```

---

## Disabling an audit device

You should never disable your only audit device. If you need to disable one:

```bash
# Only safe if you have another audit device active
vault audit disable file/
```

---

## Common issues

| Issue | Cause | Fix |
|---|---|---|
| `permission denied` on log file | vault user cannot write to log dir | `sudo chown vault:vault /var/log/vault` |
| Vault stops serving requests | Audit log disk full | Clear space or rotate logs |
| No log entries appearing | Wrong file path in config | Check `vault audit list` and verify path |
| Log file growing too fast | High request volume | Enable log rotation |

---

## Quick reference

```bash
# Enable audit log
vault audit enable file file_path=/var/log/vault/audit.log

# List audit devices
vault audit list

# Watch live audit log
sudo tail -f /var/log/vault/audit.log

# Watch formatted (requires jq)
sudo tail -f /var/log/vault/audit.log | jq .

# Disable audit device
vault audit disable file/
```

---

## Next step

Audit logging is now active. Every action from this point forward is recorded.

Proceed to **[03-policies.md](./03-policies.md)** — creating admin and readonly policies.

---

*Part of the [Vault HA Production Setup Guide](./Vault-HA-Production-Setup-Guide.md)*