# OpenChoreo MCP Reference — Developer

This document maps developer workflows to the `mcp__openchoreo-cp__*` (control plane) and `mcp__openchoreo-obs__*` (observability) MCP tools available via Claude Code. Use these instead of the `occ` CLI when working through an AI assistant.

## Tool Quick Reference

| MCP Tool | CLI Equivalent | Purpose |
|----------|---------------|---------|
| `list_namespaces` | `occ namespace list` | List all namespaces |
| `create_namespace` | `occ apply -f namespace.yaml` | Create a namespace |
| `list_projects` | `occ project list` | List projects in a namespace |
| `create_project` | `occ apply -f project.yaml` | Create a new project |
| `list_components` | `occ component list` | List components in a project |
| `get_component` | `occ component get <name>` | Get component spec and status |
| `create_component` | `occ apply -f component.yaml` | Create a new component |
| `patch_component` | `occ apply -f component.yaml` (update) | Patch component fields |
| `get_component_schema` | `occ component scaffold` (inspect) | Get component YAML schema |
| `list_component_types` | `occ componenttype list` | List namespace-scoped component types |
| `get_component_type_schema` | `occ componenttype get <name>` | Get component type schema |
| `list_cluster_component_types` | `occ clustercomponenttype list` | List cluster-scoped component types |
| `get_cluster_component_type` | `occ clustercomponenttype get <name>` | Get cluster component type |
| `get_cluster_component_type_schema` | `occ clustercomponenttype get <name>` | Get cluster component type schema |
| `list_traits` | `occ trait list` | List namespace-scoped traits |
| `get_trait_schema` | `occ trait get <name>` | Get trait schema |
| `list_cluster_traits` | `occ clustertrait list` | List cluster-scoped traits |
| `get_cluster_trait` | `occ clustertrait get <name>` | Get cluster trait |
| `get_cluster_trait_schema` | `occ clustertrait get <name>` | Get cluster trait schema |
| `list_workflows` | `occ workflow list` | List available build workflow templates |
| `get_workflow_schema` | `occ workflow get <name>` | Get workflow template schema |
| `create_workflow_run` | `occ component workflow run` | Create (queue) a workflow run |
| `trigger_workflow_run` | `occ component workflow run <name>` | Trigger a build/workflow run |
| `get_workflow_run` | `occ component workflowrun get <name>` | Get a specific workflow run |
| `list_workflow_runs` | `occ component workflowrun list` | List workflow run history |
| `list_component_releases` | `occ componentrelease list` | List component releases |
| `get_component_release` | `occ componentrelease get <name>` | Get release details |
| `get_component_release_schema` | — | Get component release schema |
| `create_component_release` | `occ componentrelease generate` | Create a component release |
| `create_workload` | `occ workload create` | Create a workload for a component |
| `update_workload` | `occ apply -f workload.yaml` (update) | Update an existing workload |
| `get_workload` | `occ workload get <name>` | Get workload details |
| `get_workload_schema` | — | Get workload YAML schema |
| `list_workloads` | `occ workload list --component <name>` | List workloads for a component |
| `list_environments` | `occ environment list` | List environments |
| `get_environment` | `occ environment get <name>` | Get environment details |
| `get_environment_release` | — | Get release deployed in an environment |
| `list_deployment_pipelines` | `occ deploymentpipeline list` | List deployment pipelines |
| `get_deployment_pipeline` | `occ deploymentpipeline get <name>` | Get deployment pipeline details |
| `deploy_release` | `occ component deploy <name>` | Deploy a release to root environment |
| `promote_component` | `occ component deploy --to <env>` | Promote component to next environment |
| `list_release_bindings` | `occ releasebinding list` | List release bindings |
| `get_release_binding` | `occ releasebinding get <name>` | Get a specific release binding |
| `patch_release_binding` | `occ apply -f releasebinding.yaml` (update) | Patch a release binding |
| `update_release_binding_state` | — | Activate or deactivate a release binding |
| `get_observer_url` | — | Get observability URL for a component |
| `list_secret_references` | `occ secretreference list` | List secret references |

## Observability Tool Quick Reference

These tools use the `mcp__openchoreo-obs__*` server and are the primary way to query logs, metrics, and traces from an AI assistant.

| MCP Tool | Purpose |
|----------|---------|
| `query_component_logs` | Query runtime logs for a component in a given environment |
| `query_workflow_logs` | Query build / workflow run logs |
| `query_http_metrics` | Query HTTP request rate, latency, and error metrics |
| `query_resource_metrics` | Query CPU and memory usage for a component |
| `query_traces` | Search distributed traces for a component |
| `query_trace_spans` | List spans within a trace |
| `get_span_details` | Get full detail for a single span |

## Key Workflows

### 1. Explore a New Cluster

```
list_namespaces              → find available namespaces
list_projects(namespace)     → find projects
list_environments(namespace) → find environments and their states
list_component_types(namespace)         → what types can I deploy?
list_cluster_component_types(namespace) → cluster-wide types?
list_cluster_traits(namespace)          → what traits are available?
list_workflows(namespace)               → what build workflows exist?
```

### 2. Scaffold and Create a Component

Before creating a component YAML, discover what's available:

```
list_cluster_component_types       → find available workload types (e.g. deployment/service)
get_cluster_component_type_schema  → inspect fields and options for a type
list_cluster_traits                → find available traits (e.g. ingress, storage)
get_cluster_trait_schema           → inspect trait fields
get_component_schema               → get the full Component resource schema
```

Then create:

```
create_component(namespace, project, name, spec_yaml)
```

### 3. Build (Trigger Workflow)

```
list_workflows(namespace)                               → what workflows are available?
list_workflow_runs(namespace, project, component)       → check existing run history
trigger_workflow_run(namespace, project, component)     → kick off a build
get_workflow_run(namespace, project, component, run_id) → check run status
```

### 4. Deploy and Promote

```
list_component_releases(namespace, project, component) → find available releases
deploy_release(namespace, project, component, release)  → deploy to root environment
get_environment_release(namespace, environment)         → confirm what's deployed
promote_component(namespace, project, component, target_env) → promote to next env
```

### 5. Inspect and Debug

```
get_component(namespace, project, component)            → spec + status conditions
list_workloads(namespace, project, component)           → running workloads
get_workload(namespace, project, component, workload)   → workload details
list_release_bindings(namespace, project, component)    → binding per environment
get_release_binding(namespace, project, component, env) → binding for a specific env
get_environment_release(namespace, environment)         → what release is live
get_observer_url(namespace, project, component)         → observability UI link
list_secret_references(namespace)                       → available secrets
```

### 6. Query Logs and Metrics (Observability MCP)

Use `mcp__openchoreo-obs__*` tools to query telemetry without leaving the AI assistant:

```
query_component_logs(namespace, project, component, environment)   → runtime logs
query_workflow_logs(namespace, project, component, workflow_run)   → build logs
query_http_metrics(namespace, project, component, environment)     → request rate / latency / errors
query_resource_metrics(namespace, project, component, environment) → CPU and memory
```

Trace debugging flow:

```
query_traces(namespace, project, component, environment)  → find relevant traces
query_trace_spans(trace_id)                               → list spans in a trace
get_span_details(span_id)                                 → inspect a single span
```

### 7. Create or Update a Workload

```
get_workload_schema(namespace, project, component)         → understand workload fields first
create_workload(namespace, project, component, name, spec) → create workload
update_workload(namespace, project, component, name, spec) → update existing workload
list_workloads(namespace, project, component)              → verify it appears
```

### 8. Manage Release Binding State

```
list_release_bindings(namespace, project, component)          → list all bindings
get_release_binding(namespace, project, component, env)       → inspect a binding
patch_release_binding(namespace, project, component, env, patch) → patch binding fields
update_release_binding_state(namespace, project, component, env, active) → activate/deactivate
```

## Exploration Workflow (Start Here)

When working with an unfamiliar cluster, always explore in this order:

```
1. list_namespaces
2. list_projects(namespace)
3. list_environments(namespace)
4. list_deployment_pipelines(namespace)
5. list_cluster_component_types(namespace)
6. list_cluster_traits(namespace)
7. list_workflows(namespace)
8. list_components(namespace, project)
```

## Common Gotchas

**Component type format is `workloadType/typeName`**: Use `get_cluster_component_type_schema` to see accepted values before setting `spec.componentTypeRef`.

**`get_component` includes status**: The response includes `status.conditions` with `type`, `status`, `reason`, and `message`. Always check conditions when debugging.

**`list_release_bindings` requires a component**: You must pass both project and component name, not just project.

**`deploy_release` vs `promote_component`**: Use `deploy_release` for initial deployment to the root environment. Use `promote_component` to advance to higher environments.

**`get_workload_schema` before writing workload YAML**: Call `get_workload_schema` to explore available fields rather than guessing. The `connections` field is an array of objects, not a map.

**`trigger_workflow_run` vs `create_workflow_run`**: `trigger_workflow_run` starts an immediate run. `create_workflow_run` queues a run with explicit configuration. Check the schema for each to confirm parameters.

**Workflow runs can lag**: A just-triggered workflow may briefly show no runs. Call `list_workflow_runs` after a moment, then verify with `get_component`.

**Two separate MCP servers**: `mcp__openchoreo-cp__*` talks to the control plane API; `mcp__openchoreo-obs__*` talks to the Observer API. Both must be configured. See the [MCP configuration guide](https://openchoreo.dev/docs/reference/mcp-servers/mcp-ai-configuration/).

**`connections` must be an array, not a map**: `get_workload_schema` may report connections as a JSON object. The API requires an array of connection objects: `[{"component": "...", "endpoint": "...", "visibility": "project", "envBindings": {"address": "ENV_VAR"}}]`.

**Use `query_component_logs` before `get_observer_url`**: For quick log triage in the AI assistant, `query_component_logs` is faster than opening the UI link. Use the URL when you need the full dashboard.

**Observability data requires a live ObservabilityPlane**: If queries return no data, confirm the ObservabilityPlane is registered and healthy — escalate to `openchoreo-platform-engineer` if needed.
