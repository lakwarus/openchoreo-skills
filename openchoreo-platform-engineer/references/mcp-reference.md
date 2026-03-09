# OpenChoreo MCP Reference — Platform Engineer

This document maps platform engineer workflows to the `mcp__openchoreo-cp__*` (control plane) and `mcp__openchoreo-obs__*` (observability) MCP tools available via Claude Code. Use these instead of the `occ` CLI when working through an AI assistant.

> **Note on platform resource management**: The following operations are **not available as MCP tools** — use `occ apply -f` instead (see `cli-and-resources.md` → Creating Platform Resources with occ):
> - `create_environment` / `create_deployment_pipeline`
> - DataPlane, BuildPlane, ObservabilityPlane CRUD
>
> Projects created via `create_project` MCP default to `deploymentPipelineRef: default`. Use `occ apply -f` to create the project with the correct pipeline, or reapply after creation.

## Tool Quick Reference

| MCP Tool | CLI Equivalent | Purpose |
|----------|---------------|---------|
| `list_namespaces` | `occ namespace list` | List namespaces |
| `create_namespace` | `occ apply -f namespace.yaml` | Create a namespace |
| `list_environments` | `occ environment list` | List environments |
| `get_environment` | `occ environment get <name>` | Get environment details |
| `get_environment_release` | — | Check what release is deployed in an env |
| `list_deployment_pipelines` | `occ deploymentpipeline list` | List deployment pipelines |
| `get_deployment_pipeline` | `occ deploymentpipeline get <name>` | Get deployment pipeline details |
| `list_component_types` | `occ componenttype list` | List namespace-scoped component types |
| `get_component_type_schema` | `occ componenttype get <name>` | Get component type schema |
| `list_cluster_component_types` | `occ clustercomponenttype list` | List cluster-scoped component types |
| `get_cluster_component_type` | `occ clustercomponenttype get <name>` | Get cluster component type |
| `get_cluster_component_type_schema` | `occ clustercomponenttype get <name>` | Get cluster component type full schema |
| `list_traits` | `occ trait list` | List namespace-scoped traits |
| `get_trait_schema` | `occ trait get <name>` | Get trait schema |
| `list_cluster_traits` | `occ clustertrait list` | List cluster-scoped traits |
| `get_cluster_trait` | `occ clustertrait get <name>` | Get cluster trait |
| `get_cluster_trait_schema` | `occ clustertrait get <name>` | Get cluster trait full schema |
| `list_workflows` | `occ workflow list` | List build workflow templates |
| `get_workflow_schema` | `occ workflow get <name>` | Get workflow template schema |
| `list_projects` | `occ project list` | List projects |
| `create_project` | `occ apply -f project.yaml` | Create a project |
| `list_components` | `occ component list` | List components (for inspection) |
| `get_component` | `occ component get <name>` | Get component spec and status |
| `patch_component` | `occ apply -f component.yaml` (update) | Patch a component |
| `list_workloads` | `occ workload list` | List workloads for a component |
| `get_workload` | `occ workload get <name>` | Get workload details |
| `get_workload_schema` | — | Get workload YAML schema |
| `list_workflow_runs` | `occ component workflowrun list` | List workflow runs |
| `get_workflow_run` | `occ component workflowrun get <name>` | Get a specific workflow run |
| `list_component_releases` | `occ componentrelease list` | List component releases |
| `get_component_release` | `occ componentrelease get <name>` | Get release details |
| `list_release_bindings` | `occ releasebinding list` | List component release bindings |
| `get_release_binding` | `occ releasebinding get <name>` | Get a specific release binding |
| `patch_release_binding` | `occ apply -f releasebinding.yaml` (update) | Patch a release binding |
| `update_release_binding_state` | — | Activate or deactivate a release binding |
| `list_secret_references` | `occ secretreference list` | List secret references |
| `get_observer_url` | — | Get observability URL for a component |

## Observability Tool Quick Reference

These tools use the `mcp__openchoreo-obs__*` server. Platform engineers use them to validate that the observability stack is collecting data correctly and to help investigate component-level issues raised by developers.

| MCP Tool | Purpose |
|----------|---------|
| `query_component_logs` | Query runtime logs for a component in a given environment |
| `query_workflow_logs` | Query build / workflow run logs |
| `query_http_metrics` | Query HTTP request rate, latency, and error metrics |
| `query_resource_metrics` | Query CPU and memory usage for a component |
| `query_traces` | Search distributed traces for a component |
| `query_trace_spans` | List spans within a trace |
| `get_span_details` | Get full detail for a single span |

## Resource Schemas

These YAML shapes are used with `kubectl apply` or `occ apply` — there are no dedicated MCP CRUD tools for plane resources.

### Namespace

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: Namespace
metadata:
  name: my-org
```

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
      value: |
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

## Key Workflows

### 1. Initial Platform Setup

Environments and DeploymentPipelines have no MCP create tools — use `occ apply -f` (see `cli-and-resources.md` → Creating Platform Resources with occ). Plane resources (DataPlane, BuildPlane, ObservabilityPlane) also use `occ apply -f`.

```bash
# Step 1 — login to occ (local setup)
occ config controlplane update default --url http://api.openchoreo.localhost:8080
occ login    # browser-based PKCE auth

# Step 2 — create environments via occ apply
occ apply -f environment-development.yaml
occ apply -f environment-production.yaml

# Step 3 — create deployment pipeline via occ apply
occ apply -f my-pipeline.yaml
```

See `cli-and-resources.md` → Creating Platform Resources with occ for YAML examples.

Then use MCP or occ to create a project with the correct pipeline:

```bash
# Via occ (sets pipeline at creation time)
occ apply -f project.yaml   # spec.deploymentPipelineRef: my-pipeline
```

Or via MCP then fix the pipeline ref with occ:

```
create_project(namespace, name)   → defaults to "default" pipeline
```
```bash
occ apply -f project.yaml         # reapply with correct deploymentPipelineRef
```

Verify with MCP:

```
list_environments(namespace)            → confirm environments visible
list_deployment_pipelines(namespace)    → confirm pipeline visible
list_projects(namespace)                → confirm pipeline assignments
```

### 2. Inspect Platform Infrastructure

```
list_namespaces                              → what namespaces exist?
list_environments(namespace)                 → what environments are available?
list_deployment_pipelines(namespace)         → what promotion paths exist?
get_deployment_pipeline(namespace, name)     → inspect pipeline spec
list_cluster_component_types(namespace)      → what component types are registered?
list_cluster_traits(namespace)               → what traits are registered?
list_workflows(namespace)                    → what build workflows exist?
```

For plane status (DataPlane, BuildPlane, ObservabilityPlane), use the REST API since there are no MCP tools for these:

```bash
# Get all dataplanes
curl -s -H "Authorization: Bearer $MCP_TOKEN" \
  "http://api.openchoreo.localhost:8080/api/v1/namespaces/default/dataplanes"

# If the REST API doesn't expose planes, kubectl is the fallback
kubectl get clusterdataplane,clusterbuildplane,clusterobservabilityplane -A
```

### 3. Register Component Types

ComponentTypes and ClusterComponentTypes define the allowed workload shapes developers can deploy. Inspect before creating:

```
list_cluster_component_types(namespace)            → what types exist?
get_cluster_component_type(namespace, name)        → inspect one type
get_cluster_component_type_schema(namespace, name) → see full schema with templates
```

Register new types via `occ apply` (preferred) or the REST API:

### 4. Register Traits

Traits are capabilities (ingress, storage, etc.) that can be attached to components:

```
list_cluster_traits(namespace)              → what traits exist?
get_cluster_trait(namespace, name)          → inspect one trait
get_cluster_trait_schema(namespace, name)   → see full schema with patches
```

### 5. Register Workflow Templates

Build workflow templates define how components are built:

```
list_workflows(namespace)              → what workflows exist?
get_workflow_schema(namespace, name)   → inspect a workflow template
```

Register new workflows via `occ apply` (preferred) or the REST API.

### 6. Inspect Tenant Usage

```
list_projects(namespace)                              → what projects exist?
list_components(namespace, project)                   → what components are deployed?
get_component(namespace, project, component)          → inspect spec and status conditions
list_environments(namespace)                          → what environments are active?
get_environment_release(namespace, env)               → what release is live per env?
list_release_bindings(namespace, project, component)  → binding per environment
get_release_binding(namespace, project, component, env) → binding for a specific env
list_workloads(namespace, project, component)         → running workloads
get_workload(namespace, project, component, workload) → workload spec and status
list_secret_references(namespace)                     → available secrets
```

### 7. Validate Observability Stack (Observability MCP)

After registering an ObservabilityPlane, confirm data is flowing end-to-end:

```
query_resource_metrics(namespace, project, component, environment) → CPU/memory arriving?
query_component_logs(namespace, project, component, environment)   → logs arriving?
query_http_metrics(namespace, project, component, environment)     → HTTP metrics arriving?
query_traces(namespace, project, component, environment)           → traces arriving?
```

If any query returns no data, check the ObservabilityPlane agent connectivity and Helm configuration for that signal type.

## Exploration Workflow (Start Here)

When connecting to a new cluster, explore infrastructure state in this order:

```
1. list_namespaces
2. list_environments(namespace)
3. list_deployment_pipelines(namespace)
4. list_cluster_component_types(namespace)
5. list_cluster_traits(namespace)
6. list_workflows(namespace)
7. list_projects(namespace)
```

For plane resources not exposed by MCP, try the REST API first:

```bash
MCP_TOKEN=$(curl -s -X POST "http://thunder.openchoreo.localhost:8080/oauth2/token" \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -u 'service_mcp_client:service_mcp_client_secret' \
  -d 'grant_type=client_credentials' | jq -r '.access_token')

curl -s -H "Authorization: Bearer $MCP_TOKEN" \
  "http://api.openchoreo.localhost:8080/api/v1/namespaces/default/dataplanes"
```

Fall back to `kubectl` only if the REST API does not expose the resource.

## Common Gotchas

**`create_environment` and `create_deployment_pipeline` are not MCP tools**: Use `occ apply -f` instead. See `cli-and-resources.md` → Creating Platform Resources with occ for YAML examples and auth setup.

**DataPlane/BuildPlane/ObservabilityPlane CRUD is not exposed via MCP or REST API**: Use `occ apply -f <file>` if occ is available, otherwise `kubectl apply`. Helm values control plane registration at install time.

**`deploymentPipelineRef` is a plain string**: In Project YAML, use `deploymentPipelineRef: default`, not an object with `kind`/`name`.

**Namespace-scoped vs cluster-scoped**: `DataPlane`/`BuildPlane`/`ObservabilityPlane` come in both namespace-scoped and cluster-scoped (`ClusterDataPlane`, etc.) variants. Use cluster-scoped for shared infrastructure; namespace-scoped for tenant isolation.

**Status conditions are your debug tool**: `get_environment`, `get_component`, and `get_workload` all return `status.conditions`. Check `reason` and `message` fields for any resource that isn't `Ready`.

**Inspect component type schema before authoring component YAML**: Call `get_cluster_component_type_schema` to discover required fields. Do not guess component type shapes.

**`get_cluster_component_type_schema` vs `get_component_type_schema`**: Cluster-scoped types (`ClusterComponentType`) are the most common. Namespace-scoped types (`ComponentType`) are for tenant isolation. Use the cluster-scoped tools first.

**Two separate MCP servers**: `mcp__openchoreo-cp__*` targets the control plane API; `mcp__openchoreo-obs__*` targets the Observer API. Both require separate registration. See the [MCP configuration guide](https://openchoreo.dev/docs/reference/mcp-servers/mcp-ai-configuration/).

**No observability data after plane registration**: Inspect the ObservabilityPlane with `kubectl describe`. Missing data usually means the agent CA cert is wrong or `observerURL` is unreachable.

**`connections` must be an array, not a map**: The `get_workload_schema` may report connections as a JSON object. The API requires an array of connection objects.
