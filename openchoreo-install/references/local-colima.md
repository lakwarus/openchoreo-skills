# Local Installation with Colima (Native k3s)

Use this path when your only local tool is Colima. No k3d required — OpenChoreo runs directly on Colima's built-in k3s cluster.

> **Recommended for:** macOS users with Colima already installed who want the simplest possible prerequisite list.
>
> **Browser access note:** The Colima VM IP (`192.168.64.x`) is reachable from the macOS terminal but Chrome may report `ERR_ADDRESS_UNREACHABLE`. Safari and Firefox usually work directly. For guaranteed browser access from any app, use the [k3d path](local-k3d.md) instead — it maps to `127.0.0.1`.

## Prerequisites

| Tool | Install |
|---|---|
| Colima | `brew install colima` |
| kubectl | `brew install kubectl` (or use `~/.colima/default/colima.yaml` path after Colima starts) |
| Helm | `brew install helm` |

Verify:
```bash
colima version && kubectl version --client && helm version --short
```

## Step 1 — Start Colima with Kubernetes

**Option A — Two separate commands** (matches your existing workflow):
```bash
# Start the VM
colima start --vm-type=vz --vz-rosetta --cpu 7 --memory 9 --network-address

# Enable Kubernetes (restarts with k3s)
colima start --kubernetes --kubernetes-version v1.33.4+k3s1
```

**Option B — Single combined command** (for a fresh machine):
```bash
colima start \
  --vm-type=vz \
  --vz-rosetta \
  --cpu 7 \
  --memory 9 \
  --disk 100 \
  --kubernetes \
  --kubernetes-version v1.33.4+k3s1 \
  --network-address
```

> `--network-address` assigns a routable IP to the VM (e.g. `192.168.64.3`). Required for nip.io domains and external service access.

Verify the cluster is up:
```bash
kubectl get nodes
# NAME     STATUS   ROLES                  AGE   VERSION
# colima   Ready    control-plane,master   ...   v1.33.x+k3s1
```

## Step 2 — Get VM IP

```bash
export COLIMA_IP=$(kubectl get node \
  -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "Colima IP: $COLIMA_IP"
```

All domain variables in the steps below use this IP via nip.io.

## Step 3 — Install prerequisites

```bash
# Gateway API CRDs
kubectl apply --server-side \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/experimental-install.yaml

# cert-manager
helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --create-namespace --version v1.19.2 \
  --set crds.enabled=true --wait --timeout 180s

# External Secrets Operator
helm upgrade --install external-secrets oci://ghcr.io/external-secrets/charts/external-secrets \
  --namespace external-secrets --create-namespace --version 1.3.2 \
  --set installCRDs=true --wait --timeout 180s

# kgateway CRDs
helm upgrade --install kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds \
  --create-namespace --namespace openchoreo-control-plane --version v2.2.1

# kgateway controller
helm upgrade --install kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
  --namespace openchoreo-control-plane --create-namespace --version v2.2.1 \
  --set controller.extraEnv.KGW_ENABLE_GATEWAY_API_EXPERIMENTAL_FEATURES=true

# OpenBao (secret backend)
helm upgrade --install openbao oci://ghcr.io/openbao/charts/openbao \
  --namespace openbao --create-namespace --version 0.25.6 \
  --set server.dev.enabled=true \
  --set server.dev.devRootToken=root \
  --wait --timeout 300s

# ClusterSecretStore
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-openbao
  namespace: openbao
---
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: default
spec:
  provider:
    vault:
      server: "http://openbao.openbao.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "openchoreo-secret-writer-role"
          serviceAccountRef:
            name: "external-secrets-openbao"
            namespace: "openbao"
EOF
```

## Step 4 — Install control plane

```bash
# Create namespace
kubectl create namespace openchoreo-control-plane --dry-run=client -o yaml | kubectl apply -f -

# Self-signed CA
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: openchoreo-selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: openchoreo-ca
  namespace: openchoreo-control-plane
spec:
  isCA: true
  commonName: openchoreo-ca
  secretName: openchoreo-ca
  issuerRef:
    name: openchoreo-selfsigned
    kind: ClusterIssuer
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: openchoreo-ca
spec:
  ca:
    secretName: openchoreo-ca
EOF

kubectl wait -n openchoreo-control-plane \
  --for=condition=Ready certificate/openchoreo-ca --timeout=60s

# Install control plane with TLS disabled initially
helm upgrade --install openchoreo-control-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-control-plane \
  --version 1.0.0-rc.1 \
  --namespace openchoreo-control-plane \
  --create-namespace \
  --timeout 15m \
  --values - <<EOF
gateway:
  tls:
    enabled: false
EOF

kubectl wait -n openchoreo-control-plane --for=condition=available --timeout=300s deployment --all

# Get LB IP (= Colima VM IP via klipper-lb)
export CP_LB_IP=$(kubectl get svc gateway-default -n openchoreo-control-plane \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Control plane LB IP: $CP_LB_IP"
export CP_BASE_DOMAIN="openchoreo.${CP_LB_IP//./-}.nip.io"
export CP_DOMAIN="console.${CP_BASE_DOMAIN}"
echo "Control plane domain: ${CP_DOMAIN}"

# TLS certificate
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cp-gateway-tls
  namespace: openchoreo-control-plane
spec:
  secretName: cp-gateway-tls
  issuerRef:
    name: openchoreo-ca
    kind: ClusterIssuer
  dnsNames:
    - "*.${CP_BASE_DOMAIN}"
    - "${CP_DOMAIN}"
  privateKey:
    rotationPolicy: Always
EOF

kubectl wait --for=condition=Ready certificate/cp-gateway-tls \
  -n openchoreo-control-plane --timeout=60s

# Install Thunder (identity provider)
helm upgrade --install thunder oci://ghcr.io/asgardeo/helm-charts/thunder \
  --namespace thunder --create-namespace --version 0.26.0 \
  --values - <<EOF
service:
  type: LoadBalancer
endpoints:
  baseUrl: "https://thunder.${CP_BASE_DOMAIN}"
EOF

kubectl wait -n thunder --for=condition=available --timeout=300s \
  deployment -l app.kubernetes.io/name=thunder

# Backstage secrets
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: backstage-secrets
  namespace: openchoreo-control-plane
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: default
  target:
    name: backstage-secrets
  data:
  - secretKey: backend-secret
    remoteRef:
      key: backstage-backend-secret
      property: value
  - secretKey: client-secret
    remoteRef:
      key: backstage-client-secret
      property: value
  - secretKey: jenkins-api-key
    remoteRef:
      key: backstage-jenkins-api-key
      property: value
EOF

# Reconfigure control plane with real hostnames and TLS
helm upgrade openchoreo-control-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-control-plane \
  --version 1.0.0-rc.1 \
  --namespace openchoreo-control-plane \
  --reuse-values \
  --timeout 10m \
  --values - <<EOF
backstage:
  secretName: backstage-secrets
  appConfig:
    app:
      baseUrl: "https://console.${CP_BASE_DOMAIN}"
    backend:
      baseUrl: "https://api.${CP_BASE_DOMAIN}"
  oidc:
    clientId: backstage
    issuer: "https://thunder.${CP_BASE_DOMAIN}"
    tokenUrl: "https://thunder.${CP_BASE_DOMAIN}/oauth2/token"
    authorizationUrl: "https://thunder.${CP_BASE_DOMAIN}/oauth2/authorize"
    userInfoUrl: "https://thunder.${CP_BASE_DOMAIN}/oauth2/userinfo"
    jwksUrl: "https://thunder.${CP_BASE_DOMAIN}/oauth2/jwks"
    tlsInsecureSkipVerify: "true"
gateway:
  tls:
    enabled: true
    hostname: "*.${CP_BASE_DOMAIN}"
    certificateRefs:
      - name: cp-gateway-tls
  http:
    hostnames:
      - "console.${CP_BASE_DOMAIN}"
      - "api.${CP_BASE_DOMAIN}"
      - "thunder.${CP_BASE_DOMAIN}"
EOF
```

Verify console:
```bash
curl -sk "https://console.${CP_BASE_DOMAIN}/healthz" | head -c 100
```

Console URL: `https://console.${CP_BASE_DOMAIN}` — `admin@openchoreo.dev` / `Admin@123`

## Step 5 — Apply default resources

```bash
kubectl apply -f https://raw.githubusercontent.com/openchoreo/openchoreo/refs/tags/v1.0.0-rc.1/samples/getting-started/all.yaml
kubectl label namespace default openchoreo.dev/control-plane=true
```

## Step 6 — Install data plane

> **Single-node port offset required:** use `httpPort=8080` / `httpsPort=8443` to avoid conflict with the control plane gateway on ports 80/443.

```bash
kubectl create namespace openchoreo-data-plane --dry-run=client -o yaml | kubectl apply -f -

kubectl get secret cluster-gateway-ca -n openchoreo-control-plane \
  -o jsonpath='{.data.ca\.crt}' | base64 -d | \
  kubectl create configmap cluster-gateway-ca \
    --from-file=ca.crt=/dev/stdin \
    -n openchoreo-data-plane \
    --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install openchoreo-data-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-data-plane \
  --version 1.0.0-rc.1 \
  --namespace openchoreo-data-plane \
  --create-namespace \
  --timeout 15m \
  --set gateway.httpPort=8080 \
  --set gateway.httpsPort=8443 \
  --set gateway.tls.enabled=false

kubectl wait -n openchoreo-data-plane --for=condition=available --timeout=300s deployment --all

export DP_LB_IP=$(kubectl get svc gateway-default -n openchoreo-data-plane \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export DP_BASE_DOMAIN="openchoreo.dp.${DP_LB_IP//./-}.nip.io"
echo "Data plane domain: ${DP_BASE_DOMAIN}"

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dp-gateway-tls
  namespace: openchoreo-data-plane
spec:
  secretName: dp-gateway-tls
  issuerRef:
    name: openchoreo-ca
    kind: ClusterIssuer
  dnsNames:
    - "*.${DP_BASE_DOMAIN}"
  privateKey:
    rotationPolicy: Always
EOF

kubectl wait --for=condition=Ready certificate/dp-gateway-tls \
  -n openchoreo-data-plane --timeout=60s

helm upgrade openchoreo-data-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-data-plane \
  --version 1.0.0-rc.1 \
  --namespace openchoreo-data-plane \
  --reuse-values \
  --set gateway.tls.enabled=true \
  --set "gateway.tls.hostname=*.${DP_BASE_DOMAIN}" \
  --set "gateway.tls.certificateRefs[0].name=dp-gateway-tls"

AGENT_CA=$(kubectl get secret cluster-agent-tls \
  -n openchoreo-data-plane -o jsonpath='{.data.ca\.crt}' | base64 -d)

DP_HTTPS_PORT=$(kubectl get gateway gateway-default -n openchoreo-data-plane \
  -o jsonpath='{.spec.listeners[?(@.protocol=="HTTPS")].port}')

kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: ClusterDataPlane
metadata:
  name: default
spec:
  planeID: default
  clusterAgent:
    clientCA:
      value: |
$(echo "$AGENT_CA" | sed 's/^/        /')
  secretStoreRef:
    name: default
  gateway:
    ingress:
      external:
        http:
          host: "apps.${DP_BASE_DOMAIN}"
          listenerName: https
          port: ${DP_HTTPS_PORT}
        name: gateway-default
        namespace: openchoreo-data-plane
EOF
```

## Step 7 — Install workflow plane (optional)

Required for source builds. Skip if you only deploy pre-built images.

```bash
kubectl create namespace openchoreo-workflow-plane --dry-run=client -o yaml | kubectl apply -f -

kubectl get secret cluster-gateway-ca -n openchoreo-control-plane \
  -o jsonpath='{.data.ca\.crt}' | base64 -d | \
  kubectl create configmap cluster-gateway-ca \
    --from-file=ca.crt=/dev/stdin \
    -n openchoreo-workflow-plane \
    --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install openchoreo-workflow-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-workflow-plane \
  --version 1.0.0-rc.1 \
  --namespace openchoreo-workflow-plane \
  --create-namespace \
  --set clusterAgent.tls.generateCerts=true

# Workflow templates (checkout + build coordinator + publish to ttl.sh)
kubectl apply \
  -f https://raw.githubusercontent.com/openchoreo/openchoreo/main/samples/getting-started/workflow-templates/checkout-source.yaml \
  -f https://raw.githubusercontent.com/openchoreo/openchoreo/main/samples/getting-started/workflow-templates.yaml \
  -f https://raw.githubusercontent.com/openchoreo/openchoreo/main/samples/getting-started/workflow-templates/publish-image.yaml

AGENT_CA=$(kubectl get secret cluster-agent-tls \
  -n openchoreo-workflow-plane -o jsonpath='{.data.ca\.crt}' | base64 -d)

kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: ClusterWorkflowPlane
metadata:
  name: default
spec:
  planeID: default
  clusterAgent:
    clientCA:
      value: |
$(echo "$AGENT_CA" | sed 's/^/        /')
  secretStoreRef:
    name: default
EOF
```

> **Source builds push to [ttl.sh](https://ttl.sh)** — images expire after 24 hours. For persistent builds, set up a local registry or use a real registry.

## Step 8 — Install observability plane (optional)

> **Single-node port offset required:** use `httpPort=9080` / `httpsPort=9443`.

```bash
kubectl create namespace openchoreo-observability-plane --dry-run=client -o yaml | kubectl apply -f -

kubectl get secret cluster-gateway-ca -n openchoreo-control-plane \
  -o jsonpath='{.data.ca\.crt}' | base64 -d | \
  kubectl create configmap cluster-gateway-ca \
    --from-file=ca.crt=/dev/stdin \
    -n openchoreo-observability-plane \
    --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: opensearch-admin-credentials
  namespace: openchoreo-observability-plane
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: default
  target:
    name: opensearch-admin-credentials
  data:
  - secretKey: username
    remoteRef:
      key: opensearch-username
      property: value
  - secretKey: password
    remoteRef:
      key: opensearch-password
      property: value
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: observer-secret
  namespace: openchoreo-observability-plane
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: default
  target:
    name: observer-secret
  data:
  - secretKey: OPENSEARCH_USERNAME
    remoteRef:
      key: opensearch-username
      property: value
  - secretKey: OPENSEARCH_PASSWORD
    remoteRef:
      key: opensearch-password
      property: value
  - secretKey: UID_RESOLVER_OAUTH_CLIENT_SECRET
    remoteRef:
      key: observer-oauth-client-secret
      property: value
EOF

kubectl wait -n openchoreo-observability-plane \
  --for=condition=Ready externalsecret/opensearch-admin-credentials \
  externalsecret/observer-secret --timeout=60s

# Core observability plane (TLS disabled initially, with port offsets)
helm upgrade --install openchoreo-observability-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-observability-plane \
  --version 1.0.0-rc.1 \
  --namespace openchoreo-observability-plane \
  --create-namespace \
  --timeout 25m \
  --set gateway.httpPort=9080 \
  --set gateway.httpsPort=9443 \
  --values - <<EOF
observer:
  openSearchSecretName: opensearch-admin-credentials
  secretName: observer-secret
gateway:
  tls:
    enabled: false
EOF

# Modules
helm upgrade --install observability-logs-opensearch \
  oci://ghcr.io/openchoreo/helm-charts/observability-logs-opensearch \
  --namespace openchoreo-observability-plane --version 0.3.8 \
  --set openSearchSetup.openSearchSecretName="opensearch-admin-credentials"

helm upgrade --install observability-metrics-prometheus \
  oci://ghcr.io/openchoreo/helm-charts/observability-metrics-prometheus \
  --namespace openchoreo-observability-plane --version 0.2.4

helm upgrade --install observability-traces-opensearch \
  oci://ghcr.io/openchoreo/helm-charts/observability-tracing-opensearch \
  --namespace openchoreo-observability-plane --version 0.3.7 \
  --set openSearch.enabled=false \
  --set openSearchSetup.openSearchSecretName="opensearch-admin-credentials"

# Get observability LB IP and domain
export OBS_LB_IP=$(kubectl get svc gateway-default -n openchoreo-observability-plane \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export OBS_BASE_DOMAIN="openchoreo.obs.${OBS_LB_IP//./-}.nip.io"
export OBS_DOMAIN="observer.${OBS_BASE_DOMAIN}"
echo "Observability domain: ${OBS_DOMAIN}"

# TLS certificate
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: obs-gateway-tls
  namespace: openchoreo-observability-plane
spec:
  secretName: obs-gateway-tls
  issuerRef:
    name: openchoreo-ca
    kind: ClusterIssuer
  dnsNames:
    - "*.${OBS_BASE_DOMAIN}"
    - "${OBS_DOMAIN}"
  privateKey:
    rotationPolicy: Always
EOF

kubectl wait --for=condition=Ready certificate/obs-gateway-tls \
  -n openchoreo-observability-plane --timeout=60s

# Reconfigure with real hostnames and TLS
helm upgrade openchoreo-observability-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-observability-plane \
  --version 1.0.0-rc.1 \
  --namespace openchoreo-observability-plane \
  --reuse-values \
  --timeout 10m \
  --values - <<EOF
observer:
  openSearchSecretName: opensearch-admin-credentials
  secretName: observer-secret
  controlPlaneApiUrl: "https://api.${CP_BASE_DOMAIN}"
  http:
    hostnames:
      - "${OBS_DOMAIN}"
  cors:
    allowedOrigins:
      - "https://console.${CP_BASE_DOMAIN}"
  authzTlsInsecureSkipVerify: true
security:
  oidc:
    issuer: "https://thunder.${CP_BASE_DOMAIN}"
    jwksUrl: "https://thunder.${CP_BASE_DOMAIN}/oauth2/jwks"
    tokenUrl: "https://thunder.${CP_BASE_DOMAIN}/oauth2/token"
    jwksUrlTlsInsecureSkipVerify: "true"
    uidResolverTlsInsecureSkipVerify: "true"
gateway:
  tls:
    enabled: true
    hostname: "*.${OBS_BASE_DOMAIN}"
    certificateRefs:
      - name: obs-gateway-tls
EOF

# Enable Fluent Bit
helm upgrade observability-logs-opensearch \
  oci://ghcr.io/openchoreo/helm-charts/observability-logs-opensearch \
  --namespace openchoreo-observability-plane --version 0.3.8 --reuse-values \
  --set fluent-bit.enabled=true

# Register and link
AGENT_CA=$(kubectl get secret cluster-agent-tls \
  -n openchoreo-observability-plane -o jsonpath='{.data.ca\.crt}' | base64 -d)

OBS_HTTPS_PORT=$(kubectl get gateway gateway-default -n openchoreo-observability-plane \
  -o jsonpath='{.spec.listeners[?(@.protocol=="HTTPS")].port}')
OBS_PORT_SUFFIX=$([ "$OBS_HTTPS_PORT" = "443" ] && echo "" || echo ":${OBS_HTTPS_PORT}")

kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: ClusterObservabilityPlane
metadata:
  name: default
spec:
  planeID: default
  clusterAgent:
    clientCA:
      value: |
$(echo "$AGENT_CA" | sed 's/^/        /')
  observerURL: https://${OBS_DOMAIN}${OBS_PORT_SUFFIX}
EOF

kubectl patch clusterdataplane default --type merge \
  -p '{"spec":{"observabilityPlaneRef":{"kind":"ClusterObservabilityPlane","name":"default"}}}'

kubectl patch clusterworkflowplane default --type merge \
  -p '{"spec":{"observabilityPlaneRef":{"kind":"ClusterObservabilityPlane","name":"default"}}}' \
  2>/dev/null || true
```

## Access URLs

All URLs use your Colima VM IP via nip.io. Replace `$CP_BASE_DOMAIN` with the value printed in Step 4.

| Service | URL |
|---|---|
| Console | `https://console.$CP_BASE_DOMAIN` |
| API | `https://api.$CP_BASE_DOMAIN` |
| Thunder (IdP admin) | `https://thunder.$CP_BASE_DOMAIN` |
| Deployed apps | `https://apps.$DP_BASE_DOMAIN:<port>` |
| Observer | `https://$OBS_DOMAIN:<port>` |

**Login:** `admin@openchoreo.dev` / `Admin@123`

## Browser access

`curl -sk` works from the terminal. For browsers:

- **Safari / Firefox** — connect to `192.168.64.x` directly; usually work without extra steps.
- **Chrome** — may report `ERR_ADDRESS_UNREACHABLE` due to app sandbox restrictions on virtual network interfaces.

**Workaround for Chrome:** run kubectl port-forward for the services you need:

```bash
# Forward control plane console/API to localhost:8443
kubectl port-forward -n openchoreo-control-plane svc/gateway-default 8443:443

# Then add to /etc/hosts (run once as admin):
echo "127.0.0.1 console.${CP_BASE_DOMAIN} api.${CP_BASE_DOMAIN} thunder.${CP_BASE_DOMAIN}" \
  | sudo tee -a /etc/hosts
```

Access via `https://console.${CP_BASE_DOMAIN}:8443` — accept the self-signed cert warning once.

For full no-config browser access, use the [k3d path](local-k3d.md) instead.

## Colima-specific notes

- **`--network-address` is required** — without it the VM has no routable IP and LoadBalancer services show no external IP.
- **Single-node port offsets** — k3s klipper-lb binds host ports. The control plane claims 80/443, so data plane must use 8080/8443 and observability must use 9080/9443.
- **kubectl is pre-configured** — Colima automatically sets the kubeconfig context; `kubectl get nodes` should work immediately after `colima start --kubernetes`.
- **Stopping and restarting** — `colima stop` then `colima start` (no flags needed) resumes the existing VM. Kubernetes state is preserved across restarts.
- **Resizing** — to increase resources: `colima stop && colima start --cpu 7 --memory 9 --kubernetes`. Data is preserved.

## Cleanup

```bash
# Option A — delete just the OpenChoreo install (keep Colima and k8s)
# Follow references/cleanup.md

# Option B — stop Colima (VM and k8s preserved, resources freed)
colima stop

# Option C — delete the VM entirely (all data lost)
colima delete
```
