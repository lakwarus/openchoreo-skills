# CLI and Resources

<!-- Quick navigation
- [occ Installation](#occ-installation)
- [Setup and Authentication](#setup-and-authentication)
- [REST API Fallback](#rest-api-fallback-when-mcp-tools-are-missing)
- [PE-Relevant occ Commands](#pe-relevant-occ-commands)
- [Resource Schemas](#resource-schemas)
-->

## occ Installation

`occ` is the OpenChoreo CLI. It is **not installed by default** — download from GitHub releases:

```bash
# macOS ARM64 (Apple Silicon)
curl -L "https://github.com/openchoreo/openchoreo/releases/download/v0.17.0/occ_v0.17.0_darwin_arm64.tar.gz" -o /tmp/occ.tar.gz
tar -xzf /tmp/occ.tar.gz -C /tmp
chmod +x /tmp/occ
/tmp/occ version   # verify

# macOS AMD64 (Intel)
curl -L "https://github.com/openchoreo/openchoreo/releases/download/v0.17.0/occ_v0.17.0_darwin_amd64.tar.gz" -o /tmp/occ.tar.gz
tar -xzf /tmp/occ.tar.gz -C /tmp && chmod +x /tmp/occ
```

Check latest release at: https://github.com/openchoreo/openchoreo/releases/latest

**Note**: The `choreo` binary at `~/.choreo/bin/choreo` is the WSO2 commercial Choreo cloud CLI — it is a different product and cannot manage OpenChoreo resources.

## Setup and Authentication

### Point occ at the control plane

```bash
# Local setup
/tmp/occ config controlplane update default --url http://api.openchoreo.localhost:8080

# AWS POC setup (if needed)
/tmp/occ config controlplane update default --url https://api.aws.openchoreo-poc.choreo.dev
```

### Authentication

**occ login with `service_mcp_client` does NOT work** — this client is not configured for the occ OIDC flow. The error will be `unauthorized_client: Client is not allowed to use the specified token endpoint authentication method`.

Instead, get a bearer token via curl and use the REST API directly (see below).

```bash
# Local setup — get a token
MCP_TOKEN=$(curl -s -X POST "http://thunder.openchoreo.localhost:8080/oauth2/token" \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -u 'service_mcp_client:service_mcp_client_secret' \
  -d 'grant_type=client_credentials' | jq -r '.access_token')

# Verify the API is reachable
curl -s -H "Authorization: Bearer $MCP_TOKEN" \
  "http://api.openchoreo.localhost:8080/api/v1/namespaces"
```

The token and identity server URLs are in `local-refresh-openchoreo-mcp.sh` at the repo root.

## REST API Fallback (when MCP tools are missing)

The MCP server does **not** expose `create_environment` or `create_deployment_pipeline`. Use the REST API directly.

### API base pattern

```
http://api.openchoreo.localhost:8080/api/v1/namespaces/{namespace}/{resource}
```

### Create an Environment

```bash
curl -s -X POST "http://api.openchoreo.localhost:8080/api/v1/namespaces/default/environments" \
  -H "Authorization: Bearer $MCP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "metadata": {
      "name": "qa",
      "namespace": "default",
      "labels": {"openchoreo.dev/name": "qa"},
      "annotations": {
        "openchoreo.dev/display-name": "QA",
        "openchoreo.dev/description": "QA"
      }
    },
    "spec": {
      "dataPlaneRef": {"kind": "DataPlane", "name": "default"},
      "isProduction": false
    }
  }'
```

Set `"isProduction": true` for production environments.

### Create a DeploymentPipeline

```bash
curl -s -X POST "http://api.openchoreo.localhost:8080/api/v1/namespaces/default/deploymentpipelines" \
  -H "Authorization: Bearer $MCP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "metadata": {
      "name": "my-pipeline",
      "namespace": "default",
      "labels": {"openchoreo.dev/name": "my-pipeline"},
      "annotations": {
        "openchoreo.dev/display-name": "My Pipeline",
        "openchoreo.dev/description": "dev → staging → production"
      }
    },
    "spec": {
      "promotionPaths": [
        {"sourceEnvironmentRef": "development", "targetEnvironmentRefs": [{"name": "staging", "requiresApproval": false}]},
        {"sourceEnvironmentRef": "staging", "targetEnvironmentRefs": [{"name": "production", "requiresApproval": false}]}
      ]
    }
  }'
```

### Update a Project's pipeline assignment

Projects are created via MCP (`create_project`) but default to `deploymentPipelineRef: default`. Use PUT to reassign:

```bash
curl -s -X PUT "http://api.openchoreo.localhost:8080/api/v1/namespaces/default/projects/foo" \
  -H "Authorization: Bearer $MCP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"metadata":{"name":"foo","namespace":"default"},"spec":{"deploymentPipelineRef":"foo-pipeline"}}'
```

**Use PUT, not PATCH** — PATCH returns an empty response on this API.

### Other useful REST endpoints

```bash
# List any resource
curl -s -H "Authorization: Bearer $MCP_TOKEN" \
  "http://api.openchoreo.localhost:8080/api/v1/namespaces/default/{resource}"

# Get a specific resource
curl -s -H "Authorization: Bearer $MCP_TOKEN" \
  "http://api.openchoreo.localhost:8080/api/v1/namespaces/default/{resource}/{name}"
```

## PE-Relevant occ Commands

PEs use both `occ` and `kubectl`. Use `occ` for OpenChoreo abstractions, `kubectl` for cluster-level operations and debugging.

### Quick reference

| Command | Alias | What it does |
|---------|-------|-------------|
| `occ dataplane list/get` | `dp` | Inspect DataPlane CRs |
| `occ buildplane list/get` | `bp` | Inspect BuildPlane CRs |
| `occ observabilityplane list/get` | `op` | Inspect ObservabilityPlane CRs |
| `occ environment list/get` | `env` | Inspect Environments |
| `occ deploymentpipeline list/get` | `deppipe` | Inspect promotion paths |
| `occ componenttype list/get` | `ct` | Inspect namespace-scoped types |
| `occ clustercomponenttype list/get` | `cct` | Inspect cluster-scoped types |
| `occ trait list/get` | `traits` | Inspect namespace-scoped traits |
| `occ clustertrait list/get` | `clustertraits` | Inspect cluster-scoped traits |
| `occ workflow list/get` | `wf` | Inspect workflow templates |
| `occ authzclusterrole list/get` | `cr` | Inspect cluster roles |
| `occ authzclusterrolebinding list/get` | `crb` | Inspect cluster role bindings |
| `occ authzrole list/get` | - | Inspect namespace roles |
| `occ authzrolebinding list/get` | `rb` | Inspect role bindings |
| `occ apply -f <file>` | - | Create/update any resource |
| `occ namespace list/get` | `ns` | Inspect namespaces |

### Key behaviors

- `occ <resource> get <name>` returns full YAML (spec + status). No `-o` flag.
- Scope flags are not uniform across `occ` subcommands. Many `list` commands accept `--project`, while many `get` commands use `--namespace` only. Check `--help` on the exact subcommand when scope matters.
- `occ apply -f` works for all resource types including platform resources.
- Service account login: `occ login --client-credentials --client-id <id> --client-secret <secret>`

## Resource Schemas

### DataPlane

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: DataPlane                     # also ClusterDataPlane
metadata:
  name: default
  namespace: default
spec:
  planeID: default                  # must match agent Helm value
  clusterAgent:
    clientCA:
      value: |                      # agent's CA certificate (PEM)
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
  gateway:
    publicVirtualHost: "apps.example.com"
    organizationVirtualHost: "internal.example.com"
    publicHTTPPort: 80
    publicHTTPSPort: 443
    ingress:
      external:
        name: gateway-default
        namespace: openchoreo-data-plane
        http:
          host: openchoreoapis.localhost
          port: 19080
          listenerName: http
  secretStoreRef:
    name: default
  imagePullSecretRefs:
    - name: registry-credentials
  observabilityPlaneRef:
    name: default
```

### BuildPlane

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: BuildPlane                    # also ClusterBuildPlane
metadata:
  name: default
  namespace: default
spec:
  planeID: default
  clusterAgent:
    clientCA:
      value: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
  secretStoreRef:
    name: default
```

### ObservabilityPlane

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: ObservabilityPlane            # also ClusterObservabilityPlane
metadata:
  name: default
  namespace: default
spec:
  planeID: default
  clusterAgent:
    clientCA:
      value: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
  observerURL: "https://observer.example.com"
```

### Environment

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: Environment
metadata:
  name: development
  namespace: default
  labels:
    openchoreo.dev/name: development
spec:
  dataPlaneRef:
    kind: DataPlane
    name: default
  isProduction: false
```

### DeploymentPipeline

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: DeploymentPipeline
metadata:
  name: default
  namespace: default
spec:
  promotionPaths:
    - sourceEnvironmentRef: development
      targetEnvironmentRefs:
        - name: staging
          requiresApproval: false
    - sourceEnvironmentRef: staging
      targetEnvironmentRefs:
        - name: production
          requiresApproval: true
```

### Project

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: Project
metadata:
  name: default
  namespace: default
spec:
  deploymentPipelineRef: default     # plain string, not an object
```

### ObservabilityAlertsNotificationChannel

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: ObservabilityAlertsNotificationChannel
metadata:
  name: email-alerts
  namespace: default
spec:
  environment: development
  isEnvDefault: true
  type: email                        # email | webhook
  emailConfig:
    from: alerts@example.com
    to: ["team@example.com"]
    smtp:
      host: smtp.example.com
      port: 587
      auth:
        username:
          secretKeyRef: {name: smtp-creds, key: username}
        password:
          secretKeyRef: {name: smtp-creds, key: password}
```

For ComponentType, Trait, and Workflow schemas, see `templates-and-workflows.md`.
