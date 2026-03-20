# Local Installation with Colima (Native k3s)

Use this path when your only local tool is Colima. No k3d required — OpenChoreo runs directly on Colima's built-in k3s cluster.

> **Recommended for:** macOS users with Colima already installed.
>
> **Browser access:** Uses `openchoreo.localhost` domains. Chrome resolves `*.localhost` to `127.0.0.1` natively (secure context — no flags needed, no `/etc/hosts` needed). A port-forward script tunnels traffic from `localhost:8080` into the cluster.

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

Activate the context:
```bash
kubectl config use-context colima
kubectl get nodes
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
kubectl apply -f - <<'EOF'
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

> **Why token auth?** Kubernetes auth in OpenBao requires the token reviewer JWT to match the cluster's API server CA. Token auth with the dev root token is reliable for local development.

## Step 3 — Install control plane

Uses the official k3d single-cluster values files directly — they already use `openchoreo.localhost` as the domain. No domain substitution needed.

```bash
# Install Thunder (identity provider)
helm upgrade --install thunder oci://ghcr.io/asgardeo/helm-charts/thunder \
  --namespace thunder --create-namespace --version 0.26.0 \
  --values https://raw.githubusercontent.com/openchoreo/openchoreo/refs/tags/v1.0.0-rc.1/install/k3d/common/values-thunder.yaml

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
helm upgrade --install openchoreo-control-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-control-plane \
  --version 1.0.0-rc.1 \
  --namespace openchoreo-control-plane \
  --create-namespace \
  --timeout 15m \
  --values https://raw.githubusercontent.com/openchoreo/openchoreo/main/install/k3d/single-cluster/values-cp.yaml

kubectl wait -n openchoreo-control-plane --for=condition=available --timeout=300s deployment --all
```

### Fix Backstage auth environment

The Backstage app-config has auth under the `development:` key, but the chart sets `NODE_ENV=production`. Patch it to `development` so the auth provider loads:

```bash
kubectl patch deployment backstage -n openchoreo-control-plane \
  --field-manager helm \
  --type=json \
  -p '[{"op":"replace","path":"/spec/template/spec/containers/0/env/0/value","value":"development"}]'

kubectl rollout status deployment/backstage -n openchoreo-control-plane --timeout=120s
```

> **Note:** This patch survives pod restarts but will be reset on the next `helm upgrade`. Re-apply after any control plane upgrade.

### Configure CoreDNS for in-cluster resolution

Pods inside the cluster cannot resolve `*.openchoreo.localhost` via the upstream DNS — it needs to point to the gateway ClusterIP. Get the ClusterIP and patch CoreDNS:

```bash
CP_GW_IP=$(kubectl get svc gateway-default -n openchoreo-control-plane \
  -o jsonpath='{.spec.clusterIP}')
echo "CP gateway ClusterIP: $CP_GW_IP"

kubectl patch configmap coredns -n kube-system --type merge -p "{
  \"data\": {
    \"Corefile\": \".:53 {\n    errors\n    health\n    ready\n    kubernetes cluster.local in-addr.arpa ip6.arpa {\n      pods insecure\n      fallthrough in-addr.arpa ip6.arpa\n    }\n    hosts {\n      ${CP_GW_IP} openchoreo.localhost\n      ${CP_GW_IP} api.openchoreo.localhost\n      ${CP_GW_IP} thunder.openchoreo.localhost\n      fallthrough\n    }\n    prometheus :9153\n    forward . /etc/resolv.conf\n    cache 30\n    loop\n    reload\n    loadbalance\n}\n\"
  }
}"

kubectl rollout restart deployment/coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system --timeout=60s
```

> **Why CoreDNS and not /etc/hosts?** `/etc/hosts` on macOS only affects the host machine. Pods inside the cluster have their own DNS via CoreDNS. Without this override, in-cluster HTTP calls (Backstage → Thunder token endpoint, API server → Thunder JWKS) fail because `*.openchoreo.localhost` would fall through to Colima's internal DNS (192.168.5.1) which returns `127.0.0.1` — correct for the host but wrong for pods (loopback inside the pod, not the gateway).

Verify console:
```bash
curl -s "http://openchoreo.localhost:8080" | head -1
# Requires the port-forward to be running (see "Every session" below)
```

Console URL: **`http://openchoreo.localhost:8080`** — `admin@openchoreo.dev` / `Admin@123`

Thunder admin: **`http://thunder.openchoreo.localhost:8080/develop`** — `admin` / `admin`

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

# k3d values-dp.yaml uses httpPort:19080 — no port conflict with CP gateway on single-node k3s
helm upgrade --install openchoreo-data-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-data-plane \
  --version 1.0.0-rc.1 \
  --namespace openchoreo-data-plane \
  --create-namespace \
  --timeout 15m \
  --values https://raw.githubusercontent.com/openchoreo/openchoreo/main/install/k3d/single-cluster/values-dp.yaml

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
          host: "openchoreoapis.openchoreo.localhost"
          listenerName: http
          port: 19080
        name: gateway-default
        namespace: openchoreo-data-plane
EOF
```

Add the data plane gateway to CoreDNS:

```bash
DP_GW_IP=$(kubectl get svc gateway-default -n openchoreo-data-plane \
  -o jsonpath='{.spec.clusterIP}')
echo "DP gateway ClusterIP: $DP_GW_IP"

CP_GW_IP=$(kubectl get svc gateway-default -n openchoreo-control-plane \
  -o jsonpath='{.spec.clusterIP}')

kubectl patch configmap coredns -n kube-system --type merge -p "{
  \"data\": {
    \"Corefile\": \".:53 {\n    errors\n    health\n    ready\n    kubernetes cluster.local in-addr.arpa ip6.arpa {\n      pods insecure\n      fallthrough in-addr.arpa ip6.arpa\n    }\n    hosts {\n      ${CP_GW_IP} openchoreo.localhost\n      ${CP_GW_IP} api.openchoreo.localhost\n      ${CP_GW_IP} thunder.openchoreo.localhost\n      ${DP_GW_IP} openchoreoapis.openchoreo.localhost\n      fallthrough\n    }\n    prometheus :9153\n    forward . /etc/resolv.conf\n    cache 30\n    loop\n    reload\n    loadbalance\n}\n\"
  }
}"

kubectl rollout restart deployment/coredns -n kube-system
kubectl rollout status deployment/coredns -n kube-system --timeout=60s
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

## Access URLs

| Service | URL |
|---|---|
| Console | `http://openchoreo.localhost:8080` |
| API | `http://api.openchoreo.localhost:8080` |
| Thunder (IdP admin) | `http://thunder.openchoreo.localhost:8080/develop` |
| Deployed apps | `http://<route>.openchoreoapis.openchoreo.localhost:19080` |

**Login:** `admin@openchoreo.dev` / `Admin@123`

## Browser access

Chrome resolves `*.localhost` to `127.0.0.1` natively — no `/etc/hosts` entries and no Chrome flags needed. The `openchoreo.localhost` domains are also treated as **secure contexts** by Chrome, so `window.crypto.randomUUID` and other Web Crypto APIs work without HTTPS.

### Every session — start the port-forward

Port-forwards die when the terminal closes. Run this in a dedicated terminal after `colima start`:

```bash
bash ~/openchoreo-aws/start-openchoreo-portforward.sh
```

Keep that terminal open. The script auto-restarts both the control-plane (`:8080`) and data-plane (`:19080`) port-forwards if they die.

## Colima-specific notes

- **`--network-address` is required** — without it, LoadBalancer services get no external IP.
- **`colima start` without flags resumes an existing VM** with its saved config. If the VM was recreated (disk reset), resource settings may default to 2 CPU / 2GB; always use explicit flags when unsure.
- **After restart, run `kubectl config use-context colima`** — if you have other clusters, the context may not switch automatically.
- **ClusterSecretStore uses token auth** (not kubernetes auth) — token auth with the dev root token is reliable for local dev.
- **CoreDNS override persists across restarts** — `colima stop` / `colima start` preserves all Kubernetes state including CoreDNS overrides. Re-apply only if you recreate the VM.
- **`NODE_ENV=development` patch** — required after any `helm upgrade` of the control plane (the chart resets it to `production`). The patch command is in Step 3.
- **k3d values files used directly** — the official `install/k3d/single-cluster/` values files already use `openchoreo.localhost` as the domain. No `sed` substitution needed on Colima.

## Cleanup

```bash
# Remove OpenChoreo from the cluster (follow references/cleanup.md)

# Or delete the VM entirely:
colima delete
```
