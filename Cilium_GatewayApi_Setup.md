# Gateway API LoadBalancer Setup with Cilium, cert-manager, and Hetzner Cloud
Complete guide to set up a production-ready Gateway API LoadBalancer on Kubernetes (RKE2) with automatic TLS certificate management using cert-manager, Cilium Gateway API, and Hetzner Cloud infrastructure.

## 📋 Table of Contents
- Overview
- Architecture
- Prerequisites
- Component Overview
- Installation Steps

    - Step 1: Enable Cilium Gateway API
    - Step 2: Install Hetzner DNS Webhook
    - Step 3: Configure DNS API Token
    - Step 4: Create RBAC Permissions
    - Step 5: Create ClusterIssuer
    - Step 6: Setup Gateway Namespace
    - Step 7: Request Wildcard Certificate
    - Step 8: Upgrade Gateway API CRDs
    - Step 9: Deploy Gateway
    - Step 10: Deploy Sample Application
    - Step 11: Configure DNS Records
- Verification
- Configuration Reference
- Troubleshooting
- Additional Resources
## 🎯 Overview
This guide walks you through setting up a complete Gateway API LoadBalancer solution that provides:
- Automatic HTTPS/TLS: Certificates automatically issued and renewed by Let's Encrypt
- Wildcard Domain Support: Single certificate for all subdomains (e.g., `*.k8s.bluespice.store`)
- HTTP to HTTPS Redirect: Automatic redirect from HTTP to HTTPS
- Cloud Native: Uses Kubernetes Gateway API standard
- Production Ready: Integrated with Hetzner Cloud LoadBalancer
- Easy Application Deployment: Simple HTTPRoute resources to expose services
## 🏗️ Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                            │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS Request
                           ▼
┌─────────────────────────────────────────────────────────────┐
│        Hetzner Cloud Load Balancer (91.98.1.173)            │
│                     Ports: 80, 443                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Cilium Gateway API Controller                  │
│                                                             │
│  ┌──────────────────────────────────────────────────┐       │
│  │  Gateway: cilium-gateway-lb                      │       │
│  │  - HTTP Listener (Port 80)  → Redirect to HTTPS  │       │
│  │  - HTTPS Listener (Port 443) → TLS Termination   │       │
│  │  - Certificate: *.k8s.example.com                │       │
│  └──────────────────────────────────────────────────┘       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      HTTPRoute                              │
│             Routes traffic to backend services              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Kubernetes Service (ClusterIP)             │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Application Pods                         │
└─────────────────────────────────────────────────────────────┘
```
## Certificate Management Flow
```
┌─────────────────────────────────────────────────────────────┐
│                    cert-manager                             │
│                                                             │
│  ┌────────────────┐      ┌─────────────────────────┐        │
│  │ ClusterIssuer  │─────▶│  Certificate Resource   │        │
│  │ (Let's Encrypt)│      │  (*.k8s.example.com)    │        │
│  └────────────────┘      └───────────┬─────────────┘        │
│                                       │                     │
│                                       ▼                     │
│                          ┌──────────────────────┐           │
│                          │  DNS-01 Challenge    │           │
│                          │  via Hetzner Webhook │           │
│                          └──────────┬───────────┘           │
│                                     │                       │
│                                     ▼                       │
│                          ┌──────────────────────┐           │
│                          │ Hetzner DNS API      │           │
│                          │ Creates TXT Record   │           │
│                          └──────────┬───────────┘           │
│                                     │                       │
│                                     ▼                       │
│                          ┌──────────────────────┐           │
│                          │ Let's Encrypt        │           │
│                          │ Validates & Issues   │           │
│                          └──────────┬───────────┘           │
│                                     │                       │
│                                     ▼                       │
│                          ┌──────────────────────┐           │
│                          │ TLS Secret Created   │           │
│                          │ in gateway-system    │           │
│                          └──────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```
## 📦 Prerequisites
Before starting, ensure you have:

- Kubernetes Cluster: RKE2 cluster with Cilium CNI installed
- Cilium Version: 1.16.0 or higher
- cert-manager: v1.16.3 or higher installed
- Hetzner Cloud Account: With DNS service enabled
- Domain: Managed by Hetzner DNS (e.g., example.com)
- Hetzner DNS API Token: With read/write permissions
- kubectl: Configured to access your cluster
- helm: v3.x installed

Required Permissions

- Cluster admin access to create CRDs, namespaces, and cluster-wide resources
- Access to Hetzner Cloud Console to create DNS API tokens
- DNS management access for your domain
## 🧩 Component Overview
### Cilium Gateway API

- Purpose: Native Kubernetes Gateway API implementation using Cilium's Envoy-based proxy
- Features: Layer 7 traffic management, TLS termination, HTTP routing
- Documentation: Cilium Gateway API

### cert-manager

- Purpose: Kubernetes add-on to automate certificate management
- Features: Automatic certificate issuance and renewal from Let's Encrypt
- Documentation: cert-manager

### Hetzner DNS Webhook

- Purpose: cert-manager webhook for DNS-01 challenge validation via Hetzner DNS
- Why DNS-01: Required for wildcard certificates, works without exposing services
- Documentation: cert-manager-webhook-hetzner

### Gateway API

- Purpose: Standard Kubernetes API for traffic routing (successor to Ingress)
- Features: More expressive, extensible, and role-oriented than Ingress
- Documentation: Gateway API
## 🚀 Installation Steps
#### Step 1: Enable Cilium Gateway API
Cilium's Gateway API support must be enabled before proceeding.
#### Edit Cilium Configuration
For RKE2 clusters, edit the Cilium configuration manifest:
```bash
# Edit the RKE2 Cilium configuration
sudo vim /var/lib/rancher/rke2/server/manifests/rke2-cilium-config.yaml
```
`File: /var/lib/rancher/rke2/server/manifests/rke2-cilium-config.yaml`
```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-cilium
  namespace: kube-system
spec:
  valuesContent: |-
    kubeProxyReplacement: true
    k8sServiceHost: "localhost"
    k8sServicePort: "6443"
    cni:
      chainingMode: "none"
    gatewayAPI:
      enabled: true          # Enable Gateway API
    envoy:
      enabled: true           # Enable Envoy proxy
    l7Proxy: true            # Enable Layer 7 proxy
```
`Restart RKE2 Server`
```bash
# Restart RKE2 server to apply changes
sudo systemctl restart rke2-server.service

# Check service status
sudo systemctl status rke2-server.service

# Follow logs (optional)
sudo journalctl -u rke2-server -f
```
`Verify Cilium Configuration`
```bash
# Wait for Cilium pods to be ready
kubectl get pods -n kube-system | grep cilium

# Verify Gateway API is enabled
kubectl get cm -n kube-system cilium-config -o yaml | grep gateway

# Expected output should show:
#   enable-gateway-api: "true"
#   gateway-api-default-gatewayclass: "cilium"
```
`Alternative: Manual ConfigMap Edit`
If you need to enable Gateway API without restarting RKE2:
```bash
# Edit Cilium ConfigMap
kubectl -n kube-system edit configmap cilium-config
```
Add these lines under `data:`:
```yaml
data:
  enable-gateway-api: "true"
  enable-envoy-config: "true"
  gateway-api-default-gatewayclass: "cilium"
```
Then restart Cilium:
```bash
kubectl rollout restart daemonset cilium -n kube-system
```
### Step 2: Install Hetzner DNS Webhook
The Hetzner DNS webhook enables cert-manager to complete DNS-01 challenges using Hetzner's DNS service.

`Add Helm Repository`
```bash
helm repo add cert-manager-webhook-hetzner https://vadimkim.github.io/cert-manager-webhook-hetzner
helm repo update
```
`Install the Webhook`
```bash
helm install cert-manager-webhook-hetzner \
  cert-manager-webhook-hetzner/cert-manager-webhook-hetzner \
  --namespace cert-manager \
  --set groupName=acme.hetzner.cloud
```
`Verify Installation`

```bash
# Check webhook pods
kubectl get pods -n cert-manager | grep hetzner

# Expected output:
# cert-manager-webhook-hetzner-xxxxx   1/1     Running   0          30s

# Check webhook logs
kubectl logs -n cert-manager -l app.kubernetes.io/name=cert-manager-webhook-hetzner
```
### Step 3: Configure DNS API Token
Create a secret containing your Hetzner DNS API token.
`Obtain Hetzner DNS API Token`

- Log in to Hetzner DNS Console
- Navigate to API Tokens
- Click Create access token
- Give it a name (e.g., k8s-cert-manager)
- Ensure it has DNS Zone read and write permissions
- Copy the token (you'll only see it once!)

`Create Secret`

File: `hetzner-dns-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hetzner-dns-token
  namespace: cert-manager
type: Opaque
stringData:
  api-key: "YOUR_HETZNER_DNS_API_TOKEN_HERE"  # Replace with your actual token
```
Apply the Secret:
```bash
kubectl apply -f hetzner-dns-secret.yaml
```
Verify Secret Created:
```bash
kubectl get secret hetzner-dns-token -n cert-manager

# Verify secret content (optional - be careful with sensitive data)
kubectl get secret hetzner-dns-token -n cert-manager -o jsonpath='{.data.api-key}' | base64 -d
```
### Step 4: Create RBAC Permissions
The webhook needs permission to read the DNS API token secret.
File: `webhook-rbac.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cert-manager-webhook-hetzner
  namespace: cert-manager
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cert-manager-webhook-hetzner:secret-reader
  namespace: cert-manager
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["hetzner-dns-token"]
    verbs: ["get", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cert-manager-webhook-hetzner:secret-reader
  namespace: cert-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cert-manager-webhook-hetzner:secret-reader
subjects:
  - kind: ServiceAccount
    name: cert-manager-webhook-hetzner
    namespace: cert-manager
```
Apply RBAC:
```bash
kubectl apply -f webhook-rbac.yaml
```
Restart Webhook to Pick Up Permissions:
```bash
kubectl rollout restart deployment cert-manager-webhook-hetzner -n cert-manager

# Wait for rollout to complete
kubectl rollout status deployment cert-manager-webhook-hetzner -n cert-manager
```
### Step 5: Create ClusterIssuer
The ClusterIssuer tells cert-manager how to request certificates from Let's Encrypt.
File: `clusterissuer.yaml`
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-gateway-dns
spec:
  acme:
    email: your-email@example.com  # ⚠️ CHANGE THIS to your email
    server: https://acme-v02.api.letsencrypt.org/directory  # Production Let's Encrypt
    privateKeySecretRef:
      name: letsencrypt-gateway-dns
    solvers:
    - dns01:
        webhook:
          groupName: acme.hetzner.cloud
          solverName: hetzner
          config:
            secretName: hetzner-dns-token
            zoneName: example.com  # ⚠️ CHANGE THIS to your DNS zone
            apiUrl: https://dns.hetzner.com/api/v1
```
`Important Configuration Notes:`

- email: Replace with your email address (used for Let's Encrypt notifications)
- zoneName: Your DNS zone in Hetzner (e.g., example.com, not *.example.com)
- server: Production Let's Encrypt URL (rate limited). For testing, use staging: https://acme-staging-v02.api.letsencrypt.org/directory

Apply ClusterIssuer:

```bash
kubectl apply -f clusterissuer.yaml
```
Verify ClusterIssuer is Ready:
```bash
kubectl get clusterissuer letsencrypt-gateway-dns

# Expected output:
# NAME                      READY   AGE
# letsencrypt-gateway-dns   True    10s

# Check detailed status
kubectl describe clusterissuer letsencrypt-gateway-dns
```
### Step 6: Setup Gateway Namespace
Create a dedicated namespace for Gateway resources.
```bash
kubectl create namespace gateway-system
```
### Step 7: Request Wildcard Certificate
Request a wildcard TLS certificate for your domain.
File: `wildcard-certificate.yaml`
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-k8s-example
  namespace: gateway-system
spec:
  secretName: wildcard-k8s-example-tls
  issuerRef:
    name: letsencrypt-gateway-dns
    kind: ClusterIssuer
    group: cert-manager.io
  dnsNames:
    - "*.k8s.example.com"  # ⚠️ CHANGE THIS to your wildcard domain
```
`Configuration Notes:`

- dnsNames: Wildcard domain (e.g., *.k8s.example.com) covers all subdomains like app.k8s.example.com, api.k8s.example.com, etc.
- secretName: Name of the Kubernetes secret where the certificate will be stored
- namespace: Must match the Gateway namespace (gateway-system)

Apply Certificate:
```bash
kubectl apply -f wildcard-certificate.yaml
```
Watch Certificate Creation (This may take 2-5 minutes):
```bash
# Watch certificate status
kubectl get certificate -n gateway-system -w

# In another terminal, watch cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager -f
```
`Verify Certificate is Ready:`
```bash
# Check certificate status
kubectl get certificate wildcard-k8s-example -n gateway-system

# Expected output:
# NAME                    READY   SECRET                      AGE
# wildcard-k8s-example    True    wildcard-k8s-example-tls    2m

# Verify secret was created
kubectl get secret wildcard-k8s-example-tls -n gateway-system

# View certificate details
kubectl describe certificate wildcard-k8s-example -n gateway-system
```
Certificate Lifecycle:
The certificate will automatically renew before expiration (Let's Encrypt certificates are valid for 90 days and auto-renew at 60 days).

### Step 8: Upgrade Gateway API CRDs
Ensure you have the latest Gateway API CRDs installed.
```bash
# Install/upgrade Gateway API CRDs to v1.2.1
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# Wait for CRDs to be established
kubectl wait --for condition=established --timeout=60s crd/gateways.gateway.networking.k8s.io

# Verify CRDs are installed
kubectl get crd | grep gateway

# Expected output should include:
# gatewayclasses.gateway.networking.k8s.io
# gateways.gateway.networking.k8s.io
# httproutes.gateway.networking.k8s.io
# grpcroutes.gateway.networking.k8s.io
# referencegrants.gateway.networking.k8s.io
```
Restart Cilium Operator:
The Cilium operator needs to be restarted to recognize the new Gateway API CRDs.
```bash
kubectl rollout restart deployment cilium-operator -n kube-system

# Wait for rollout to complete
kubectl rollout status deployment cilium-operator -n kube-system
```
Verify Cilium Recognized Gateway API:
```bash
# Check Cilium operator logs
kubectl logs -n kube-system -l name=cilium-operator --tail=50 | grep -i gateway

# Should see logs like:
# "Checking for required GatewayAPI resources"
# "Successfully reconciled Gateway"
```
### Step 9: Deploy Gateway
Create the Gateway resource that will expose your applications.
File: `gateway.yaml`
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gateway-lb
  namespace: gateway-system
  annotations:
    io.cilium.gateway/loadbalancer-mode: shared
spec:
  gatewayClassName: cilium
  infrastructure:
    annotations:
      load-balancer.hetzner.cloud/name: cilium-gateway-lb
      load-balancer.hetzner.cloud/location: nbg1  # ⚠️ CHANGE to your Hetzner location
      load-balancer.hetzner.cloud/use-private-ip: "true"
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: wildcard-k8s-example-tls  # ⚠️ CHANGE to match your certificate secret
    allowedRoutes:
      namespaces:
        from: All
```
`Configuration Notes:`

- location: Hetzner datacenter location (nbg1, fsn1, hel1, etc.)
- certificateRefs.name: Must match the secret name from your Certificate resource
- allowedRoutes.from: All: Allows HTTPRoutes from any namespace to attach to this Gateway

`Hetzner Location Codes:`

- nbg1 - Nuremberg, Germany
- fsn1 - Falkenstein, Germany
- hel1 - Helsinki, Finland

Apply Gateway:
```bash
kubectl apply -f gateway.yaml

# Wait for reconciliation
sleep 10
```

Verify Gateway Status:
```bash
# Check Gateway is programmed
kubectl get gateway -n gateway-system

# Expected output:
# NAME                CLASS    ADDRESS       PROGRAMMED   AGE
# cilium-gateway-lb   cilium   91.98.1.173   True         30s

# Check detailed status
kubectl describe gateway cilium-gateway-lb -n gateway-system
```
Verify LoadBalancer Service:
```bash
# Check LoadBalancer service with both ports
kubectl get svc -n gateway-system

# Expected output:
# NAME                               TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)
# cilium-gateway-cilium-gateway-lb   LoadBalancer   10.43.188.227   91.98.1.173       80:31446/TCP,443:30478/TCP
```
`Key Points:`

- Both ports 80 (HTTP) and 443 (HTTPS) should be exposed
- EXTERNAL-IP is your Hetzner Load Balancer public IP
- Gateway status should show PROGRAMMED: True
- Both listeners (http and https) should be in Accepted and Programmed state

### Step 10: Deploy Sample Application
Deploy a sample Nginx application to test the Gateway.
File: `nginx-demo.yaml`
```yaml
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: default
  labels:
    app: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo-service
  namespace: default
  labels:
    app: nginx-demo
spec:
  type: ClusterIP
  selector:
    app: nginx-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
---
# HTTPRoute for HTTPS
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-demo-route
  namespace: default
spec:
  parentRefs:
    - name: cilium-gateway-lb
      namespace: gateway-system
      sectionName: https  # Routes to HTTPS listener
  hostnames:
    - "nginx.k8s.example.com"  # ⚠️ CHANGE to your subdomain
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: nginx-demo-service
          port: 80
---
# HTTPRoute for HTTP to HTTPS redirect
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-demo-redirect
  namespace: default
spec:
  parentRefs:
    - name: cilium-gateway-lb
      namespace: gateway-system
      sectionName: http  # Routes to HTTP listener
  hostnames:
    - "nginx.k8s.example.com"  # ⚠️ CHANGE to your subdomain
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```
Configuration Notes:

- hostnames: Must be a subdomain under your wildcard certificate (e.g., nginx.k8s.example.com for *.k8s.example.com certificate)
- parentRefs.sectionName: https: Routes traffic through the HTTPS listener with TLS termination
- RequestRedirect: Automatically redirects HTTP requests to HTTPS with a 301 status code

Apply Application:
```bash
kubectl apply -f nginx-demo.yaml

# Wait for deployment to be ready
kubectl rollout status deployment/nginx-demo -n default
```
Verify Application:
```bash
# Check pods are running
kubectl get pods -n default -l app=nginx-demo

# Expected output:
# NAME                          READY   STATUS    RESTARTS   AGE
# nginx-demo-5d5b5c5c5c-xxxxx   1/1     Running   0          30s
# nginx-demo-5d5b5c5c5c-yyyyy   1/1     Running   0          30s

# Check service
kubectl get svc nginx-demo-service -n default

# Check HTTPRoutes
kubectl get httproute -n default

# Expected output:
# NAME                  HOSTNAMES                     AGE
# nginx-demo-route      ["nginx.k8s.example.com"]     1m
# nginx-demo-redirect   ["nginx.k8s.example.com"]     1m
```
Verify Gateway Has Attached Routes:
```bash
kubectl describe gateway cilium-gateway-lb -n gateway-system | grep "Attached Routes"

# Expected output should show:
# Attached Routes:  1  (for http listener)
# Attached Routes:  1  (for https listener)
```

---

### Step 11: Configure DNS Records

Add DNS records in Hetzner DNS Console to point to your LoadBalancer IP.

#### Option 1: Wildcard DNS Record (Recommended)

Add a single wildcard record that covers all subdomains:
```
Type: A
Name: *.k8s.example.com
Value: 91.98.1.173  # Your LoadBalancer EXTERNAL-IP
TTL: 300
```

**Benefits:**
- All subdomains automatically resolve (e.g., `app1.k8s.example.com`, `app2.k8s.example.com`)
- No need to add individual records for each application

#### Option 2: Individual DNS Records

Add specific records for each subdomain:
```
Type: A
Name: nginx.k8s.example.com
Value: 91.98.1.173
TTL: 300
```
## 📚 Configuration Reference

### Gateway API Resources
`Gateway`

- Purpose: Entry point for traffic into the cluster
- Scope: Namespace-scoped (in gateway-system)
- Key Features:
    - Defines listeners (HTTP, HTTPS, TLS, etc.)
    - References TLS certificates
    - Controls which namespaces can attach routes

`HTTPRoute`

- Purpose: Routes HTTP(S) traffic to backend services
- Scope: Namespace-scoped (same as application)
- Key Features:
    - Path-based routing (/api, /admin)
    - Header-based routing
    - Request/response modification
    - Traffic splitting (canary deployments)
    - Redirects and rewrites

`GatewayClass`

- Purpose: Defines the controller that implements the Gateway
- Scope: Cluster-scoped
- For Cilium: Automatically created with name cilium

`Certificate Renewal`
Let's Encrypt certificates are valid for 90 days. cert-manager automatically:

- Renews certificates at 60 days before expiration
- Creates a new Certificate Request
- Completes DNS-01 challenge via Hetzner webhook
- Updates the TLS secret
- Gateway automatically picks up the new certificate (no restart needed)

## 🔧 Troubleshooting
Certificate Not Issuing
Symptoms:

- Certificate status shows Ready:   False 
- cert-manager logs show errors

Debug Steps:
```bash
# 1. Check certificate status
kubectl describe certificate wildcard-k8s-example -n gateway-system

# 2. Check certificate request
kubectl get certificaterequest -n gateway-system
kubectl describe certificaterequest -n gateway-system

# 3. Check ACME order
kubectl get order -n gateway-system
kubectl describe order -n gateway-system

# 4. Check challenge
kubectl get challenge -n gateway-system
kubectl describe challenge -n gateway-system

# 5. Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager --tail=100

# 6. Check webhook logs
kubectl logs -n cert-manager -l app.kubernetes.io/name=cert-manager-webhook-hetzner
```
## 🎯 Next Steps
Deploy More Applications
To expose additional applications, simply create:

- Deployment & Service (standard Kubernetes resources)
- HTTPRoute pointing to your Gateway
Example template:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: my-namespace
spec:
  parentRefs:
    - name: cilium-gateway-lb
      namespace: gateway-system
      sectionName: https
  hostnames:
    - "my-app.k8s.example.com"  # Must be covered by wildcard cert
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app-service
          port: 80
```
#### Advanced Features
Explore these Gateway API capabilities:

- Path-based routing: Route /api and /admin to different services
- Header-based routing: Route based on HTTP headers
- Traffic splitting: A/B testing and canary deployments
- Request/response modification: Rewrite headers, paths
- Multiple Gateways: Separate internal and external traffic
- TLSRoute: Non-HTTP TLS traffic routing

Happy Deploying! 🚀