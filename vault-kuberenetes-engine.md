# HashiCorp Vault Integration with Kubernetes & Vault Secrets Operator

This project documents the setup and configuration required to integrate HashiCorp Vault with Kubernetes. It specifically covers the workflow for syncing a Docker Registry credential (`.dockerconfigjson`) from Vault to a Kubernetes Secret using the **HashiCorp Vault Secrets Operator (VSO)**.

## 📋 Prerequisites

* **Kubernetes Cluster** with HashiCorp Vault installed (Namespace: `vault-system`).
* **CLI Tools:** `kubectl`, `vault`, `helm`.
* **Access:** Admin access to the Vault pod and local access to `~/.docker/config.json`.

---

## ⚙️ Part 1: Configure Vault (Server-Side)

These steps configure the Vault server to accept authentication requests from Kubernetes Service Accounts.

### 1. Access the Vault Pod
Exec into the running Vault pod:
```bash
kubectl exec -it vault-0 -n vault -- sh
```
```bash
vault login
```
- login with `root credentials`

### 2. Enable Kubernetes Authentication
Enable the auth engine that allows K8s ServiceAccounts to log in:
```bash
vault auth enable kubernetes
```
### 3. Configure Kubernetes Connection
Vault uses the internal environment variables of the pod to communicate with the K8s API:
```bash
vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```
### 4. Create Access Policy
Create a read-only policy `(ro-docker-policy)` granting access to the specific secret path:
```bash
vault policy write ro-docker-policy - <<EOF
path "kv/data/your/docker-registry" {
  capabilities = ["read"]
}
EOF
```

### 5. Create the Role (Binding)
Bind the policy to a Kubernetes Service Account.

bound_service_account_names: The SA used by the app (default).

bound_service_account_namespaces: The namespaces allowed to login.
```bash
vault write auth/kubernetes/role/your-role \
    bound_service_account_names=default \
    bound_service_account_namespaces="*" \
    policies=ro-docker-policy \
    ttl=30m
```
Note: Type `exit` to leave the pod and return to your local terminal.

## 🔐 Part 2: Store the Secret (Client-Side)
These steps upload your local Docker credentials to Vault.

### 1. Port Forward Vault
Open a terminal to expose Vault locally:
```bash
kubectl port-forward vault-0 8200:8200 -n vault
```
### 2. Login Locally
In a separate terminal, export the address and log in `(requires root token or admin credentials)`:
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
vault login
```
### 3. Upload config.json
Read your local docker config and write it to the Vault KV path.
```bash
# Read local docker config
JSON_CONTENT=$(sudo cat /home/sahu/.docker/config.json)

# Upload to Vault
# Path: kv/your/docker-registry
# Key: .dockerconfigjson
vault kv put kv/your/docker-registry .dockerconfigjson="$JSON_CONTENT"
```

# 🛠️ Part 3: Vault Secrets Operator Setup
Install the operator that manages the synchronization between Vault and Kubernetes Secrets.
```bash
# 1. Add HashiCorp Helm Repo
helm repo add hashicorp [https://helm.releases.hashicorp.com](https://helm.releases.hashicorp.com)
helm repo update

# 2. Install the Operator
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
    -n vault-secrets-operator-system \
    --create-namespace
```
# 🚀 Part 4: Kubernetes Resources (Helm Chart)
Add the following files to your Helm Chart (templates/ directory) to configure the connection.

### 1. `templates/vault_connection.yaml`
Defines the connection endpoint for the Vault Service.
```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: vault-connection
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  address: "http://vault-active.vault.svc.cluster.local:8200"
```
### 2. `templates/vault_auth.yaml`
Configures authentication using the `your-role` created in Part 1.

```YAML
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: your-role
    serviceAccount: default
  vaultConnectionRef: vault-connection
```
### 3. `templates/vault_secret.yaml`
Instructs the operator to fetch the secret and create a Kubernetes Secret named `regcred2`.
```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: {{ .Values.dockerRegistrySecret.name }}-source
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  type: kv-v2
  mount: kv
  path: your/docker-registry
  
  refreshAfter: 30m
  vaultAuthRef: vault-auth
  destination:
    name: {{ .Values.dockerRegistrySecret.name }}
    create: true
    type: kubernetes.io/dockerconfigjson
```
### 4. values.yaml configuration
Ensure your values file contains the target secret name.
```yaml
dockerRegistrySecret:
  name: "regcred"
```
# 🔍 Troubleshooting
Vault Address: Ensure `spec.address` in `VaultConnection` matches the internal DNS of your Vault service.

Namespace Mismatch: If the `VaultAuth` fails, check that the namespace where the application is running matches the `bound_service_account_namespaces` in the Vault Role.

Secret Path: Ensure the `path` in `VaultStaticSecret` matches exactly where you ran the `vault kv put` command.