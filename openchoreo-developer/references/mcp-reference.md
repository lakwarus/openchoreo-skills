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
| `list_workflows` | `occ workflow list` | List namespace-scoped build workflow templates |
| `get_workflow_schema` | `occ workflow get <name>` | Get workflow template schema |
| `list_cluster_workflows` | `occ clusterworkflow list` | List cluster-scoped workflow templates |
| `get_cluster_workflow` | `occ clusterworkflow get <name>` | Get full cluster workflow spec |
| `get_cluster_workflow_schema` | `occ clusterworkflow get <name>` | Get cluster workflow parameter schema |
| `create_workflow_run` | `occ component workflow run` | Create (queue) a workflow run with explicit parameters |
| `trigger_workflow_run` | `occ component workflow run <name>` | Trigger a build/workflow run for a component |
| `get_workflow_run` | `occ component workflowrun get <name>` | Get a specific workflow run |
| `list_workflow_runs` | `occ component workflowrun list` | List workflow run history |
| `create_workload` | `occ workload create` | Create a workload for a BYO-image component |
| `update_workload` | `occ apply -f workload.yaml` (update) | Update an existing workload (source-build components) |
| `get_workload` | `occ workload get <name>` | Get workload details |
| `get_workload_schema` | — | Get workload YAML schema |
| `list_workloads` | `occ workload list --component <name>` | List workloads for a component |
| `list_environments` | `occ environment list` | List environments |
| `list_deployment_pipelines` | `occ deploymentpipeline list` | List deployment pipelines |
| `get_deployment_pipeline` | `occ deploymentpipeline get <name>` | Get deployment pipeline details |
| `list_release_bindings` | `occ releasebinding list` | List release bindings |
| `get_release_binding` | `occ releasebinding get <name>` | Get a specific release binding |
| `patch_release_binding` | `occ apply -f releasebinding.yaml` (update) | Patch a release binding (overrides, env, release) |
| `update_release_binding_state` | — | Set release binding state: `Active` or `Undeploy` |
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
list_component_types(namespace)         → namespace-scoped types
list_cluster_component_types            → cluster-wide types (most common)
list_cluster_traits                     → what traits are available?
list_workflows(namespace)               → namespace-scoped workflow templates
list_cluster_workflows                  → cluster-scoped workflow templates
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

### 4. Activate and Manage Deployments

After a build completes, release bindings are created automatically (or on first `create_workload` for BYO). Use these tools to inspect and control deployment state:

```
list_release_bindings(namespace, project, component)           → see binding per environment
get_release_binding(namespace, binding_name)                   → inspect status, endpoints, state
update_release_binding_state(namespace, binding_name, Active)  → activate a deployment
update_release_binding_state(namespace, binding_name, Undeploy)→ undeploy from an environment
patch_release_binding(namespace, binding_name, overrides)      → update env overrides or release ref
```

> **Note**: There are no MCP tools for `deploy_release` or `promote_component`. Promotion to a downstream environment is done via `occ apply -f releasebinding.yaml` with the target environment, or by patching the release binding's `environment` field.

### 5. Inspect and Debug

```
get_component(namespace, project, component)            → spec + status conditions
list_workloads(namespace, project, component)           → running workloads
get_workload(namespace, workload_name)                  → workload details
list_release_bindings(namespace, project, component)    → binding per environment
get_release_binding(namespace, binding_name)            → binding status, endpoints, invokeURL
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

### 9. Deploy a Third-Party / Multi-Service App with Pre-built Images

When the user asks you to deploy a well-known public or open-source multi-service app:

```
# 1. Find pre-built images — check release/ directory, README, CI pipelines
WebFetch or gh to fetch official kubernetes-manifests.yaml (or Helm values, docker-compose)
   → extract image URLs and ALL env vars per service

# 2. Create project
create_project(namespace, name)

# 3. Create all components — NO workflow parameter
create_component(namespace, project, name, componentType)   → repeat for each service
   componentType examples:
     deployment/service         → backend gRPC/REST/TCP services
     deployment/web-application → public-facing frontend
     deployment/worker          → background workers, load generators
     statefulset/datastore      → stateful stores (Redis, databases)

# 4. Apply workloads in batch via occ apply
# Batch 1: workloads without connections (simpler, fewer failure modes)
# Batch 2: workloads with connections
occ apply -f /tmp/<app>-workloads.yaml
occ apply -f /tmp/<app>-workloads-connections.yaml

# 5. Verify each component
list_release_bindings(namespace, component)  → Ready or ResourcesProgressing?

# 6. Investigate any failing component immediately — before assuming platform issue
query_component_logs(namespace, project, component, environment)
   → crash before "listening on port"? → vendor SDK crash, missing env var, or startup panic
   → connection refused from another service? → dependency not yet ready or wrong port/env var
```

**Key rules for this workflow:**

- Never set `workflow` in `create_component` for BYO image deployments
- Always extract ALL env vars from official manifests — dependencies inject service addresses but not `PORT`, feature flags, or vendor SDK disable flags
- If a service crash-loops before logging a "listening" message, look for a native module load error or vendor SDK init failure — apply the disable flag from the official manifests
- If a required env var references an optional or not-yet-deployed service, set a placeholder value to prevent startup panics
- `dependencies` must be a **list**, not a map — each entry needs a `name` field
- Source builds fail for repos that use `ARG BUILDPLATFORM` multi-stage syntax (exit code 125) — switch to BYO immediately when you see this error

---

## Exploration Workflow (Start Here)

When working with an unfamiliar cluster, always explore in this order:

```
1. list_namespaces
2. list_projects(namespace)
3. list_environments(namespace)
4. list_deployment_pipelines(namespace)
5. list_cluster_component_types         → cluster-wide types (most common)
6. list_component_types(namespace)      → namespace-scoped types (if any)
7. list_cluster_traits                  → cluster-wide traits
8. list_cluster_workflows               → cluster-scoped build workflows
9. list_workflows(namespace)            → namespace-scoped workflows (if any)
10. list_components(namespace, project) → components already deployed
```

## Common Gotchas

**Component type format is `workloadType/typeName`**: Use `get_cluster_component_type_schema` to see accepted values before setting `spec.componentTypeRef`.

**`get_component` includes status**: The response includes `status.conditions` with `type`, `status`, `reason`, and `message`. Always check conditions when debugging.

**`list_release_bindings` requires a component**: You must pass both project and component name, not just project.

**No `deploy_release` or `promote_component` MCP tools**: Deployment state is managed via `update_release_binding_state` (Active/Undeploy) and `patch_release_binding`. To promote to a downstream environment, use `occ apply -f releasebinding.yaml` with the target environment set.

**`get_workload_schema` before writing workload YAML**: Call `get_workload_schema` to explore available fields rather than guessing. The `dependencies` field is an array of objects, not a map.

**`trigger_workflow_run` vs `create_workflow_run`**: `trigger_workflow_run` starts a build for a component using its configured workflow and parameters (pass optional `commit` SHA to pin to a revision). `create_workflow_run` creates a standalone run by workflow name with explicit parameters — use for workflows not tied to a component.

**Workflow runs can lag**: A just-triggered workflow may briefly show no runs. Call `list_workflow_runs` after a moment, then verify with `get_component`.

**Two separate MCP servers**: `mcp__openchoreo-cp__*` talks to the control plane API; `mcp__openchoreo-obs__*` talks to the Observer API. Both must be configured. See the [MCP configuration guide](https://openchoreo.dev/docs/reference/mcp-servers/mcp-ai-configuration/).

**`dependencies` must be an array, not a map**: `get_workload_schema` may describe the field as a JSON object. The API requires an array of dependency objects: `[{"component": "...", "endpoint": "...", "visibility": "project", "envBindings": {"address": "ENV_VAR"}}]`.

**TCP `address` binding injects `host:port`, not a protocol DSN**: For databases (PostgreSQL, MySQL) and message brokers (NATS, Redis), the injected `address` value is a plain `host:port` string. Apps that expect `postgres://user:pass@host/db` or `nats://host:4222` will fail to parse it. Declare the dependency for the topology diagram but set the full DSN as a literal env var instead. Get the hostname from `get_release_binding` → `endpoints[*].serviceURL.host`.

**Source-build workloads have no endpoints or connections until you add them**: The `generate-workload-cr` build step creates a minimal workload with just the image. If the repo has no `workload.yaml`, always call `update_workload` after a successful build to add endpoints, env vars, and dependencies. Without this the component deploys but has no routing, no connections, and the cell diagram renders incomplete.

**File mount `mountPath` is a directory**: The controller appends the `key` name to `mountPath` to form the final file path. Set `mountPath` to the parent directory (`/usr/share/nginx/html`), not the full file path (`/usr/share/nginx/html/config.json`). Using the full file path doubles the filename: `.../config.json/config.json`.

**Browser-facing apps need `https://` and `wss://` backend URLs**: OpenChoreo serves web-application components over HTTPS. Any backend URLs injected at runtime (e.g., via a mounted `config.json`) must use `https://` and `wss://`. HTTP/WS URLs are blocked by browsers as mixed content — no visible error, requests just silently fail. Always get external URLs from `get_release_binding` → `endpoints[*].externalURLs` and use the `https` scheme.

**`update_workload` only needs `namespace_name`, `workload_name`, and `workload_spec`**: The `project_name` and `component_name` parameters are not accepted by this tool. Use `list_workloads` to get the workload name first.

**Use `query_component_logs` for quick triage**: `query_component_logs` is the fastest way to check runtime logs from the AI assistant. `query_workflow_logs` covers build/CI logs. Use the Backstage UI for the full dashboard.

**Observability data requires a live ObservabilityPlane**: If queries return no data, confirm the ObservabilityPlane is registered and healthy — escalate to `openchoreo-platform-engineer` if needed.
