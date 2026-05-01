# Setup Devcontainer GitHub Copilot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `setup-devcontainer` skill under `plugins/essentials/skills/setup-devcontainer/` as a GitHub + GitHub Copilot adaptation of the upstream GitLab + Claude Code/opencode skill.

**Architecture:** Keep the upstream seven-phase workflow and reference-driven structure, but rewrite the forge-specific logic around GitHub Actions and `gh`, and rewrite the AI-tool-specific logic around GitHub-authenticated IDE/devcontainer workflows that are compatible with GitHub Copilot. Do not fake unsupported Claude-style behavior; when GitHub/Copilot signals are missing, fall back to explicit user questions and a hardened generic devcontainer path.

**Tech Stack:** Markdown skill files, GitHub-style skill metadata, bash/rg/git for validation, existing upstream skill content as source material.

---

## File Structure

- Modify: `plugins/essentials/skills/setup-devcontainer/SKILL.md` — main procedural skill with the seven phases and links to reference docs.
- Create: `plugins/essentials/skills/setup-devcontainer/references/dockerfile.md` — Dockerfile rules, GitHub Actions image reuse, `gh` tooling, UID entrypoint guidance.
- Create: `plugins/essentials/skills/setup-devcontainer/references/devcontainer-json.md` — `devcontainer.json` patterns for GitHub-authenticated, IDE-compatible setups.
- Create: `plugins/essentials/skills/setup-devcontainer/references/task-runner.md` — `just`/`make`/`task`/script wrapper patterns, Docker socket option, firewall flag.
- Create: `plugins/essentials/skills/setup-devcontainer/references/firewall.md` — GitHub-oriented allowlist logic and firewall rules.
- Create: `plugins/essentials/skills/setup-devcontainer/references/docker-support.md` — Docker CLI + Compose detection and socket/GID handling.
- Create: `plugins/essentials/skills/setup-devcontainer/references/dev-tools.md` — ecosystem tool detection and install-scope rules.
- Create: `plugins/essentials/skills/setup-devcontainer/references/common-mistakes.md` — red flags and anti-patterns specific to the GitHub/Copilot adaptation.
- Input reference: `docs/superpowers/specs/2026-05-01-setup-devcontainer-github-copilot-design.md` — approved design to keep implementation aligned.

### Task 1: Replace the empty main skill file

**Files:**
- Modify: `plugins/essentials/skills/setup-devcontainer/SKILL.md`
- Validate: `docs/superpowers/specs/2026-05-01-setup-devcontainer-github-copilot-design.md`

- [ ] **Step 1: Confirm the baseline is still empty**

Run: `test -s plugins/essentials/skills/setup-devcontainer/SKILL.md && echo non-empty || echo empty`
Expected: `empty`

- [ ] **Step 2: Write the front matter and intro**

```md
---
name: setup-devcontainer
description: Generates a hardened .devcontainer/ setup for GitHub-hosted development and GitHub Copilot-compatible IDE use. Analyzes project toolchain, reuses GitHub Actions container images when possible, pins dependencies, handles arbitrary UIDs, SSH agent forwarding, optional Docker socket support, and optional network firewall. Triggers on "devcontainer", "dev container", "containerize development", "GitHub Copilot in a container", "GitHub devcontainer".
license: MIT
compatibility: Requires docker and bash. Generates configs for the Dev Container spec and GitHub-oriented development workflows.
metadata:
  author: matheusg18
  version: "1.0"
---

# Set Up a Dev Container

Create a `.devcontainer/` setup following the Dev Container spec that works for GitHub-based development and GitHub Copilot-compatible IDE use.
```

- [ ] **Step 3: Fill the seven phases and rewrite the references**

```md
## Process

### Phase 1: Project Analysis
- inspect `.github/workflows/*.yml`, Dockerfiles, compose files, manifests, and task runners
- detect GitHub Actions container images and reuse them when suitable
- ask targeted questions when GitHub runner or Copilot expectations are not observable

### Phase 4: CI Validation
- generate GitHub Actions validation guidance instead of `.gitlab-ci.yml`

### Phase 7: Testing
- validate `gh --version`, `gh auth status`, toolchain access, UID behavior, git/SSH, and optional Docker/firewall features

## Reference
- [references/dockerfile.md](references/dockerfile.md)
- [references/devcontainer-json.md](references/devcontainer-json.md)
- [references/firewall.md](references/firewall.md)
- [references/task-runner.md](references/task-runner.md)
- [references/common-mistakes.md](references/common-mistakes.md)
- [references/docker-support.md](references/docker-support.md)
- [references/dev-tools.md](references/dev-tools.md)
```

- [ ] **Step 4: Validate the structure and terminology**

Run: `rg '^### Phase [1-7]:' plugins/essentials/skills/setup-devcontainer/SKILL.md && ! rg 'GitLab|glab|Claude Code|opencode|voice mode' plugins/essentials/skills/setup-devcontainer/SKILL.md`
Expected: seven `### Phase` matches and no GitLab/Claude/opencode leftovers

- [ ] **Step 5: Commit**

```bash
git add plugins/essentials/skills/setup-devcontainer/SKILL.md
git commit -m "docs(skill): rewrite setup-devcontainer for GitHub Copilot" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task 2: Create the Dockerfile reference

**Files:**
- Create: `plugins/essentials/skills/setup-devcontainer/references/dockerfile.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/SKILL.md`

- [ ] **Step 1: Confirm the reference file does not exist yet**

Run: `test -f plugins/essentials/skills/setup-devcontainer/references/dockerfile.md && echo exists || echo missing`
Expected: `missing`

- [ ] **Step 2: Write the Dockerfile guidance**

```md
# Dockerfile Reference

## GitHub-oriented base image reuse
- inspect GitHub Actions workflows for `container:` images and reusable toolchain images
- prefer reusing those images through multi-stage builds when they already contain the right stack

## Hardening rules
- pin images with `tag@sha256:digest`
- add Renovate annotations for versioned tools
- set `HOME` and `PATH` before installs
- do not set `WORKDIR`
- use sticky-bit permissions where needed

## GitHub tooling
- install `gh`
- keep git + SSH available
- do not describe unsupported GitHub Copilot standalone install flows
```

- [ ] **Step 3: Validate the main skill points to this file**

Run: `rg 'references/dockerfile.md' plugins/essentials/skills/setup-devcontainer/SKILL.md plugins/essentials/skills/setup-devcontainer/references/dockerfile.md`
Expected: at least one match in `SKILL.md` and one heading in the new reference

- [ ] **Step 4: Commit**

```bash
git add plugins/essentials/skills/setup-devcontainer/references/dockerfile.md
git commit -m "docs(skill): add Dockerfile reference for setup-devcontainer" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task 3: Create the devcontainer.json reference

**Files:**
- Create: `plugins/essentials/skills/setup-devcontainer/references/devcontainer-json.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/SKILL.md`

- [ ] **Step 1: Confirm the reference file does not exist yet**

Run: `test -f plugins/essentials/skills/setup-devcontainer/references/devcontainer-json.md && echo exists || echo missing`
Expected: `missing`

- [ ] **Step 2: Write the devcontainer guidance**

```md
# devcontainer.json Reference

## Supported modes
- GitHub-authenticated development
- generic IDE/devcontainer compatibility
- optional Docker socket support
- optional firewall mode

## GitHub Copilot guidance
- center on IDE and GitHub-authenticated workflows
- avoid `.claude` mounts and opencode-specific directories
- prefer read-only GitHub-related config mounts only when the host files actually exist
```

- [ ] **Step 3: Validate the terminology**

Run: `! rg '\\.claude|opencode|Claude Code' plugins/essentials/skills/setup-devcontainer/references/devcontainer-json.md`
Expected: no matches

- [ ] **Step 4: Commit**

```bash
git add plugins/essentials/skills/setup-devcontainer/references/devcontainer-json.md
git commit -m "docs(skill): add devcontainer json reference" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task 4: Create the task runner reference

**Files:**
- Create: `plugins/essentials/skills/setup-devcontainer/references/task-runner.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/SKILL.md`

- [ ] **Step 1: Confirm the reference file does not exist yet**

Run: `test -f plugins/essentials/skills/setup-devcontainer/references/task-runner.md && echo exists || echo missing`
Expected: `missing`

- [ ] **Step 2: Write the task runner guidance**

```md
# Task Runner Reference

## Detection
- justfile
- Makefile
- Taskfile.yml
- package manager scripts

## Recipe requirements
- one stable local entry point such as `just dev-shell`
- optional Docker socket mount
- optional `--firewall` mode
- conditional mounts only when host files exist
- prefer GitHub auth reuse through safe, explicit mounts instead of tool-specific hidden directories
```

- [ ] **Step 3: Validate references and wording**

Run: `rg 'references/task-runner.md|dev-shell|firewall' plugins/essentials/skills/setup-devcontainer/SKILL.md plugins/essentials/skills/setup-devcontainer/references/task-runner.md`
Expected: matches in both files

- [ ] **Step 4: Commit**

```bash
git add plugins/essentials/skills/setup-devcontainer/references/task-runner.md
git commit -m "docs(skill): add task runner reference" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task 5: Create the firewall reference

**Files:**
- Create: `plugins/essentials/skills/setup-devcontainer/references/firewall.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/SKILL.md`

- [ ] **Step 1: Confirm the reference file does not exist yet**

Run: `test -f plugins/essentials/skills/setup-devcontainer/references/firewall.md && echo exists || echo missing`
Expected: `missing`

- [ ] **Step 2: Write the firewall guidance**

```md
# Firewall Reference

## Use only for autonomous or sandboxed modes
- default to normal development without the firewall
- require explicit opt-in

## GitHub-oriented allowlist defaults
- `github.com`
- `api.github.com`
- project package registries inferred from manifests
- container registries required by the project or Docker support

## Behavior
- default DROP policy
- allow only necessary DNS, SSH, HTTP, and HTTPS traffic
- self-test blocked and allowed domains
```

- [ ] **Step 3: Validate GitHub-specific wording**

Run: `rg 'github.com|api.github.com|DROP policy|self-test' plugins/essentials/skills/setup-devcontainer/references/firewall.md`
Expected: matches for the GitHub endpoints and firewall behavior

- [ ] **Step 4: Commit**

```bash
git add plugins/essentials/skills/setup-devcontainer/references/firewall.md
git commit -m "docs(skill): add firewall reference" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task 6: Create the Docker support reference

**Files:**
- Create: `plugins/essentials/skills/setup-devcontainer/references/docker-support.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/SKILL.md`

- [ ] **Step 1: Confirm the reference file does not exist yet**

Run: `test -f plugins/essentials/skills/setup-devcontainer/references/docker-support.md && echo exists || echo missing`
Expected: `missing`

- [ ] **Step 2: Write the Docker support guidance**

```md
# Docker Support Reference

## Detection signals
- compose files
- Dockerfiles outside `.devcontainer/`
- task runner commands using `docker build`, `docker compose`, or similar

## Guidance
- install Docker CLI and Compose only when needed or explicitly requested
- handle socket GID mapping safely
- add Docker registry domains to the firewall allowlist only when Docker support is enabled
```

- [ ] **Step 3: Validate the cross-reference**

Run: `rg 'references/docker-support.md|Docker support' plugins/essentials/skills/setup-devcontainer/SKILL.md plugins/essentials/skills/setup-devcontainer/references/docker-support.md`
Expected: matches in both files

- [ ] **Step 4: Commit**

```bash
git add plugins/essentials/skills/setup-devcontainer/references/docker-support.md
git commit -m "docs(skill): add docker support reference" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task 7: Create the dev tools and common mistakes references

**Files:**
- Create: `plugins/essentials/skills/setup-devcontainer/references/dev-tools.md`
- Create: `plugins/essentials/skills/setup-devcontainer/references/common-mistakes.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/SKILL.md`

- [ ] **Step 1: Confirm both reference files are missing**

Run: `for f in plugins/essentials/skills/setup-devcontainer/references/dev-tools.md plugins/essentials/skills/setup-devcontainer/references/common-mistakes.md; do test -f "$f" && echo "exists $f" || echo "missing $f"; done`
Expected: two `missing` lines

- [ ] **Step 2: Write the dev tools guidance**

```md
# Dev Tools Reference

## Detection
- inspect CI commands
- inspect dependency manifests and tool config files

## Install scope
- project-managed tools stay in the project dependency graph
- globally invoked tools missing from project deps can be installed in the container

## Examples
- ruff
- golangci-lint
- prettier
- kubectl / helm / helmfile when repository signals exist
```

- [ ] **Step 3: Write the common mistakes guidance**

```md
# Common Mistakes

- Do not copy GitLab runner logic into GitHub Actions wording
- Do not reference `glab`, `.gitlab-ci.yml`, `.claude`, or `opencode`
- Do not promise unsupported GitHub Copilot install or auth paths
- Do not enable the firewall by default for normal development
- Do not guess when repository evidence is missing
```

- [ ] **Step 4: Validate the terminology**

Run: `! rg 'GitLab|glab|\\.gitlab-ci|\\.claude|opencode' plugins/essentials/skills/setup-devcontainer/references/dev-tools.md plugins/essentials/skills/setup-devcontainer/references/common-mistakes.md`
Expected: no matches

- [ ] **Step 5: Commit**

```bash
git add plugins/essentials/skills/setup-devcontainer/references/dev-tools.md plugins/essentials/skills/setup-devcontainer/references/common-mistakes.md
git commit -m "docs(skill): add remaining setup-devcontainer references" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task 8: Run end-to-end validation across the skill

**Files:**
- Validate: `plugins/essentials/skills/setup-devcontainer/SKILL.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/references/dockerfile.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/references/devcontainer-json.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/references/task-runner.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/references/firewall.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/references/docker-support.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/references/dev-tools.md`
- Validate: `plugins/essentials/skills/setup-devcontainer/references/common-mistakes.md`

- [ ] **Step 1: Verify every referenced file exists**

Run: `for f in plugins/essentials/skills/setup-devcontainer/references/dockerfile.md plugins/essentials/skills/setup-devcontainer/references/devcontainer-json.md plugins/essentials/skills/setup-devcontainer/references/task-runner.md plugins/essentials/skills/setup-devcontainer/references/firewall.md plugins/essentials/skills/setup-devcontainer/references/docker-support.md plugins/essentials/skills/setup-devcontainer/references/dev-tools.md plugins/essentials/skills/setup-devcontainer/references/common-mistakes.md; do test -f "$f" || exit 1; done && echo ok`
Expected: `ok`

- [ ] **Step 2: Verify the main skill links to every reference**

Run: `for ref in dockerfile devcontainer-json firewall task-runner docker-support dev-tools common-mistakes; do rg "references/${ref}\\.md" plugins/essentials/skills/setup-devcontainer/SKILL.md >/dev/null || exit 1; done && echo ok`
Expected: `ok`

- [ ] **Step 3: Verify upstream-specific leftovers are gone**

Run: `! rg 'GitLab|glab|\\.gitlab-ci|Claude Code|opencode|voice mode' plugins/essentials/skills/setup-devcontainer`
Expected: no matches

- [ ] **Step 4: Verify GitHub-specific terms are present**

Run: `rg 'GitHub|GitHub Actions|GitHub Copilot|gh auth status|\\.github/workflows' plugins/essentials/skills/setup-devcontainer/SKILL.md plugins/essentials/skills/setup-devcontainer/references/*.md`
Expected: multiple matches across the skill and references

- [ ] **Step 5: Commit**

```bash
git add plugins/essentials/skills/setup-devcontainer
git commit -m "docs(skill): finish setup-devcontainer GitHub Copilot adaptation" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```
