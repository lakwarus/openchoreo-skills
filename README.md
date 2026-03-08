# OpenChoreo Skills

This repository contains additional **OpenChoreo Skills** implemented on top of the original skill framework created for OpenChoreo.  
The goal of this repository is to extend the skill set to support more automation scenarios, including Git repository operations, platform workflows, and developer self-service capabilities.

These skills are designed to work with the OpenChoreo MCP / Skills runtime and can be used by AI agents, automation workflows, or developer portal integrations.

---

## Acknowledgement

The original OpenChoreo skills framework was created by **Isala Piyarisi** and is available here:

https://github.com/isala404/openchoreo/tree/occ-skill/docs/skills

This repository builds on top of that work and extends the skill set with additional capabilities while keeping the same skill model and structure.

Special thanks to Isala for creating the initial developer skills foundation.

---

## What this repository adds

This repo extends the original skills with:

- Git repository automation skills
- Additional developer self-service skills
- Platform workflow skills
- Extended MCP-compatible skill definitions
- Examples for building custom skills

Current skills include:

- create-git-repo
- create-project
- create-component
- trigger-workflow
- list-resources
- get-logs
- get-deployment-status

(more skills will be added over time)

---

## Use cases

These skills can be used for:

- AI agents automating developer workflows
- Self-service developer portals
- Platform engineering automation
- CI/CD and GitOps integrations
- Internal developer platform extensions
- OpenChoreo MCP integrations
- Migrating existing Kubernetes workloads into OpenChoreo-managed components
- Converting existing K8s deployments into skill-driven platform workflows
- Automating onboarding of legacy services into OpenChoreo
  

---

## Compatibility

These skills are designed for:

- OpenChoreo
- OpenChoreo MCP server
- OpenChoreo Skills runtime
- AI agent integrations (Codex / MCP / LLM agents)
- Kubernetes-based OpenChoreo deployments

---

## Structure

```
skills/
  gitrepo/
  developer/
  workflow/
  platform/
docs/
examples/
```

Each skill contains:

- skill.yaml
- schema.json
- implementation
- examples

---

## Creating new skills

To create a new skill:

1. Copy an existing skill  
2. Update `skill.yaml`  
3. Define input / output schema  
4. Implement handler  
5. Register skill in MCP server  

Example:

```
skills/
  create-git-repo/
    skill.yaml
    schema.json
    handler.go
```

---

## Contributing

Contributions are welcome.

You can:

- Add new skills
- Improve existing skills
- Fix schemas
- Add examples
- Improve documentation

Please keep compatibility with the original skill model.

---

## License

Same license as OpenChoreo / original skills unless stated otherwise.

---

## Maintainer

Lakmal Warusawithana  
OpenChoreo / WSO2  

Extending original skills created by  
Isala Piyarisi

---

## Repository Description (short)

Extended OpenChoreo skills including GitRepo, workflow, and developer automation skills.  
Based on original skills by Isala Piyarisi.
