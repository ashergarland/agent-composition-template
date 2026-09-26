# Agent Composition Template

A small, reusable repository for composing one canonical agent from published
[Agent Tool Platform](https://github.com/ashergarland/agent-tool-platform) packages.
It turns human-edited agent source into a deterministic Agent Kit lock and VS Code host adapter.
It does not execute capabilities, own environment preparation or provider setup, deploy anything,
persist Agent Instance state, or manage agents.

This repository is intended to become a GitHub template after implementation and independent
review. The future flow is:

```text
agent-composition-template
  -> Use this template
  -> new agent repository
  -> edit canonical source
  -> build
  -> commit deterministic lock and adapter
  -> Prepare later
```

## Source and generated files

Canonical, human-edited source:

- `agent.yaml` holds the Agent Kit schema version, stable identity, version, capability selections,
  optional exact capability versions/profiles, and the repository-relative instruction file.
- `instructions/agent.md` holds the top-level agent instructions.
- `routing/workflows.yaml`, `workflows/`, and `evaluations/` are agent-owned source locations for
  concrete repositories. M5.5 does not feed them to Agent Kit or implement routing, workflow, or
  evaluation behavior.

Generated, reviewable outputs:

- `agent.lock` records the exact capability releases, profiles, bindings, artifacts, source
  revisions, requirements, registry-entry digests, and adapter version selected by Agent Kit.
- `.github/agents/<agent-id>.agent.md` is the generated VS Code agent adapter.
- `.vscode/mcp.json` is the generated VS Code MCP configuration.

The entire `.github/agents/` directory is owned by this generator for `*.agent.md` files. A build
removes stale generated agent files, such as an adapter left by an agent ID change. Do not hand-edit
generated files; edit canonical source and rebuild.

## Create an agent

1. Create a repository from this template once its GitHub template setting has been enabled.
2. Run `npm ci`.
3. Edit identity and capability selections in `agent.yaml`.
4. Edit the top-level instructions in `instructions/agent.md`.
5. Add agent-owned routing, workflow, and evaluation source as the concrete agent requires. These
   locations are not runtime implementations and are not consumed by this M5.5 build.
6. Run `npm run agent:build`.
7. Review the source and generated diff.
8. Run `npm test` and `npm run agent:check`.
9. Commit the source, `package-lock.json`, `agent.lock`, and generated VS Code files together.

Capability IDs come from the static first-party Registry carried by the exact installed
`@agent-tool-platform/capability-registry` package. Its public
`loadFirstPartyCapabilityRegistry()` and `createCapabilityRegistryReader()` APIs expose
`reader.listCapabilities()` for discovery. Do not copy Registry records into an agent repository.
Omitting a capability version resolves the current entry from the pinned Registry package; Agent
Kit records the exact resolved version in `agent.lock`. Set `version` or `profile` on a capability
selection only when the agent intentionally requires that exact Registry value.

Example selection:

```yaml
capabilities:
  - id: ast-summarizer
  - id: git-optimizer
    version: "0.1.0"
    profile: local-package
```

## Build and check

```powershell
npm run agent:build
npm run agent:check
```

The build reads `agent.yaml` and its instruction file, constructs the canonical Agent Kit
`AgentDefinition`, loads the published first-party Registry, and calls `buildVsCodeAgent()`.
Build remains the canonical composition-generation step. Agent Kit remains authoritative for
definition validation, capability resolution, lock semantics, readiness semantics, and adapter
generation. The YAML layer only represents the instruction file as a repository-relative source
reference.

`agent:build` writes all generated outputs and reports Agent Kit's derived readiness states without
installing capability artifacts or changing the environment. `agent:check` computes the same
outputs in memory and fails if checked-in bytes are missing, stale, or accompanied by an obsolete
generated agent file. It never repairs files.

For identical canonical source and `package-lock.json`, repeated builds are byte-identical. Normal
builds use only the static Registry data in the installed exact package; they do not query a mutable
network registry.

## Platform versions and upgrades

The template pins `@agent-tool-platform/agent-kit@0.4.0` and
`@agent-tool-platform/capability-registry@0.4.0` as exact direct dependencies. Runtime is not a
direct dependency; Agent Kit carries `@agent-tool-platform/runtime@0.4.0` transitively.

Perform an intentional lockstep Platform upgrade explicitly:

```powershell
npm install --save-exact @agent-tool-platform/agent-kit@X.Y.Z `
  @agent-tool-platform/capability-registry@X.Y.Z
npm test
npm run agent:build
npm run agent:check
```

Review `package.json`, `package-lock.json`, `agent.lock`, and both adapter files before committing.
Never substitute workspace, `file:`, or `link:` dependencies: this repository is an independent
consumer proof.

## Lifecycle boundary

This template covers canonical agent source through Agent Kit Build, lock, and host-adapter
generation. Platform 0.4.0 also provides public `createPreparationPlan()`, `prepareAgent()`, and
Agent Instance APIs for downstream consumers. The template verifies that its Build output is
accepted by those APIs using synthetic readiness evidence, but it is not itself a Prepare
implementation and creates no persistent Agent Instance state.

Capability installation, MCP process startup, provider authentication and setup, cloud resources,
deployment, Agent Instance persistence and management, fleet management, and telemetry remain
outside this template.
