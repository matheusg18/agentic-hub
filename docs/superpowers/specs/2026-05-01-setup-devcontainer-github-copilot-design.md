# Setup Devcontainer for GitHub and GitHub Copilot

## Problem

The repository contains an empty `plugins/essentials/skills/setup-devcontainer/SKILL.md`. We need a complete replacement for the upstream `setup-devcontainer` skill that preserves its overall structure and depth, but adapts it from a GitLab + Claude Code/opencode workflow to a GitHub + GitHub Copilot workflow.

The goal is not to force a false 1:1 mapping between Claude Code and GitHub Copilot. The skill should keep the same seven-phase workflow and hardened devcontainer focus, while replacing forge- and agent-specific logic with the closest supported GitHub equivalents.

## Approach

We will keep the skill name and location:

- `plugins/essentials/skills/setup-devcontainer/SKILL.md`

We will also create a `references/` directory beside it so the skill can stay readable and delegate detailed patterns to focused reference documents.

## Scope

### In scope

- Preserve the original seven phases:
  1. Project analysis
  2. Dockerfile generation
  3. `devcontainer.json` generation
  4. CI validation
  5. Optional firewall
  6. Task runner integration
  7. Testing
- Replace GitLab-specific guidance with GitHub-specific guidance:
  - `.gitlab-ci.yml` -> `.github/workflows/*.yml`
  - `glab` -> `gh`
  - GitLab runner detection -> GitHub Actions workflow and runner signal analysis
- Replace Claude Code/opencode-centric guidance with GitHub Copilot-centric guidance:
  - remove `.claude` and opencode-specific config mounts
  - center the skill on GitHub-authenticated development environments and GitHub Copilot-compatible IDE/devcontainer workflows
- Keep the original emphasis on hardened Dockerfiles, CI image reuse, pinned dependencies, arbitrary UID support, SSH agent forwarding, optional Docker socket support, optional firewall mode, and task runner integration
- Rewrite the verification checklist around GitHub and GitHub Copilot-compatible signals

### Out of scope

- Inventing unsupported GitHub Copilot install or config flows just to mirror Claude Code behavior
- Preserving GitLab-only behavior in the new skill
- Adding unrelated new devcontainer features beyond what is needed for the GitHub/Copilot adaptation

## Design

### Phase 1: Project analysis

The skill will inspect:

- `.github/workflows/*.yml`
- Dockerfiles, compose files, and task runners
- project manifests and tool configs
- Git LFS, Kubernetes, Docker, and registry signals already handled by the upstream skill

It will still ask the user targeted questions when repository evidence is not sufficient.

The GitHub adaptation changes phase 1 in three key ways:

1. CI image reuse is based on GitHub Actions workflows rather than GitLab jobs.
2. Forge CLI integration is based on `gh` and `gh auth status`.
3. GitHub Copilot is treated as a compatibility target for GitHub-authenticated IDE/devcontainer environments, not as a drop-in standalone binary analogous to Claude Code.

### Phase 2: Dockerfile

The generated Dockerfile guidance will retain the upstream hardening model:

- pinned base images with digests
- Renovate annotations for versioned tooling
- early `HOME` and `PATH` setup
- arbitrary UID entrypoint
- no `WORKDIR`
- sticky-bit permissions where appropriate
- optional Docker CLI, firewall, Git LFS, Kubernetes, and ecosystem dev tool layers

GitHub-specific tooling should prioritize `gh`, git, SSH, and whatever is required to keep the container compatible with GitHub-hosted development workflows.

### Phase 3: devcontainer.json

The skill will still generate a full `devcontainer.json`, but the personalization questions will be rewritten around:

- GitHub-authenticated development
- IDE/devcontainer compatibility
- optional Docker access
- optional firewall mode
- generic compatibility with GitHub Copilot usage patterns instead of Claude-specific mounts

### Phase 4: CI validation

Validation moves from GitLab CI to GitHub Actions. The skill will either:

- add a dedicated workflow for devcontainer validation, or
- provide instructions consistent with an existing workflow layout

The workflow should validate the generated devcontainer configuration and build behavior when `.devcontainer/**/*` changes.

### Phase 5: Firewall

The firewall phase remains optional and keeps the same purpose: enabling a more constrained autonomous/sandbox mode.

The allowlist logic must be updated to GitHub-oriented defaults, including GitHub-hosted endpoints and project registry endpoints inferred from the repository.

### Phase 6: Task runner integration

The skill will continue to detect and integrate with existing task runners such as `just`, `make`, `task`, or package manager scripts.

The generated guidance should wrap container launch in a stable local command, support conditional mounts, support optional Docker socket usage, and keep the firewall mode opt-in.

### Phase 7: Testing

The final verification checklist will be rewritten to validate:

- toolchain access inside the container
- arbitrary UID behavior
- git and SSH behavior
- `gh --version`
- `gh auth status`
- project-specific build and lint commands
- Docker, Git LFS, Kubernetes, and firewall checks when those features are enabled
- GitHub/Copilot-compatible behavior only where the signal is real and observable

## References to create

The skill should include a `references/` directory mirroring the upstream pattern, adapted to GitHub and GitHub Copilot:

- `dockerfile.md`
- `devcontainer-json.md`
- `firewall.md`
- `task-runner.md`
- `docker-support.md`
- `dev-tools.md`
- `common-mistakes.md`

Each reference should capture concrete patterns and decision rules so the main skill file can stay procedural and concise.

## Error handling

The skill must not guess when evidence is missing. When GitHub workflows, authenticated `gh`, or Copilot-specific requirements are unclear, it should:

1. say what is missing
2. ask the user directly
3. fall back to a hardened generic devcontainer path rather than fabricating unsupported assumptions

## Validation and review

Before implementation is considered complete, the repository should contain:

- a non-empty `SKILL.md`
- all referenced markdown files present under `references/`
- internal links that resolve
- language consistent with GitHub + GitHub Copilot rather than GitLab + Claude Code/opencode
