# Code-Agent Workspace

Use this reference when the devcontainer is expected to host a coding agent, not just a human editor shell.

## Detection signals

Scan for repo-local agent guidance before generating files:

- `.github/copilot-instructions.md`
- `.github/instructions/**/*.instructions.md`
- `AGENTS.md`
- `Copilot.md`, `GEMINI.md`, `CODEX.md`
- `.claude/`, `.cursor/`, or similar repo-local agent config
- `.mcp.json`
- docs or scripts that run Copilot CLI, Claude Code, Cursor, or another agent CLI

Preserve existing guidance. If it conflicts with the generated devcontainer docs, call out the conflict instead of overwriting it.

## Agent state and auth persistence

Decide whether agent state should be ephemeral or persisted.

| Mode | Use when | Pattern |
|---|---|---|
| Ephemeral | CI-like runs, shared containers, privacy-sensitive work | no agent state mount; authenticate per session if needed |
| Per-project persistent | normal local agent use where rebuilds should keep auth/settings/session state | named volume with `${devcontainerId}` |
| Shared host state | rare; user explicitly wants host settings reused and accepts leakage risk | mount only specific supported paths, read-only where possible |

Prefer per-project volumes over broad host mounts. Examples:

```json
{
  "mounts": [
    "source=copilot-state-${devcontainerId},target=/tmp/home/.copilot,type=volume"
  ]
}
```

Adapt the target path to the chosen agent and container user. Common examples:

| Agent surface | Typical state path | Notes |
|---|---|---|
| Copilot CLI | `~/.copilot` | session state, settings, local CLI data |
| Claude Code | `~/.claude` | auth, settings, session history |
| GitHub CLI | `~/.config/gh` | already covered by shared/isolated GitHub auth paths |

For hosted environments such as Codespaces, prefer supported secrets or environment variables over mounting host credential files.

## Instruction and policy files

Do not generate broad agent instructions as part of the devcontainer unless the user asks. Instead:

- detect existing instruction files;
- make sure they are inside the mounted workspace;
- document which files the chosen agent will read;
- keep generated `.devcontainer/README.md` aligned with those instructions.

For organization-managed policy, use the agent's supported managed settings or server-side policy mechanism when available. Repository Dockerfiles are useful for team defaults, but anyone with write access can change them.

## Agent CLI install policy

Ask before installing an agent CLI inside the container.

Choose the install path in this order:

1. **Host IDE attach** when the agent is an IDE feature and does not need an in-container CLI.
2. **Official Dev Container Feature** when the agent vendor provides one and version/update behavior is acceptable.
3. **Dockerfile install** when you need pinning, reproducible builds, disabled auto-update, or extra policy files.
4. **No install** when the devcontainer only needs compatibility for the host-side agent.

If an agent auto-updates itself, document that behavior. If reproducibility matters, pin the agent CLI version through the Dockerfile and disable auto-update only when the vendor supports doing so.

## MCP-aware setup

When `.mcp.json` or MCP docs exist:

- keep MCP config at project scope when it is meant to travel with the repo;
- install stdio server binaries and runtime dependencies in the Dockerfile only when the repo actually needs them;
- avoid baking secrets into MCP config;
- include MCP tools in Copilot CLI permission guidance when Copilot CLI is the agent surface;
- verify MCP servers after the container is built.

For remote MCP servers, document the required domains for URL permission prompts or enterprise network policy. Do not grant broad MCP access by default.

## Host vs container workflow

Be explicit about where each workflow runs.

Use the **container** for:

- code search, editing, tests, linters, formatters, package managers;
- web/documentation research through the agent's approved URL flow;
- headless builds and scripts;
- high-trust agent execution that should not touch the host outside mounted paths.

Use the **host** for:

- mobile simulators, native device workflows, or OS-specific tooling that the container cannot support;
- browser/UI flows that require host GUI integration unless the repo already provides a container-friendly path;
- secrets or credential stores that should not be mounted into the container.

If both are needed, document the split in `.devcontainer/README.md` so the agent does not try to automate unsupported UI/native flows inside the container.
