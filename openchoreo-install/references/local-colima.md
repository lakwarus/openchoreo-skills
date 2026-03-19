# Local Installation with Colima (Native k3s)

Use this path when your only local tool is Colima. No k3d required — OpenChoreo runs directly on Colima's built-in k3s cluster.

> **Recommended for:** macOS users with Colima already installed.
>
> **Browser access note:** The Colima VM IP (`192.168.64.x`) works from curl and Safari/Firefox. Chrome may report `ERR_ADDRESS_UNREACHABLE`. For guaranteed Chrome access, use the [k3d path](local-k3d.md) instead.

## Prerequisites

| Tool | Install |
|---|---|
| Colima | `brew install colima` |
| kubectl | `brew install kubectl` |
| Helm | `brew install helm` |

## Step 1 — Start Colima with Kubernetes

**Option A — Two commands** (existing Colima users):
```bash
colima start --vm-type=vz --vz-rosetta --cpu 7 --memory 9 --network-address
colima start --kubernetes --kubernetes-version v1.33.4+k3s1
```

**Option B — Single command** (fresh machine):
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

> `--network-address` is required — it assigns a routable IP to the VM (e.g. `192.168.64.3`). Without it, LoadBalancer services get no external IP.

Activate the context and get the VM IP:
```bash
kubectl config use-context colima
kubectl get nodes

export COLIMA_IP=$(kubectl get node \
  -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
export DOMAIN="openchoreo.${COLIMA_IP//./-}.nip.io"
echo "Domain: $DOMAIN   IP: $COLIMA_IP"
```

## Step 2 — Install prerequisites

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

# OpenBao — use the official k3d values file (sets up dev mode + seeds all required secrets)
helm upgrade --install openbao oci://ghcr.io/openbao/charts/openbao \
  --namespace openbao --create-namespace --version 0.25.6 \
  --values https://raw.githubusercontent.com/openchoreo/openchoreo/main/install/k3d/common/values-openbao.yaml \
  --wait --timeout 300s
```

> **OpenBao note:** The k3d `values-openbao.yaml` includes a `postStart` script that configures Kubernetes auth and seeds all required secrets (backstage, opensearch, observer, etc.). Wait for the pod to be fully Running before proceeding.

Verify the secrets were seeded:
```bash
kubectl exec -n openbao openbao-0 -- \
  bao kv get -token=root secret/backstage-backend-secret 2>/dev/null || \
  echo "Secrets not ready yet — wait 30s and retry"
```

If the postStart script didn't complete (secrets missing), run it manually:
```bash
kubectl exec -n openbao openbao-0 -- /bin/sh -c '
export BAO_ADDR=http://127.0.0.1:8200
export BAO_TOKEN=root
bao auth enable kubernetes 2>/dev/null || true
bao write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
bao policy write openchoreo-secret-writer-policy - <<POLICY
path "secret/data/*" { capabilities = ["create", "read", "update", "delete"] }
path "secret/metadata/*" { capabilities = ["create", "read", "update", "delete", "list"] }
POLICY
bao write auth/kubernetes/role/openchoreo-secret-writer-role \
  bound_service_account_names="*" \
  bound_service_account_namespaces="openbao,openchoreo-workflow-plane" \
  policies=openchoreo-secret-writer-policy ttl=20m
bao kv put secret/backstage-backend-secret value="local-dev-backend-secret"
bao kv put secret/backstage-client-secret value="backstage-portal-secret"
bao kv put secret/backstage-jenkins-api-key value="placeholder-not-in-use"
bao kv put secret/observer-oauth-client-secret value="openchoreo-observer-resource-reader-client-secret"
bao kv put secret/opensearch-username value="admin"
bao kv put secret/opensearch-password value="ThisIsTheOpenSearchPassword1"
echo Done
'
```

Create ClusterSecretStore using **token auth** (simpler than kubernetes auth for local dev):
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: openbao-root-token
  namespace: openbao
type: Opaque
stringData:
  token: root
---
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
        tokenSecretRef:
          name: openbao-root-token
          key: token
          namespace: openbao
EOF

kubectl wait --for=condition=Ready clustersecretstore/default --timeout=30s
```

> **Why token auth?** Kubernetes auth in OpenBao requires the token reviewer JWT to match the cluster's API server CA. On a fresh k3s cluster, the kubernetes auth method often fails with `403 permission denied` until the auth config is updated with the correct `kubernetes_ca_cert`. Token auth with the dev root token avoids this entirely for local development.

## Step 3 — Install control plane

The install uses the official k3d single-cluster values files with `openchoreo.localhost` replaced by your nip.io domain. This preserves all Thunder bootstrap scripts (OAuth clients, users, redirect URIs) exactly.

```bash
# Install Thunder (identity provider)
curl -fsSL https://raw.githubusercontent.com/openchoreo/openchoreo/refs/tags/v1.0.0-rc.1/install/k3d/common/values-thunder.yaml | \
  sed "s/openchoreo\.localhost/${DOMAIN}/g" | \
  helm upgrade --install thunder oci://ghcr.io/asgardeo/helm-charts/thunder \
    --namespace thunder --create-namespace --version 0.26.0 \
    --values -

kubectl wait -n thunder --for=condition=available --timeout=300s deployment -l app.kubernetes.io/name=thunder

# Backstage ExternalSecret
kubectl create namespace openchoreo-control-plane --dry-run=client -o yaml | kubectl apply -f -

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

kubectl wait -n openchoreo-control-plane \
  --for=condition=Ready externalsecret/backstage-secrets --timeout=60s

# Control plane
curl -fsSL https://raw.githubusercontent.com/openchoreo/openchoreo/main/install/k3d/single-cluster/values-cp.yaml | \
  sed "s/openchoreo\.localhost/${DOMAIN}/g" | \
  helm upgrade --install openchoreo-control-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-control-plane \
    --version 1.0.0-rc.1 \
    --namespace openchoreo-control-plane \
    --create-namespace \
    --timeout 15m \
    --values -

kubectl wait -n openchoreo-control-plane --for=condition=available --timeout=300s deployment --all
```

Verify console:
```bash
curl -s "http://${DOMAIN}:8080" | head -1
# Should return <!DOCTYPE html>
```

Console URL: **`http://${DOMAIN}:8080`** — `admin@openchoreo.dev` / `Admin@123`

Thunder admin: **`http://thunder.${DOMAIN}:8080/develop`** — `admin` / `admin`

## Step 4 — Apply default resources

```bash
kubectl apply -f https://raw.githubusercontent.com/openchoreo/openchoreo/refs/tags/v1.0.0-rc.1/samples/getting-started/all.yaml
kubectl label namespace default openchoreo.dev/control-plane=true
```

## Step 5 — Install data plane

```bash
kubectl create namespace openchoreo-data-plane --dry-run=client -o yaml | kubectl apply -f -

kubectl get secret cluster-gateway-ca -n openchoreo-control-plane \
  -o jsonpath='{.data.ca\.crt}' | base64 -d | \
  kubectl create configmap cluster-gateway-ca \
    --from-file=ca.crt=/dev/stdin \
    -n openchoreo-data-plane \
    --dry-run=client -o yaml | kubectl apply -f -

# k3d values-dp.yaml uses httpPort:19080 — works without port offset conflicts on single-node k3s
curl -fsSL https://raw.githubusercontent.com/openchoreo/openchoreo/main/install/k3d/single-cluster/values-dp.yaml | \
  sed "s/openchoreo\.localhost/${DOMAIN}/g" | \
  helm upgrade --install openchoreo-data-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-data-plane \
    --version 1.0.0-rc.1 \
    --namespace openchoreo-data-plane \
    --create-namespace \
    --timeout 15m \
    --values -

kubectl wait -n openchoreo-data-plane --for=condition=available --timeout=300s deployment --all

AGENT_CA=$(kubectl get secret cluster-agent-tls \
  -n openchoreo-data-plane -o jsonpath='{.data.ca\.crt}' | base64 -d)

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
          host: "openchoreoapis.${COLIMA_IP//./-}.nip.io"
          listenerName: http
          port: 19080
        name: gateway-default
        namespace: openchoreo-data-plane
EOF
```

## Step 6 — Install workflow plane (optional)

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
  --values https://raw.githubusercontent.com/openchoreo/openchoreo/main/install/k3d/single-cluster/values-wp.yaml

# Workflow templates
kubectl apply \
  -f https://raw.githubusercontent.com/openchoreo/openchoreo/main/samples/getting-started/workflow-templates/checkout-source.yaml \
  -f https://raw.githubusercontent.com/openchoreo/openchoreo/main/samples/getting-started/workflow-templates.yaml \
  -f https://raw.githubusercontent.com/openchoreo/openchoreo/main/samples/getting-started/workflow-templates/publish-image.yaml

kubectl wait -n openchoreo-workflow-plane --for=condition=available --timeout=300s deployment --all

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

Argo Workflows UI: `http://${COLIMA_IP}:10081`

## Access URLs

Replace `192-168-64-3` with your actual IP (dashes, not dots).

| Service | URL |
|---|---|
| Console | `http://openchoreo.192-168-64-3.nip.io:8080` |
| API | `http://api.openchoreo.192-168-64-3.nip.io:8080` |
| Thunder (IdP admin) | `http://thunder.openchoreo.192-168-64-3.nip.io:8080/develop` |
| Deployed apps | `http://openchoreoapis.192-168-64-3.nip.io:19080` |
| Argo Workflows UI | `http://192.168.64.3:10081` |

**Login:** `admin@openchoreo.dev` / `Admin@123`

## Browser access (Chrome workaround)

If Chrome shows `ERR_ADDRESS_UNREACHABLE`, run a port-forward and add a hosts entry:

```bash
# Forward console gateway to localhost
kubectl port-forward -n openchoreo-control-plane svc/gateway-default 8080:8080 &

# Add to /etc/hosts (one-time, requires sudo):
echo "127.0.0.1 openchoreo.192-168-64-3.nip.io api.openchoreo.192-168-64-3.nip.io thunder.openchoreo.192-168-64-3.nip.io" | sudo tee -a /etc/hosts
```

Then access `http://openchoreo.192-168-64-3.nip.io:8080` — Chrome will route through the local port-forward.

## Colima-specific notes

- **`--network-address` is required** — without it, LoadBalancer services show no external IP.
- **`colima start` without flags resumes an existing VM** with its saved config. If the VM was recreated (disk reset), resource settings may default to 2 CPU / 2GB; always use explicit flags when unsure.
- **After restart, run `kubectl config use-context colima`** — if you have other clusters, the context may not switch automatically.
- **ClusterSecretStore uses token auth** (not kubernetes auth) — kubernetes auth requires extra CA cert configuration in k3s that varies per cluster; root token auth is reliable for local dev.
- **Domain substitution via `sed`** — all installs use the official k3d values files with `openchoreo.localhost` substituted by the nip.io domain. This preserves Thunder's bootstrap OAuth client registrations exactly.
- **Stopping and restarting** — `colima stop` / `colima start` preserves all Kubernetes state.

## Cleanup

```bash
# Remove OpenChoreo from the cluster (follow references/cleanup.md)

# Or delete the VM entirely:
colima delete
```
