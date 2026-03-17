# 00 — Vault HA Cluster Architecture

> *"A cluster that survives the failure of one node is not lucky — it is designed. Design with intention, test with purpose, and never stop learning."*

---

## Overview

This document describes the architecture of a production-grade, 3-node HashiCorp Vault HA cluster deployed on AWS EC2 using Raft integrated storage and mutual TLS (mTLS) for all inter-node communication.

---

## Architecture Diagram

```
                         [ Team / Applications ]
                                   |
                            [ Nginx Proxy ]
                           vault-node-1:80
                                   |
              ┌────────────────────┼────────────────────┐
              |                    |                    |
       [ vault-node-1 ]    [ vault-node-2 ]    [ vault-node-3 ]
       Active / Leader        Standby              Standby
       <Node-1-Private-IP>   <Node-2-Private-IP>  <Node-3-Private-IP>
       AZ: us-east-1a        AZ: us-east-1b       AZ: us-east-1c
       HTTPS  :8200           HTTPS  :8200          HTTPS  :8200
       mTLS   :8201           mTLS   :8201          mTLS   :8201
              |                    |                    |
              └──────── Raft Replication (mTLS) ────────┘
```

---

## Why 3 Nodes?

Vault HA uses the **Raft consensus algorithm** to elect a leader and replicate data. Raft requires a quorum (majority) to make decisions. With 3 nodes, the cluster tolerates 1 node failure while maintaining full operation.

| Nodes | Quorum | Tolerates | HA? |
|-------|--------|-----------|-----|
| 1     | 1      | 0 failures | No  |
| 2     | 2      | 0 failures | No  |
| **3** | **2**  | **1 failure** | **Yes** |
| 5     | 3      | 2 failures | Yes |

**3 nodes is the minimum for true HA.** With 2 nodes, if 1 goes down you lose quorum and the entire cluster freezes — no reads, no writes.

---

## Raft Consensus — How it Works

Raft is a distributed consensus algorithm that ensures all nodes agree on the same data, even during failures.

### Leader election

Every cluster has exactly one leader at a time. The leader handles all write requests and replicates data to followers. If followers stop receiving heartbeats from the leader, they trigger an election:

```
Normal operation:
  Node 1 (Leader) ──heartbeat──► Node 2 (Follower)
                   ──heartbeat──► Node 3 (Follower)

Leader fails:
  Node 2: "I haven't heard from the leader — I'll run for election"
  Node 3: "I vote for Node 2"
  Node 2 wins majority → becomes new leader
  Time to elect: ~2-5 seconds
```

### Data replication

All writes go to the leader. The leader replicates to followers before acknowledging success:

```
Client ──write──► Leader
                  Leader ──replicate──► Follower 1 ✓
                  Leader ──replicate──► Follower 2 ✓
                  Majority confirmed → write committed
                  Leader ──acknowledge──► Client
```

---

## Two Communication Channels

Vault has two distinct network channels, both secured with TLS:

| Channel | Port | Protocol | Carries |
|---------|------|----------|---------|
| API / UI | 8200 | HTTPS (TLS) | Client requests, tokens, secrets, auth |
| Cluster / Raft | 8201 | mTLS | Raft log replication, leader election |

### What is mTLS?

Regular TLS: only the **server** proves its identity to the client.

Mutual TLS (mTLS): **both sides** prove their identity to each other. On port 8201, every Vault node presents its certificate to every other node. A rogue server cannot join the cluster — it has no certificate signed by your CA.

---

## Certificate Architecture

```
Your CA (ca.crt + ca.key)       ← Created once, never leaves your workstation
├── vault-node-1.crt            ← Signed by CA, SAN includes Node 1 private IP
├── vault-node-2.crt            ← Signed by CA, SAN includes Node 2 private IP
└── vault-node-3.crt            ← Signed by CA, SAN includes Node 3 private IP
```

Each node gets its own certificate containing its private IP as a Subject Alternative Name (SAN). If the IP in the cert does not match the server's actual IP, TLS handshakes fail with `x509: certificate is not valid for this IP`.

Every node also receives `ca.crt` so it can verify the certificates of the other nodes.

---

## AWS Infrastructure

### Node details

| Node | Public IP | Private IP | Availability Zone |
|------|-----------|------------|-------------------|
| vault-node-1 | `<Node-1-Public-IP>` | `<Node-1-Private-IP>` | us-east-1a |
| vault-node-2 | `<Node-2-Public-IP>` | `<Node-2-Private-IP>` | us-east-1b |
| vault-node-3 | `<Node-3-Public-IP>` | `<Node-3-Private-IP>` | us-east-1c |

Each node is in a **different Availability Zone** — so a single AZ outage cannot take down the entire cluster.

### Instance specifications

| Spec | Value | Reason |
|------|-------|--------|
| Instance type | m7i-flex.large | 8 GB RAM — Vault needs headroom for Raft index |
| vCPU | 2 | Sufficient for Vault workloads |
| RAM | 8 GB | Comfortable — OS takes ~500 MB, Vault needs the rest |
| Storage | 20 GB gp3 | OS ~5 GB + Vault data + logs |
| OS | Ubuntu 24.04 LTS | Stable, well-supported |

> Why not t3.small (2 GB RAM)? Vault keeps its entire Raft storage index in memory. On 2 GB, after the OS takes ~500 MB, you risk OOM kills under load. Always use at least 4 GB RAM for Vault, ideally 8 GB.

### Security group rules

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 22 | TCP | Your IP/32 | SSH access |
| 80 | TCP | Team IPs | Nginx reverse proxy (Vault UI) |
| 8200 | TCP | `<VPC-CIDR>` | Vault API (VPC internal only) |
| 8201 | TCP | `<VPC-CIDR>` | Raft replication (VPC internal only) |

> Ports 8200 and 8201 are restricted to VPC CIDR — they are never exposed to the public internet.

---

## Storage Architecture

Vault uses **Raft integrated storage** — no external database or Consul required. Each node stores its own copy of the Raft log on disk.

```
/opt/vault/data/          ← Raft storage directory on each node
├── raft/
│   ├── raft.db           ← BoltDB database (Raft log)
│   └── snapshots/        ← Local Raft snapshots
└── vault.db              ← Vault encrypted data
```

All data in `/opt/vault/data/` is **encrypted at rest** by Vault's encryption layer before being written to disk. Even if someone steals the disk, they cannot read the secrets without the unseal keys.

---

## Seal / Unseal Mechanism

Vault uses **Shamir's Secret Sharing** to protect the master encryption key.

### How it works

```
Master key
    │
    ▼ Split into 5 shares (key-shares=5)
┌───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │   ← 5 unseal keys generated at init
└───┴───┴───┴───┴───┘
    Any 3 of 5 needed to reconstruct master key (key-threshold=3)
```

### Sealed vs unsealed

| State | Meaning | Can serve requests? |
|-------|---------|---------------------|
| Sealed | Master key not in memory | No |
| Unsealed | Master key loaded | Yes |

After every restart, Vault starts **sealed**. You must provide 3 of the 5 unseal keys on each node to unseal it. This is why auto-unseal with AWS KMS is recommended for production — so nodes unseal automatically after a restart.

---

## Nginx Reverse Proxy

Nginx runs on vault-node-1 and exposes the Vault UI to the team on port 80. It proxies requests to Vault on `127.0.0.1:8200`.

```
Team browser ──HTTP:80──► Nginx (vault-node-1) ──HTTPS:8200──► Vault
```

This setup means:
- Team members only need the Node 1 public IP
- Port 8200 is never directly exposed to the internet
- All traffic to Vault goes through Nginx

> For production: replace Nginx with an AWS ALB to distribute traffic across all 3 nodes and eliminate the single point of failure on Node 1.

---

## File Layout on Each Server

```
/etc/vault.d/
├── vault.hcl               ← Vault configuration file
└── tls/
    ├── ca.crt              ← CA certificate (verify other nodes)
    ├── vault-nodeX.crt     ← This node's TLS certificate
    └── vault-nodeX.key     ← This node's private key (chmod 640)

/opt/vault/
├── data/                   ← Raft storage (owned by vault user)
└── snapshots/              ← Manual and automated backups

/var/log/vault/
└── audit.log               ← Vault audit log (all requests recorded)
```

---

## Setup Order

The correct order for configuring a fresh Vault cluster:

| Step | Action | File |
|------|--------|------|
| 1 | Enable audit log | [02-audit-log.md](./02-audit-log.md) |
| 2 | Create policies | [03-policies.md](./03-policies.md) |
| 3 | Enable auth methods | [04-auth-methods.md](./04-auth-methods.md) |
| 4 | Enable secrets engines | [05-kv-secrets-engine.md](./05-kv-secrets-engine.md) |
| 5 | Create users | [04-auth-methods.md](./04-auth-methods.md) |
| 6 | Take Raft snapshot | [06-raft-snapshot.md](./06-raft-snapshot.md) |

---

## Startup Checklist (after every restart)

Every time EC2 instances are restarted:

```bash
# 1. Check Vault service started
sudo systemctl status vault

# 2. Unseal all 3 nodes (Vault always starts sealed)
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
vault operator unseal 
vault operator unseal 
vault operator unseal 

# 3. Verify cluster health on Node 1
export VAULT_TOKEN=""
vault operator raft list-peers

# 4. Restart Nginx on Node 1 if needed
sudo systemctl restart nginx
```

---

## Next Step

Proceed to **[01-cluster-setup.md](./01-cluster-setup.md)** for the full step-by-step cluster installation guide.

---

*Part of the [Vault HA Production Setup Guide](./Vault-HA-Production-Setup-Guide.md)*