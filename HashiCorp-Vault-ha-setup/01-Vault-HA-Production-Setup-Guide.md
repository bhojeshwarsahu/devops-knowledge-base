# HashiCorp Vault HA Cluster Setup Guide

A complete guide to setting up a production-grade, TLS-secured, 3-node HashiCorp Vault HA cluster on AWS using Raft integrated storage.

---

## Architecture Overview

```
                        [ Clients ]
                             |
                     [ Load Balancer ]
                    HAProxy / Nginx / ALB
                             |
          ┌──────────────────┼──────────────────┐
          |                  |                  |
   [ vault-node-1 ]   [ vault-node-2 ]   [ vault-node-3 ]
   <Node-1-Private-IP>      <Node-2-Private-IP>      <Node-3-Private-IP>
   Active/Leader        Standby            Standby
   HTTPS :8200          HTTPS :8200        HTTPS :8200
   Raft  :8201          Raft  :8201        Raft  :8201
          |                  |                  |
          └──────── mTLS Raft Replication ───────┘
```

### Why 3 Nodes?

Raft consensus requires a quorum (majority) to elect a leader:

| Nodes | Quorum | Tolerates | HA? |
|-------|--------|-----------|-----|
| 1     | 1      | 0 failures | No  |
| 2     | 2      | 0 failures | No  |
| 3     | 2      | 1 failure  | Yes |
| 5     | 3      | 2 failures | Yes |

---

## Infrastructure

### Server Details

| Node Name    | Public IP      | Private IP     | AZ          |
|--------------|----------------|----------------|-------------|
| vault-node-1 | <Node-1-Public-IP>   | <Node-1-Private-IP>  | us-east-1a  |
| vault-node-2 | <Node-2-Public-IP>     | <Node-2-Private-IP>  | us-east-1b  |
| vault-node-3 | <Node-3-Public-IP>  | <Node-3-Private-IP>   | us-east-1c  |

### Instance Specifications

- **Instance type:** m7i-flex.large
- **vCPU:** 2
- **RAM:** 8 GB
- **Storage:** 20 GB gp3 EBS
- **OS:** Ubuntu 24.04 LTS

### Security Group Rules

| Port | Protocol | Source          | Purpose                  |
|------|----------|-----------------|--------------------------|
| 22   | TCP      | Your IP/32      | SSH access               |
| 8200 | TCP      | <VPC-CIDR>   | Vault API + UI           |
| 8201 | TCP      | <VPC-CIDR>   | Raft replication         |

> Never open ports 8200 or 8201 to 0.0.0.0/0 — restrict to VPC CIDR only.

---

## TLS Communication

Vault has two channels that must be secured:

| Channel              | Port | What it carries                    |
|----------------------|------|------------------------------------|
| API / UI             | 8200 | Tokens, secrets, auth requests     |
| Cluster / Raft       | 8201 | Raft log replication, leader election |

Vault uses **mutual TLS (mTLS)** on port 8201 — both sides present certificates to verify each other's identity.

### Certificate Structure

```
Your CA (ca.crt + ca.key)
├── vault-node-1.crt  →  IP: <Node-1-Private-IP>
├── vault-node-2.crt  →  IP: <Node-2-Private-IP>
└── vault-node-3.crt  →  IP: <Node-3-Private-IP>
```

Each node's certificate must contain its own private IP as a Subject Alternative Name (SAN). The CA cert is distributed to all nodes so each node can verify the others.

---

## Step-by-Step Setup

### Phase 1 — Prepare All 3 Servers

Run on **each node**:

```bash
# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add HashiCorp repo
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install Vault
sudo apt-get update && sudo apt-get install vault -y

# Verify
vault version

# Create required directories
sudo mkdir -p /opt/vault/data
sudo mkdir -p /etc/vault.d/tls
sudo chown -R vault:vault /opt/vault/data
sudo chown -R vault:vault /etc/vault.d
```

---

### Phase 2 — Generate TLS Certificates (Local Machine)

```bash
mkdir -p ~/vault-certs && cd ~/vault-certs

# Step 1 — Create CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/C=IN/ST=Haryana/O=VaultPractice/CN=VaultCA"

# Step 2 — vault-node-1 cert (IP: <Node-1-Private-IP>)
openssl genrsa -out vault-node-1.key 2048
cat > vault-node-1.cnf <<EOF
[req]
req_extensions     = v3_req
distinguished_name = req_distinguished_name
prompt             = no
[req_distinguished_name]
CN = vault-node-1
[v3_req]
keyUsage         = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName   = @alt_names
[alt_names]
DNS.1 = vault-node-1
DNS.2 = localhost
IP.1  = <Node-1-Private-IP>
IP.2  = 127.0.0.1
EOF
openssl req -new -key vault-node-1.key -out vault-node-1.csr -config vault-node-1.cnf
openssl x509 -req -days 1825 -in vault-node-1.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out vault-node-1.crt -extensions v3_req -extfile vault-node-1.cnf

# Step 3 — vault-node-2 cert (IP: <Node-2-Private-IP>)
openssl genrsa -out vault-node-2.key 2048
cat > vault-node-2.cnf <<EOF
[req]
req_extensions     = v3_req
distinguished_name = req_distinguished_name
prompt             = no
[req_distinguished_name]
CN = vault-node-2
[v3_req]
keyUsage         = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName   = @alt_names
[alt_names]
DNS.1 = vault-node-2
DNS.2 = localhost
IP.1  = <Node-2-Private-IP>
IP.2  = 127.0.0.1
EOF
openssl req -new -key vault-node-2.key -out vault-node-2.csr -config vault-node-2.cnf
openssl x509 -req -days 1825 -in vault-node-2.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out vault-node-2.crt -extensions v3_req -extfile vault-node-2.cnf

# Step 4 — vault-node-3 cert (IP: <Node-3-Private-IP>)
openssl genrsa -out vault-node-3.key 2048
cat > vault-node-3.cnf <<EOF
[req]
req_extensions     = v3_req
distinguished_name = req_distinguished_name
prompt             = no
[req_distinguished_name]
CN = vault-node-3
[v3_req]
keyUsage         = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName   = @alt_names
[alt_names]
DNS.1 = vault-node-3
DNS.2 = localhost
IP.1  = <Node-3-Private-IP>
IP.2  = 127.0.0.1
EOF
openssl req -new -key vault-node-3.key -out vault-node-3.csr -config vault-node-3.cnf
openssl x509 -req -days 1825 -in vault-node-3.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out vault-node-3.crt -extensions v3_req -extfile vault-node-3.cnf

# Step 5 — Verify each cert has the correct IP
echo "=== vault-node-1 (should show <Node-1-Private-IP>) ==="
openssl x509 -in vault-node-1.crt -text -noout | grep -A1 "Subject Alternative"

echo "=== vault-node-2 (should show <Node-2-Private-IP>) ==="
openssl x509 -in vault-node-2.crt -text -noout | grep -A1 "Subject Alternative"

echo "=== vault-node-3 (should show <Node-3-Private-IP>) ==="
openssl x509 -in vault-node-3.crt -text -noout | grep -A1 "Subject Alternative"
```

---

### Phase 3 — Distribute Certificates to Servers

```bash
cd ~/vault-certs

# vault-node-1
scp -i ~/.ssh/pvt-key.pem ca.crt vault-node-1.crt vault-node-1.key \
  ubuntu@<Node-1-Public-IP>:/tmp/

# vault-node-2
scp -i ~/.ssh/pvt-key.pem ca.crt vault-node-2.crt vault-node-2.key \
  ubuntu@<Node-2-Public-IP>:/tmp/

# vault-node-3
scp -i ~/.ssh/pvt-key.pem ca.crt vault-node-3.crt vault-node-3.key \
  ubuntu@<Node-3-Public-IP>:/tmp/
```

On **vault-node-1**:
```bash
sudo mv /tmp/ca.crt /tmp/vault-node-1.crt /tmp/vault-node-1.key /etc/vault.d/tls/
sudo chown -R vault:vault /etc/vault.d/tls/
sudo chmod 640 /etc/vault.d/tls/*.key
sudo chmod 644 /etc/vault.d/tls/*.crt
```

On **vault-node-2**:
```bash
sudo mv /tmp/ca.crt /tmp/vault-node-2.crt /tmp/vault-node-2.key /etc/vault.d/tls/
sudo chown -R vault:vault /etc/vault.d/tls/
sudo chmod 640 /etc/vault.d/tls/*.key
sudo chmod 644 /etc/vault.d/tls/*.crt
```

On **vault-node-3**:
```bash
sudo mv /tmp/ca.crt /tmp/vault-node-3.crt /tmp/vault-node-3.key /etc/vault.d/tls/
sudo chown -R vault:vault /etc/vault.d/tls/
sudo chmod 640 /etc/vault.d/tls/*.key
sudo chmod 644 /etc/vault.d/tls/*.crt
```

---

### Phase 4 — Configure Vault on Each Node

**vault-node-1** (`/etc/vault.d/vault.hcl`):

```hcl
ui            = true
cluster_name  = "vault-ha-cluster"
disable_mlock = true

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-1"

  retry_join {
    leader_api_addr         = "https://<Node-2-Private-IP>:8200"
    leader_ca_cert_file     = "/etc/vault.d/tls/ca.crt"
    leader_client_cert_file = "/etc/vault.d/tls/vault-node-1.crt"
    leader_client_key_file  = "/etc/vault.d/tls/vault-node-1.key"
  }
  retry_join {
    leader_api_addr         = "https://<Node-3-Private-IP>:8200"
    leader_ca_cert_file     = "/etc/vault.d/tls/ca.crt"
    leader_client_cert_file = "/etc/vault.d/tls/vault-node-1.crt"
    leader_client_key_file  = "/etc/vault.d/tls/vault-node-1.key"
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_cert_file      = "/etc/vault.d/tls/vault-node-1.crt"
  tls_key_file       = "/etc/vault.d/tls/vault-node-1.key"
  tls_client_ca_file = "/etc/vault.d/tls/ca.crt"
}

api_addr     = "https://<Node-1-Private-IP>:8200"
cluster_addr = "https://<Node-1-Private-IP>:8201"
```

**vault-node-2** (`/etc/vault.d/vault.hcl`):

```hcl
ui            = true
cluster_name  = "vault-ha-cluster"
disable_mlock = true

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-2"

  retry_join {
    leader_api_addr         = "https://<Node-1-Private-IP>:8200"
    leader_ca_cert_file     = "/etc/vault.d/tls/ca.crt"
    leader_client_cert_file = "/etc/vault.d/tls/vault-node-2.crt"
    leader_client_key_file  = "/etc/vault.d/tls/vault-node-2.key"
  }
  retry_join {
    leader_api_addr         = "https://<Node-3-Private-IP>:8200"
    leader_ca_cert_file     = "/etc/vault.d/tls/ca.crt"
    leader_client_cert_file = "/etc/vault.d/tls/vault-node-2.crt"
    leader_client_key_file  = "/etc/vault.d/tls/vault-node-2.key"
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_cert_file      = "/etc/vault.d/tls/vault-node-2.crt"
  tls_key_file       = "/etc/vault.d/tls/vault-node-2.key"
  tls_client_ca_file = "/etc/vault.d/tls/ca.crt"
}

api_addr     = "https://<Node-2-Private-IP>:8200"
cluster_addr = "https://<Node-2-Private-IP>:8201"
```

**vault-node-3** (`/etc/vault.d/vault.hcl`):

```hcl
ui            = true
cluster_name  = "vault-ha-cluster"
disable_mlock = true

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-3"

  retry_join {
    leader_api_addr         = "https://<Node-1-Private-IP>:8200"
    leader_ca_cert_file     = "/etc/vault.d/tls/ca.crt"
    leader_client_cert_file = "/etc/vault.d/tls/vault-node-3.crt"
    leader_client_key_file  = "/etc/vault.d/tls/vault-node-3.key"
  }
  retry_join {
    leader_api_addr         = "https://<Node-2-Private-IP>:8200"
    leader_ca_cert_file     = "/etc/vault.d/tls/ca.crt"
    leader_client_cert_file = "/etc/vault.d/tls/vault-node-3.crt"
    leader_client_key_file  = "/etc/vault.d/tls/vault-node-3.key"
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_cert_file      = "/etc/vault.d/tls/vault-node-3.crt"
  tls_key_file       = "/etc/vault.d/tls/vault-node-3.key"
  tls_client_ca_file = "/etc/vault.d/tls/ca.crt"
}

api_addr     = "https://<Node-3-Private-IP>:8200"
cluster_addr = "https://<Node-3-Private-IP>:8201"
```

---

### Phase 5 — Start Vault on All 3 Nodes

```bash
sudo systemctl enable vault
sudo systemctl start vault
sudo systemctl status vault
```

Verify Vault is responding:

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
vault status
```

Expected output (before initialization):

```
Initialized    false
Sealed         true
Storage Type   raft
HA Enabled     true
```

---

### Phase 6 — Initialize the Cluster (vault-node-1 ONLY)

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"

vault operator init \
  -key-shares=5 \
  -key-threshold=3
```

This outputs 5 unseal keys and 1 root token. **Save them immediately and securely — you will never see them again.**

```
Unseal Key 1: <key-1>
Unseal Key 2: <key-2>
Unseal Key 3: <key-3>
Unseal Key 4: <key-4>
Unseal Key 5: <key-5>

Initial Root Token: hvs.xxxxxxxxxxxxxxxxxx
```

---

### Phase 7 — Unseal All 3 Nodes

Run on **each node** — provide any 3 of the 5 unseal keys:

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"

vault operator unseal <Unseal-Key-1>
vault operator unseal <Unseal-Key-2>
vault operator unseal <Unseal-Key-3>
```

After the 3rd key, you will see `Sealed: false`.

---

### Phase 8 — Verify the Cluster

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<root-token>"

vault operator raft list-peers
```

Expected output:

```
Node          Address                State     Voter
vault-node-1  <Node-1-Private-IP>:8201    leader    true
vault-node-2  <Node-2-Private-IP>:8201    follower  true
vault-node-3  <Node-3-Private-IP>:8201     follower  true
```

---

## Environment Variables

Add to `~/.bashrc` on each node:

```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
export VAULT_TOKEN="<root-token>"
```

---

## Testing HA Failover

```bash
# Stop the active leader (on vault-node-1)
sudo systemctl stop vault

# On vault-node-2 — a new leader should elect within seconds
vault operator raft list-peers
```

One of the remaining nodes becomes the new leader automatically.

---

## File Reference

| File | Location | Purpose |
|------|----------|---------|
| `ca.key` | Local machine only | Signs all node certs — never copy to servers |
| `ca.crt` | All 3 nodes | Lets each node verify certs signed by your CA |
| `vault-nodeX.crt` | Node X only | Node's TLS identity certificate |
| `vault-nodeX.key` | Node X only | Private key — 640 permissions, never shared |
| `vault.hcl` | `/etc/vault.d/vault.hcl` | Vault configuration |
| Raft data | `/opt/vault/data/` | Vault encrypted storage |

---

## Common Commands

```bash
# Check cluster status
vault status

# List Raft peers
vault operator raft list-peers

# Unseal a node
vault operator unseal <key>

# Check Vault logs
sudo journalctl -u vault -f

# Restart Vault
sudo systemctl restart vault

# Verify TLS cert on a node
openssl x509 -in /etc/vault.d/tls/vault-nodeX.crt -text -noout | grep -A1 "Subject Alternative"

# Test TLS from local machine
curl --cacert ~/vault-certs/ca.crt https://<node-ip>:8200/v1/sys/health
```

---

## Important Notes

- **Unseal keys:** Keep all 5 keys in separate secure locations. You need 3 to unseal.
- **Root token:** Use only for initial setup. Create admin policies and tokens for regular use.
- **Auto-unseal:** For production, configure AWS KMS auto-unseal so nodes unseal automatically after restart.
- **Certificates:** Current certs are valid for 5 years (1825 days). Set a reminder to rotate them.
- **Cost saving:** Stop all 3 instances when not practicing — stopped instances only incur EBS storage cost (~$0.06/day total).
- **TLS disable:** Never use `tls_disable = true` in production or even practice beyond initial testing.

---

## Final Thought

> *"Security is not a product, but a process. Every certificate you sign, every port you restrict, and every unseal key you protect is a deliberate act of discipline — and discipline, repeated daily, becomes expertise."*

> *"A cluster that survives the failure of one node is not lucky — it is designed. Design with intention, test with purpose, and never stop learning."*

> *"The best engineers are not those who never make mistakes — they are those who build systems resilient enough to survive them."*

---

*Built with patience, curiosity, and a lot of `vault operator unseal` commands.* 🔐