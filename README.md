# OpenChoreo Skills

This repository contains **OpenChoreo Skills** — structured guides and reference documentation that AI agents (such as Claude Code) and automation tools use to operate on the OpenChoreo internal developer platform.

Each skill is a directory with a `SKILL.md` that tells an AI agent how to work in that role, and a `references/` directory with detailed topic files the agent reads on demand.

---

## Acknowledgement

The original OpenChoreo skills framework was created by **Isala Piyarisi** and is available here:

https://github.com/isala404/openchoreo/tree/occ-skill/docs/skills

This repository builds on that work and extends it with additional reference documentation, MCP tool guides, and platform engineer coverage.

Special thanks to Isala for creating the initial developer skills foundation.

---

## Skills in this repository

### `openchoreo-developer`

For application-level work: deploying apps, debugging Components and Workloads, writing app-facing YAML, and operating through `occ` or the MCP server.

**References included:**

| File | Contents |
|------|----------|
| `concepts.md` | Resource hierarchy, abstractions, cell architecture |
| `cli-reference.md` | `occ` install, setup, all commands, gotchas |
| `deployment-guide.md` | BYOI, source builds, `workload.yaml`, connections, promotion |
| `resource-schemas.md` | Exact YAML shapes for all developer-facing resources |
| `mcp-reference.md` | MCP tool quick reference, developer workflow → MCP tool mapping |
| `platform-engineer.md` | PE-managed capabilities and escalation wording |

---

### `openchoreo-platform-engineer`

For platform-level work: cluster setup, Helm, kubectl, planes, ComponentTypes, Traits, Workflows, gateways, secret stores, GitOps, and observability.

**References included:**

| File | Contents |
|------|----------|
| `operations.md` | Namespace provisioning, topology, upgrades |
| `templates-and-workflows.md` | ComponentType, Trait, Workflow, CEL authoring |
| `integrations.md` | Secret stores, registries, identity, RBAC, webhooks |
| `observability.md` | Logs, metrics, traces, alerts, notification channels |
| `troubleshooting.md` | Failure isolation, health checks, common patterns |
| `cli-and-resources.md` | PE-relevant `occ` commands and platform resource schemas |
| `mcp-reference.md` | MCP tool quick reference, platform setup → MCP tool mapping |
| `gitops.md` | GitOps repository layout and release flow |
| `community-modules.md` | Pluggable gateways and observability backends |
| `advanced-setup.md` | Certificates, private Git, custom builds, identity-provider swaps |
| `repo-and-context7.md` | Controller logic, CRD definitions, Helm chart details |

---

## Using with Claude Code

Claude Code supports loading external skill repositories as custom skills. When configured, Claude Code reads the `SKILL.md` files in this repository and uses them to guide its behaviour when working with OpenChoreo.

### Step 1 — Add this repository as a skill source

Add the following to your `~/.claude/settings.json` (or workspace `.claude/settings.json`):

```json
{
  "skills": [
    {
      "type": "git",
      "url": "https://github.com/lakmal-gunasekara/openchoreo-skills",
      "skills": [
        "openchoreo-developer",
        "openchoreo-platform-engineer"
      ]
    }
  ]
}
```

Alternatively, use the Claude Code CLI to add the skill:

```bash
claude skill add https://github.com/lakmal-gunasekara/openchoreo-skills
```

### Step 2 — Configure the OpenChoreo MCP server

The MCP tools referenced in both skills (`mcp__openchoreo-cp__*`) require the OpenChoreo MCP server to be running and registered. Use the `refresh-openchoreo-mcp.sh` script in this repository to configure it:

```bash
bash refresh-openchoreo-mcp.sh
```

This script authenticates with the control plane, registers the `openchoreo-cp` MCP server with a fresh bearer token, and optionally adds the `openchoreo-obs` observability server.

To register the MCP server manually, add it to your Claude Code MCP config:

```json
{
  "mcpServers": {
    "openchoreo-cp": {
      "command": "npx",
      "args": ["-y", "@openchoreo/mcp-server"],
      "env": {
        "OPENCHOREO_CP_URL": "https://<your-control-plane>",
        "OPENCHOREO_CP_TOKEN": "<bearer-token>"
      }
    }
  }
}
```

### Step 3 — Use the skills

Once configured, Claude Code will automatically activate the relevant skill based on your task:

- Tasks about deploying apps, debugging components, or writing YAML → `openchoreo-developer` activates
- Tasks about cluster setup, plane registration, ComponentTypes, or platform config → `openchoreo-platform-engineer` activates
- Tasks that cross both boundaries → both skills activate together

**Example prompts:**

```
Deploy my Node.js app in the current directory to OpenChoreo
```

```
Set up a new namespace with a dev and staging environment on OpenChoreo
```

```
Debug why my component is stuck in a pending state
```

```
Register a new ComponentType for a gRPC service
```

### Using with other AI tools

Any AI tool or agent that supports the Claude skills / MCP skill model can use this repository. Point the tool at this repository root and load the `SKILL.md` for the relevant role. The `references/` files are loaded on demand by the skill routing rules in each `SKILL.md`.

For **Cursor**, **Windsurf**, or other editors with MCP support, configure the OpenChoreo MCP server as above and reference the `SKILL.md` files directly in your system prompt or context window.

---

## Repository structure

```
openchoreo-developer/
  SKILL.md                  # Developer skill guide — loaded by AI agent
  references/
    concepts.md
    cli-reference.md
    deployment-guide.md
    resource-schemas.md
    mcp-reference.md        # MCP tool guide for developers
    platform-engineer.md

openchoreo-platform-engineer/
  SKILL.md                  # Platform engineer skill guide — loaded by AI agent
  references/
    operations.md
    templates-and-workflows.md
    integrations.md
    observability.md
    troubleshooting.md
    cli-and-resources.md
    mcp-reference.md        # MCP tool guide for platform engineers
    gitops.md
    community-modules.md
    advanced-setup.md
    repo-and-context7.md

refresh-openchoreo-mcp.sh   # Helper script to configure the MCP server
```

---

## Use cases

- AI agents automating developer workflows on OpenChoreo
- Self-service developer portals backed by an AI agent
- Platform engineering automation via MCP
- CI/CD and GitOps integrations
- Migrating existing Kubernetes workloads into OpenChoreo-managed components
- Onboarding legacy services into OpenChoreo through conversational AI

---

## Contributing

Contributions are welcome.

You can:

- Add new reference files
- Improve existing skills or references
- Add examples
- Improve the MCP tool guides

Please keep the progressive-discovery pattern: `SKILL.md` stays lean and routes to references rather than duplicating content.

---

## License

Same license as OpenChoreo / original skills unless stated otherwise.

---

## Maintainer

Lakmal Warusawithana
OpenChoreo / WSO2

Extending original skills created by
Isala Piyarisi
