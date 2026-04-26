# Vault OIDC Authentication Setup with Auth0

This guide walks through setting up HashiCorp Vault OIDC authentication using Auth0 as the identity provider.

---

## Prerequisites

- HashiCorp Vault installed and running
- Auth0 account (free at [auth0.com](https://auth0.com))
- Vault CLI installed on your local machine
- EC2 or server with Vault running

---

## Architecture Overview

```
Local Machine          Auth0 (OIDC Provider)        Vault (EC2/Server)
────────────           ─────────────────────        ──────────────────
vault login     ──►    Login page appears
Enter creds     ──►    Auth0 verifies user
                       Returns OIDC token    ──►    Vault validates token
                                                    Issues Vault token ✅
```

---

## Part 1 — Auth0 Setup

### Step 1 — Create an Auth0 Account

1. Go to [https://auth0.com](https://auth0.com) and sign up
2. After login, note your **tenant domain**:
   ```
   dev-XXXX.us.auth0.com
   ```

### Step 2 — Create an Application

1. In Auth0 dashboard → **Applications → Applications**
2. Click **"Create Application"**
3. Name: `vault-oidc`
4. Type: **Regular Web Application**
5. Click **Create**

### Step 3 — Configure Application URIs

In the app **Settings** tab, scroll to **Application URIs** and fill in:

| Field | Value |
|-------|-------|
| Allowed Callback URLs | `http://localhost:8250/oidc/callback` |
| Allowed Logout URLs | `http://localhost:8250` |
| Allowed Web Origins | `http://localhost:8250` |

Click **Save Changes**.

### Step 4 — Note Down Credentials

From the **Settings** tab, copy:

```
Domain        : dev-XXXX.us.auth0.com
Client ID     : <your-client-id>
Client Secret : <your-client-secret>  (click eye icon to reveal)
```

### Step 5 — Create a Test User

1. Left sidebar → **User Management → Users**
2. Click **"Create User"**
3. Fill in email and password
4. Connection: `Username-Password-Authentication`
5. Click **Create**

---

## Part 2 — Vault Configuration

### Step 1 — Enable OIDC Auth Method

```bash
vault auth enable oidc
```

### Step 2 — Configure OIDC with Auth0

```bash
vault write auth/oidc/config \
  oidc_discovery_url="https://dev-XXXX.us.auth0.com/" \
  oidc_client_id="<your-client-id>" \
  oidc_client_secret="<your-client-secret>" \
  default_role="default"
```

> Replace `dev-XXXX`, `<your-client-id>`, and `<your-client-secret>` with your actual values.

### Step 3 — Create a Vault Role

```bash
vault write auth/oidc/role/default \
  bound_audiences="<your-client-id>" \
  allowed_redirect_uris="http://localhost:8250/oidc/callback" \
  user_claim="sub" \
  policies="default"
```

### Step 4 — Verify Configuration

```bash
vault auth list
vault read auth/oidc/config
```

---

## Part 3 — Testing the Login

### Option A — Local Machine (Recommended)

If your Vault uses a self-signed certificate, set this first:

```bash
export VAULT_ADDR="https://<your-vault-ip>:8200"
export VAULT_SKIP_VERIFY=true
```

Then login:

```bash
vault login -method=oidc role="default"
```

1. Copy the URL printed in the terminal
2. Paste it in your browser
3. Log in with your Auth0 test user credentials
4. Vault issues a token on success ✅

### Option B — SSH Tunnel (If Vault is on EC2)

**Terminal 1 — Create SSH tunnel:**
```bash
ssh -L 8200:localhost:8200 -L 8250:localhost:8250 ubuntu@<ec2-public-ip>
```

**Terminal 2 — Login:**
```bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_SKIP_VERIFY=true
vault login -method=oidc role="default"
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `permission denied` on audit log | Vault user lacks write access | `sudo chown vault:vault /var/log/vault_audit.log` |
| `Client sent HTTP to HTTPS server` | Wrong protocol in VAULT_ADDR | Use `https://` not `http://` |
| `x509: certificate is valid for 127.0.0.1` | TLS cert mismatch | Set `export VAULT_SKIP_VERIFY=true` |
| `Timed out waiting for response` | No browser on server | Use SSH tunnel or local machine |
| `xdg-open not found` | No GUI on EC2 | Copy URL manually and open in local browser |

---

## Security Notes

> ⚠️ `VAULT_SKIP_VERIFY=true` disables TLS verification. Use only for **testing/development**.  
> In production, use a valid TLS certificate that includes your server's public IP or domain.

- Never commit `Client Secret` to version control
- Use proper Vault policies to restrict user access to specific secret paths
- Rotate Auth0 API tokens regularly

---

## References

- [Vault OIDC Auth Method Docs](https://developer.hashicorp.com/vault/docs/auth/jwt)
- [Auth0 Documentation](https://auth0.com/docs)
- [HashiCorp Learn — OIDC Auth](https://developer.hashicorp.com/vault/tutorials/auth-methods/oidc-auth)