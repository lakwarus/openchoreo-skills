# OpenChoreo MCP Reference — Platform Engineer

This document maps platform engineer workflows to the `mcp__openchoreo-cp__*` MCP tools available via Claude Code. Use these instead of the `occ` CLI when working through an AI assistant.

## Tool Quick Reference

| MCP Tool | CLI Equivalent | Purpose |
|----------|---------------|---------|
| `list_namespaces` | `occ namespace list` | List namespaces |
| `get_namespace` | `occ namespace get <name>` | Get namespace details |
| `create_namespace` | `occ apply -f namespace.yaml` | Create a namespace |
| `list_dataplanes` | `occ dataplane list` | List namespace-scoped DataPlanes |
| `get_dataplane` | `occ dataplane get <name>` | Get DataPlane details |
| `create_dataplane` | `occ apply -f dataplane.yaml` | Create a DataPlane |
| `list_cluster_dataplanes` | `occ dataplane list` (cluster scope) | List cluster-scoped DataPlanes |
| `get_cluster_dataplane` | `occ dataplane get <name>` (cluster scope) | Get ClusterDataPlane details |
| `create_cluster_dataplane` | `occ apply -f clusterdataplane.yaml` | Create a ClusterDataPlane |
| `list_cluster_buildplanes` | `occ buildplane list` | List BuildPlanes |
| `list_observability_planes` | `occ observabilityplane list` | List namespace-scoped ObservabilityPlanes |
| `list_cluster_observability_planes` | `occ observabilityplane list` (cluster) | List cluster-scoped ObservabilityPlanes |
| `list_environments` | `occ environment list` | List environments |
| `get_environment` | `occ environment get <name>` | Get environment details |
| `create_environment` | `occ apply -f environment.yaml` | Create an environment |
| `get_deployment_pipeline` | `occ deploymentpipeline get <name>` | Get deployment pipeline |
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
| `get_project` | `occ project get <name>` | Get project details |
| `create_project` | `occ apply -f project.yaml` | Create a project |
| `list_components` | `occ component list` | List components (for inspection) |
| `get_component` | `occ component get <name>` | Get component spec and status |
| `list_secret_references` | `occ secretreference list` | List secret references |
| `apply_resource` | `occ apply -f <file>` | Create/update any resource from YAML |
| `get_resource` | `occ <resource> get <name>` | Get any resource YAML |
| `delete_resource` | `occ <resource> delete <name>` | Delete any resource |
| `explain_schema` | — | Explain schema for any resource kind |
| `get_environment_release` | — | Check what release is deployed in an env |
| `list_release_bindings` | `occ releasebinding list` | List component release bindings |

## Resource Schemas

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

Set up namespace, planes, environments, and pipelines in order:

```
create_namespace(name)
create_cluster_dataplane(name, spec_yaml)    → or create_dataplane for ns-scoped
create_environment(namespace, name, dataplane_ref, is_production)
apply_resource(deployment_pipeline_yaml)     → define promotion paths
create_project(namespace, name, pipeline_ref)
```

### 2. Inspect Platform Infrastructure

```
list_namespaces                              → what namespaces exist?
list_cluster_dataplanes(namespace)           → what dataplanes are registered?
get_cluster_dataplane(namespace, name)       → inspect spec + agent status
list_cluster_buildplanes(namespace)          → what build capacity exists?
list_cluster_observability_planes(namespace) → what observability is configured?
list_environments(namespace)                 → what environments are available?
get_deployment_pipeline(namespace, name)     → inspect promotion paths
```

### 3. Register Component Types

ComponentTypes and ClusterComponentTypes define the allowed workload shapes developers can deploy. Inspect before creating:

```
list_cluster_component_types(namespace)         → what types exist?
get_cluster_component_type(namespace, name)     → inspect one type
get_cluster_component_type_schema(namespace, name) → see full schema with templates
explain_schema(kind="ClusterComponentType", path="spec") → explore fields
apply_resource(component_type_yaml)             → register a new type
```

### 4. Register Traits

Traits are capabilities (ingress, storage, etc.) that can be attached to components:

```
list_cluster_traits(namespace)              → what traits exist?
get_cluster_trait(namespace, name)          → inspect one trait
get_cluster_trait_schema(namespace, name)   → see full schema with patches
explain_schema(kind="ClusterTrait", path="spec") → explore fields
apply_resource(trait_yaml)                  → register a new trait
```

### 5. Register Workflow Templates

Build workflow templates define how components are built:

```
list_workflows(namespace)                   → what workflows exist?
get_workflow_schema(namespace, name)        → inspect a workflow template
apply_resource(workflow_yaml)               → register a new workflow template
```

### 6. Inspect Tenant Usage

```
list_projects(namespace)                    → what projects exist?
list_components(namespace, project)         → what components are deployed?
get_component(namespace, project, component) → inspect spec and status conditions
list_environments(namespace)                → what environments are active?
get_environment_release(namespace, env)     → what release is live per env?
list_release_bindings(namespace, project, component) → binding per environment
list_secret_references(namespace)           → available secrets
```

### 7. Generic Resource Operations

```
apply_resource(yaml_content)               → create or update any resource
get_resource(kind, name, namespace)        → fetch any resource YAML
delete_resource(kind, name, namespace)     → delete any resource
explain_schema(kind, path)                 → understand any resource schema
```

## Exploration Workflow (Start Here)

When connecting to a new cluster, explore infrastructure state in this order:

```
1. list_namespaces
2. list_cluster_dataplanes(namespace)
3. list_cluster_buildplanes(namespace)
4. list_cluster_observability_planes(namespace)
5. list_environments(namespace)
6. get_deployment_pipeline(namespace, "default")
7. list_cluster_component_types(namespace)
8. list_cluster_traits(namespace)
9. list_workflows(namespace)
```

## Common Gotchas

**`deploymentPipelineRef` is a plain string**: In Project YAML, use `deploymentPipelineRef: default`, not an object with `kind`/`name`.

**`apply_resource` is universal**: You can create any resource (DataPlane, Environment, DeploymentPipeline, ComponentType, Trait, Workflow, etc.) with `apply_resource(yaml)`.

**`explain_schema` before authoring YAML**: Always call `explain_schema(kind="DataPlane", path="spec")` (or relevant kind) before hand-authoring resource YAML to discover required fields.

**Namespace-scoped vs cluster-scoped**: `DataPlane`/`BuildPlane`/`ObservabilityPlane` come in both namespace-scoped and cluster-scoped (`ClusterDataPlane`, etc.) variants. Use cluster-scoped for shared infrastructure; namespace-scoped for tenant isolation.

**Status conditions are your debug tool**: `get_dataplane`, `get_environment`, `get_component` all return `status.conditions`. Check `reason` and `message` fields for any resource that isn't `Ready`.

**`get_resource` is a fallback**: For resource kinds not covered by a dedicated MCP tool, use `get_resource(kind, name, namespace)` to retrieve full YAML.
