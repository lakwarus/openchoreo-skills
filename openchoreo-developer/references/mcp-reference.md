# OpenChoreo MCP Reference — Developer

This document maps developer workflows to the `mcp__openchoreo-cp__*` MCP tools available via Claude Code. Use these instead of the `occ` CLI when working through an AI assistant.

## Tool Quick Reference

| MCP Tool | CLI Equivalent | Purpose |
|----------|---------------|---------|
| `list_namespaces` | `occ namespace list` | List all namespaces |
| `get_namespace` | `occ namespace get <name>` | Get namespace details |
| `list_projects` | `occ project list` | List projects in a namespace |
| `get_project` | `occ project get <name>` | Get project details |
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
| `update_component_traits` | `occ apply -f component.yaml` (traits section) | Update component traits |
| `list_component_traits` | `occ trait list --component <name>` | List traits attached to a component |
| `list_workflows` | `occ workflow list` | List available build workflow templates |
| `get_workflow_schema` | `occ workflow get <name>` | Get workflow template schema |
| `list_component_workflows` | `occ component workflow list` | List workflows for a component |
| `list_component_workflows_org_level` | `occ workflow list` (org scope) | List workflows at org level |
| `get_component_workflow_schema` | `occ component workflow get` | Get component workflow schema |
| `get_component_workflow_schema_org_level` | — | Get component workflow schema at org level |
| `update_component_workflow_schema` | — | Update component workflow configuration |
| `trigger_component_workflow` | `occ component workflow run <name>` | Trigger a build/workflow |
| `list_component_workflow_runs` | `occ component workflowrun list <name>` | List workflow run history |
| `list_component_releases` | `occ componentrelease list` | List component releases |
| `get_component_release` | `occ componentrelease get <name>` | Get release details |
| `get_component_release_schema` | — | Get component release schema |
| `create_component_release` | `occ componentrelease generate` | Create a component release |
| `create_workload` | `occ workload create` | Create a workload for a component |
| `get_component_workloads` | `occ workload list --component <name>` | List workloads for a component |
| `list_environments` | `occ environment list` | List environments |
| `get_environment` | `occ environment get <name>` | Get environment details |
| `get_environment_release` | — | Get release deployed in an environment |
| `deploy_release` | `occ component deploy <name>` | Deploy a release to root environment |
| `promote_component` | `occ component deploy --to <env>` | Promote component to next environment |
| `list_release_bindings` | `occ releasebinding list` | List release bindings |
| `patch_release_binding` | `occ apply -f releasebinding.yaml` (update) | Patch a release binding |
| `update_component_binding` | — | Update component environment binding |
| `get_component_observer_url` | — | Get observability URL for a component |
| `list_secret_references` | `occ secretreference list` | List secret references |
| `apply_resource` | `occ apply -f <file>` | Create/update any resource from YAML |
| `get_resource` | `occ <resource> get <name>` | Get any resource by kind and name |
| `delete_resource` | `occ <resource> delete <name>` | Delete any resource |
| `explain_schema` | — | Explain schema for any resource kind |

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
explain_schema(kind="Component", path="spec") → drill into spec fields
```

Then create:

```
create_component(namespace, project, name, spec_yaml)
```

### 3. Build (Trigger Workflow)

```
list_component_workflows(namespace, project, component)   → what workflows are configured?
trigger_component_workflow(namespace, project, component) → kick off a build
list_component_workflow_runs(namespace, project, component) → monitor run history
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
get_component(namespace, project, component)   → spec + status conditions
get_component_workloads(namespace, project, component) → running workloads
list_release_bindings(namespace, project, component)   → binding per environment
get_environment_release(namespace, environment)        → what release is live
get_component_observer_url(namespace, project, component) → observability link
list_secret_references(namespace)                      → available secrets
```

### 6. Create a Workload Directly

```
create_workload(namespace, project, component, name, image_or_descriptor)
get_component_workloads(namespace, project, component)  → verify it appears
```

### 7. Generic Resource Operations

```
apply_resource(yaml_content)             → create or update any resource
get_resource(kind, name, namespace)      → fetch any resource YAML
delete_resource(kind, name, namespace)   → delete any resource
explain_schema(kind, path)               → understand any resource schema
```

## Exploration Workflow (Start Here)

When working with an unfamiliar cluster, always explore in this order:

```
1. list_namespaces
2. list_projects(namespace)
3. list_environments(namespace)
4. list_cluster_component_types(namespace)
5. list_cluster_traits(namespace)
6. list_workflows(namespace)
7. list_components(namespace, project)
```

## Common Gotchas

**Component type format is `workloadType/typeName`**: Use `get_cluster_component_type_schema` to see accepted values before setting `spec.componentTypeRef`.

**`get_component` includes status**: The response includes `status.conditions` with `type`, `status`, `reason`, and `message`. Always check conditions when debugging.

**`list_release_bindings` requires a component**: You must pass both project and component name, not just project.

**`deploy_release` vs `promote_component`**: Use `deploy_release` for initial deployment to the root environment. Use `promote_component` to advance to higher environments.

**`explain_schema` is your scaffold tool**: When you need to write resource YAML, call `explain_schema(kind="Component", path="spec")` to explore available fields rather than guessing.

**Workflow runs can lag**: A just-triggered workflow may briefly show no runs. Call `list_component_workflow_runs` after a moment, then verify with `get_component`.
