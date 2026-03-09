# CLI and Resources

<!-- Quick navigation
- [occ Installation and Login](#occ-installation-and-login)
- [Creating Platform Resources with occ](#creating-platform-resources-with-occ)
- [PE-Relevant occ Commands](#pe-relevant-occ-commands)
- [Resource Schemas](#resource-schemas)
-->

## occ Installation and Login

> **Official docs**: https://openchoreo.dev/docs/user-guide/cli-installation/
>
> **Prerequisite**: `occ` must be installed **and** logged in before any `occ` commands or `occ apply` will work. Complete both steps below before attempting to manage platform resources.

### Step 1 — Install

```bash
# macOS Apple Silicon (ARM64)
curl -L https://github.com/openchoreo/openchoreo/releases/download/v0.17.0/occ_v0.17.0_darwin_arm64.tar.gz \
  | tar -xz && sudo mv occ /usr/local/bin/

# macOS Intel (AMD64)
curl -L https://github.com/openchoreo/openchoreo/releases/download/v0.17.0/occ_v0.17.0_darwin_amd64.tar.gz \
  | tar -xz && sudo mv occ /usr/local/bin/

# Linux x64
curl -L https://github.com/openchoreo/openchoreo/releases/download/v0.17.0/occ_v0.17.0_linux_amd64.tar.gz \
  | tar -xz && sudo mv occ /usr/local/bin/

# Linux ARM64
curl -L https://github.com/openchoreo/openchoreo/releases/download/v0.17.0/occ_v0.17.0_linux_arm64.tar.gz \
  | tar -xz && sudo mv occ /usr/local/bin/

occ version   # verify
```

Check latest release: https://github.com/openchoreo/openchoreo/releases/latest

> **Note**: `~/.choreo/bin/choreo` is the WSO2 commercial Choreo cloud CLI — a different product that cannot manage OpenChoreo resources.

### Step 2 — Configure and Login

```bash
# Point occ at the control plane (replace URL with your actual endpoint)
occ config controlplane update default --url http://api.openchoreo.localhost:8080

# Login (opens browser for PKCE authentication)
occ login

# Verify connection
occ namespace list
```

For local setup, ensure `/etc/hosts` has entries for `api.openchoreo.localhost`, `thunder.openchoreo.localhost`, and `observer.openchoreo.localhost` pointing to `127.0.0.1`.

> **Important**: The `service_mcp_client` used for MCP tokens does **not** work with `occ login --client-credentials`. Use browser-based login (`occ login`) or a service account specifically configured for the occ OIDC flow.

## Creating Platform Resources with occ

> **Prerequisite**: Complete occ Installation and Login above before running any command in this section. If `occ` is not installed or not logged in, these commands will fail.

Once authenticated, use `occ apply -f` to create any platform resource. This covers operations not available as MCP tools (Environment, DeploymentPipeline, DataPlane, etc.).

> **Gotcha**: `occ apply -f -` (stdin) does not work — error: `path - does not exist`. Write YAML to a temp file first, then apply.

### Create an Environment

```bash
cat > /tmp/env.yaml <<'EOF'
apiVersion: openchoreo.dev/v1alpha1
kind: Environment
metadata:
  name: qa
  namespace: default
  labels:
    openchoreo.dev/name: qa
  annotations:
    openchoreo.dev/display-name: QA
    openchoreo.dev/description: QA
spec:
  dataPlaneRef:
    kind: DataPlane
    name: default
  isProduction: false
EOF
occ apply -f /tmp/env.yaml
```

Set `isProduction: true` for production environments.

### Create a DeploymentPipeline

```bash
cat > /tmp/pipeline.yaml <<'EOF'
apiVersion: openchoreo.dev/v1alpha1
kind: DeploymentPipeline
metadata:
  name: foo-pipeline
  namespace: default
  annotations:
    openchoreo.dev/display-name: Foo Pipeline
    openchoreo.dev/description: "development → qa → production"
spec:
  promotionPaths:
    - sourceEnvironmentRef: development
      targetEnvironmentRefs:
        - name: qa
          requiresApproval: false
    - sourceEnvironmentRef: qa
      targetEnvironmentRefs:
        - name: production
          requiresApproval: false
EOF
```

### Create or Update a Project

```bash
cat <<EOF | occ apply -f -
apiVersion: openchoreo.dev/v1alpha1
kind: Project
metadata:
  name: foo
  namespace: default
spec:
  deploymentPipelineRef: foo-pipeline   # plain string, not an object
EOF
```

> **Note**: `create_project` via MCP always defaults to `deploymentPipelineRef: default`. Use `occ apply` when you need a specific pipeline assigned at creation time.

### Verify after applying

```bash
occ environment list
occ deploymentpipeline list
occ project list
```

## PE-Relevant occ Commands

Prefer `occ` for all OpenChoreo resource management. Use `kubectl` only for cluster-level operations that `occ` cannot reach (e.g. controller logs, raw CRD inspection).

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
- Service account login: `occ login --client-credentials --client-id <id> --client-secret <secret>` — only works if that client is configured for the occ OIDC flow. The `service_mcp_client` is **not** — use browser-based `occ login` instead.

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
