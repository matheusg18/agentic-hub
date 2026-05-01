---
name: setup-devcontainer
description: Generates a hardened .devcontainer/ setup for GitHub-authenticated development and GitHub Copilot-compatible IDE use. Analyses the project toolchain, reuses CI images, pins dependencies with Renovate annotations, handles arbitrary UIDs, git worktrees, SSH agent forwarding, optional Docker access, and optional network firewall. Triggers on "devcontainer", "dev container", "containerise development", "run GitHub tooling in a container", and "GitHub Copilot devcontainer".
license: MIT
compatibility: Requires docker and bash. Generates configs for the Dev Container spec (VS Code, JetBrains, DevPod, devcontainer CLI).
metadata:
  author: lx-industries
  version: "1.0"
---

# Set Up a Dev Container

Create a `.devcontainer/` setup following the [Dev Container spec](https://containers.dev/) that works for GitHub-authenticated development and GitHub Copilot-compatible IDE use (VS Code, JetBrains, DevPod, devcontainer CLI).

## Process

### Phase 1: Project Analysis

Before writing any files, investigate the project thoroughly.

**1. Identify the language ecosystem and build tools:**

```
- Primary language(s) and version(s)
- Package manager(s) and their registry URLs (cargo → crates.io, npm → registry.npmjs.org, pip → pypi.org, go → proxy.golang.org, etc.)
- Build tools (just, make, gradle, etc.)
- Task runner (justfile, Makefile, Taskfile.yml, package.json scripts)
- Required system libraries
```

Registry URLs are needed for Phase 5 (firewall allowlist) if the user opts in.

**2. Check for existing CI container images:**

Look in CI configuration, container registries, and Dockerfiles:
- `.github/workflows/`, `Jenkinsfile`
- `images/`, `docker/`, or similar directories
- Container registry for the project (if any, including GHCR)

If the project already builds CI images with the right toolchain, **reuse them as base images** via multi-stage build. This avoids duplicating toolchain setup and keeps the dev container aligned with CI.

**3. Choose the development mode:**

Ask: "Should the container share host GitHub auth and local developer settings, start isolated, or support both?"
- A) Shared host auth and local settings (default)
- B) Isolated container state only
- C) Both, with one path chosen as the default

This choice propagates through all subsequent phases. Host-sharing logic is conditional on the shared path being selected. Isolated-volume logic is conditional on the isolated path being selected.

**4. Identify additional tools needed beyond CI:**

The dev container likely needs tools CI images lack:
- GitHub CLI (`gh`)
- SSH client for git operations
- git and certificate tooling needed for GitHub-authenticated workflows
- Any interactive development tools

Treat GitHub Copilot as an IDE compatibility target, not as a standalone binary to install inside the container unless the project provides an explicit, supported requirement.

**5. Detect ecosystem dev tools:**

Identify dev/lint/format/test tools the project depends on by scanning two sources:

- **CI configs** — look for tool invocations in scripts (for example `cargo clippy`, `ruff check`, `golangci-lint run`, `prettier --check`)
- **Project manifests** — look for tool declarations in dependency files and config files (for example `pyproject.toml` dev groups, `package.json` devDependencies, `.golangci.yml`, `.clippy.toml`)

For each detected tool, determine the install scope:
- **Project-managed** — declared in the project's dependency file → skip Dockerfile install, the project's package manager handles it
- **Global** — invoked in CI or configured in the project but not a project dependency → install in the Dockerfile

See [references/dev-tools.md](references/dev-tools.md) for detection signals, install scope rules, and Dockerfile patterns per ecosystem.

**6. Check for Git LFS usage:**

Look for signals that the project uses Git Large File Storage:
- `.gitattributes` containing `filter=lfs` entries
- `.lfsconfig` file (may contain a custom `lfs.url`)

If detected, install `git-lfs` in the Dockerfile (see [references/dockerfile.md](references/dockerfile.md) for the pattern) and add LFS server domains to the firewall allowlist if Phase 5 is enabled (see [references/firewall.md](references/firewall.md)).

For custom LFS servers, parse the URL from `.lfsconfig` (`lfs.url`) or `git config lfs.url` to extract the hostname for the firewall allowlist.

**7. Check for Kubernetes tooling:**

Look for signals that the project uses Kubernetes, Helm, or Helmfile:
- Manifests: `Chart.yaml`, `helmfile.yaml`, `helmfile.yml`, `kustomization.yaml`, `values.yaml`
- Config: `kubeconfig`, `.helmignore`, `Chart.lock`, `requirements.yaml`
- CI commands: `kubectl apply`, `helm install`, `helm upgrade`, `helm lint`, `helmfile sync`, `helmfile diff`, `kustomize build`
- Task runner scripts referencing `kubectl`, `helm`, or `helmfile`

If signals found, install kubectl/helm/helmfile in the Dockerfile (see [references/dev-tools.md](references/dev-tools.md) for patterns) and forward `KUBECONFIG` + mount `~/.kube` into the container (see [references/devcontainer-json.md](references/devcontainer-json.md) and [references/task-runner.md](references/task-runner.md)).

**8. Check for Docker/Compose usage:**

Look for signals that the project needs Docker access inside the devcontainer:
- Compose files: `docker-compose.yml`, `docker-compose.yaml`, `compose.yml`, `compose.yaml`
- Container build files: `Dockerfile` or `Containerfile` outside `.devcontainer/`
- Testcontainers dependencies in package manifests
- References to `docker build`, `docker compose`, or `podman build` in task runner scripts

See [references/docker-support.md](references/docker-support.md) for the full detection signal list.

If signals found, recommend enabling Docker CLI + Compose support. If none are found, still offer the option.

**9. Check GitHub Actions runner and container signals:**

Inspect `.github/workflows/` for signals that affect devcontainer validation and image reuse:
- Existing container build jobs (`docker build`, `docker/build-push-action`, `devcontainers/ci`)
- Custom runner labels (`runs-on: [self-hosted, ...]`)
- Reusable workflows that already encapsulate container build logic
- Existing registry login steps and pushed image tags

If a workflow already builds the right image or has the right runner constraints, carry those assumptions into Phase 4. If the repository relies on self-hosted runners or private registries and the requirement is unclear, ask the user instead of guessing.

**10. Check GitHub authentication expectations:**

Look for signals that the repository expects authenticated GitHub tooling:
- `gh` usage in scripts, docs, or workflows
- Release, package, issue, PR, or Actions automation in project docs
- SSH-based git remotes or instructions for `gh auth login`

If detected, plan for `gh` installation and a verification path based on `gh auth status`. Also identify whether SSH or HTTPS auth is the expected default for git operations.

**11. Check for GitHub Copilot-compatible workflow needs:**

Ask which environment the user expects to attach from:
- VS Code
- JetBrains Gateway / JetBrains devcontainer support
- DevPod or another Dev Container host
- devcontainer CLI only

Only add mounts, env vars, or workflow guidance that the chosen host genuinely supports. Do not invent editor-specific secret sharing or extension bootstrapping paths.

**12. Present findings to the user** before proceeding.

### Phase 2: Dockerfile

Generate `.devcontainer/Dockerfile` and `.devcontainer/entrypoint.sh`.

See [references/dockerfile.md](references/dockerfile.md) for complete Dockerfile patterns covering: base image pinning with digests, Renovate annotations, NPM supply-chain hardening, layer ordering, GitHub CLI install, git/SSH configuration, permissions (sticky bit), arbitrary UID entrypoint, and worktree compatibility.

Key principles:
- Pin every image with `tag@sha256:digest`
- Every versioned dependency gets a `# renovate:` annotation + `ARG`
- `ENV HOME` and `ENV PATH` set **before** any tool installs
- Never set `WORKDIR` (worktree compatibility)
- `chmod 1777` not `chmod 777`

Always include the GitHub-oriented baseline tooling needed for the chosen workflow: `gh`, git, SSH support, CA certificates, and any proven project requirements from Phase 1.

If Docker support was enabled in Phase 1, also add the Docker CLI layer and `/etc/group` writable. See [references/docker-support.md](references/docker-support.md) for the Dockerfile additions and entrypoint GID handling.

If Git LFS was detected in Phase 1, add `git-lfs` to the system packages layer and run `git lfs install --system`. See [references/dockerfile.md](references/dockerfile.md) for the pattern.

If ecosystem dev tools were detected in Phase 1, add the appropriate install layers. See [references/dev-tools.md](references/dev-tools.md) for Dockerfile patterns, Renovate annotations, and layer placement per ecosystem. Only install tools that need global scope — skip tools managed by the project's dependency file.

If Kubernetes tooling was detected in Phase 1, add kubectl/helm/helmfile install layers. See [references/dev-tools.md](references/dev-tools.md) for Dockerfile patterns and Renovate annotations.

Keep the image compatible with GitHub Copilot-driven IDE workflows by supporting standard shells, git, SSH, forwarded ports, and the editor's normal devcontainer attach flow rather than relying on unsupported in-container editor hacks.

### Phase 3: devcontainer.json

Generate `.devcontainer/devcontainer.json` based on the Phase 1 development mode choice.

See [references/devcontainer-json.md](references/devcontainer-json.md) for both paths (shared host settings vs isolated), mount configurations, and the rationale for each key decision (`init`, workspace mount, SSH agent, COLORTERM, git config handling, and GitHub auth sharing options).

Key decisions:
- **Path A** (shared host settings): reuse host GitHub auth and local developer settings only through supported bind mounts or environment forwarding confirmed in Phase 1; only add GitHub-related host mounts when the host files or directories actually exist
- **Path B** (isolated): named Docker volumes for container state, no host auth, user config, or SSH agent mounts
- Both paths: `"init": true`, host-native `workspaceFolder`, `COLORTERM` forwarding
- Path A only: read-only `~/.gitconfig` and any other supported host config mounts, kept conditional on host file existence
- Path A only: forward the SSH agent socket only when host auth reuse is selected

If Docker support was enabled in Phase 1, add the Docker socket bind mount (`/var/run/docker.sock`). The entrypoint handles GID — no `runArgs` needed.

If Kubernetes tooling was detected in Phase 1, add `~/.kube` bind mount and `KUBECONFIG` env var to `remoteEnv`. See [references/devcontainer-json.md](references/devcontainer-json.md) for the mount and env var configuration.

If authenticated GitHub CLI usage is expected, either:
- reuse supported host `gh` auth state in Path A when the host auth files exist, or
- document the first-run `gh auth login` flow for Path B

See [references/devcontainer-json.md](references/devcontainer-json.md) for the supported mount patterns and fallbacks.

If the user needs forwarded services for browser-based development tooling, add `forwardPorts` and `portsAttributes` only for services confirmed in the repository or explicitly requested by the user.

### Phase 4: CI Validation

Add GitHub Actions jobs that run on changes to `.devcontainer/`.

**Schema validation** (no special runner required):

```yaml
name: Devcontainer Validation

on:
  pull_request:
    paths:
      - .devcontainer/**/*
      - .github/workflows/devcontainer.yml
  push:
    paths:
      - .devcontainer/**/*
      - .github/workflows/devcontainer.yml

jobs:
  devcontainer-validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.x'
      - run: pip install check-jsonschema
      - run: check-jsonschema --schemafile https://raw.githubusercontent.com/devcontainers/spec/main/schemas/devContainer.schema.json .devcontainer/devcontainer.json
```

**Build verification** (preserve existing runner assumptions when present):

If the repository already has a workflow or reusable workflow that builds container images, extend it so `.devcontainer/**/*` changes exercise the devcontainer build path.

If no suitable workflow exists, add a dedicated job such as:

```yaml
  devcontainer-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build .devcontainer/
```

If the repository depends on self-hosted runners, private registries, or pre-existing login steps, preserve those labels, permissions, and authentication steps rather than replacing them with generic defaults.

Both jobs should only trigger on changes to `.devcontainer/**/*` or the workflow file itself.

### Phase 5: Network Firewall (Optional)

**Ask the user:** will this container be used for autonomous or sandboxed execution? If not, skip this phase — the firewall breaks normal development.

See [references/firewall.md](references/firewall.md) for complete firewall implementation: allowlist file generation (auto-detected from Phase 1 registries and GitHub endpoints), `firewall.sh` script, Dockerfile additions (iptables, ipset, gosu), entrypoint modifications (firewall + privilege drop via gosu), and devcontainer.json additions for IDE-based firewall mode.

Key principles:
- Allowlist file with domains, resolved to IPs at startup
- Default DROP policy, allow only DNS/SSH/HTTP/HTTPS to listed domains
- Self-test (verify blocked domain is unreachable)
- `gosu` for privilege drop after firewall setup
- Include GitHub endpoints required by the actual workflow (`github.com`, `api.github.com`, `uploads.github.com`, `ghcr.io`) when those services are in use
- Include package registry domains discovered in Phase 1
- Only add editor- or GitHub Copilot-specific domains when the user's chosen environment demonstrably requires them

If Docker support was enabled in Phase 1, add Docker Hub registry domains (`registry-1.docker.io`, `auth.docker.io`, `production.cloudflare.docker.com`) to the allowlist. Scan project Dockerfiles and compose files for additional registries. See [references/docker-support.md](references/docker-support.md).

If Git LFS was detected in Phase 1, add the LFS server domains to the allowlist. See [references/firewall.md](references/firewall.md) for the domain table.

### Phase 6: Task Runner Integration

**Wrap the container launch** in a task runner recipe so the entry point is `just dev-shell` (or equivalent).

See [references/task-runner.md](references/task-runner.md) for detection logic, recipe structure, and the `--firewall` flag for firewalled autonomous mode.

Key principles:
- Detect existing task runner (justfile, Makefile, Taskfile.yml, package.json)
- Ask user before adding recipes
- Conditional mounts (only mount configs that exist on the host and are supported by the chosen path)
- TTY auto-detection
- `--firewall` flag for opt-in firewall mode
- Single recipe, not separate isolated/full variants

If Docker support was enabled in Phase 1, add a conditional Docker socket mount with `--group-add` to the recipe. See [references/docker-support.md](references/docker-support.md).

If Kubernetes tooling was detected in Phase 1, add a conditional `~/.kube` mount and `KUBECONFIG` env var to the task runner recipe. See [references/task-runner.md](references/task-runner.md) for the recipe additions.

If the shared-auth path was selected in Phase 1, add conditional host mounts or env forwarding for GitHub auth, SSH agent access, and any confirmed local config paths. Keep all mounts explicit and optional.

### Phase 7: Testing

Run verifications inside the built container using the task runner recipe from Phase 6 (for example `just dev-shell <command>`). If no task runner was configured, use `docker run` directly.

**Verification checklist:**
- [ ] All language toolchain commands work (compiler, package manager)
- [ ] `whoami` resolves (entrypoint injected passwd entry for arbitrary UID)
- [ ] `git status` works with arbitrary UID
- [ ] PID 1 is `tini`/`docker-init` (check `cat /proc/1/cmdline`)
- [ ] Git identity resolves from read-only `.gitconfig` (`git config user.name`)
- [ ] `.gitconfig` is not writable (`git config --global user.name test` should fail)
- [ ] SSH agent is accessible (`ssh-add -l` lists keys)
- [ ] SSH-based git operations work (`git ls-remote` or `ssh -T git@github.com`)
- [ ] `gh --version` works
- [ ] `gh auth status` succeeds when shared auth or explicit container login is configured
- [ ] Any project-specific build commands succeed
- [ ] Ecosystem dev tools work, if installed (for example `cargo clippy --version`, `ruff --version`, `golangci-lint --version`)

**Git LFS verification (Git LFS only):**
- [ ] `git lfs version` — git-lfs is installed and on PATH
- [ ] `git lfs env` — LFS is configured correctly (shows system-level install)
- [ ] If firewall enabled: `git lfs pull` succeeds (LFS server domains are reachable)

**Kubernetes verification (Kubernetes tooling only):**
- [ ] `kubectl version --client` — kubectl is installed and on PATH
- [ ] `helm version` — Helm is installed (if detected)
- [ ] `helmfile --version` — Helmfile is installed (if detected)
- [ ] `echo $KUBECONFIG` — env var is set and points to a readable file
- [ ] `kubectl config view` — kubeconfig is loaded and contexts are visible

**Docker verification (Docker support only):**
- [ ] `docker version` — CLI installed, daemon reachable via socket
- [ ] `docker compose version` — compose plugin installed
- [ ] `docker buildx version` — buildx plugin installed
- [ ] `docker info` — full daemon connectivity
- [ ] `docker run --rm hello-world` — end-to-end pull + run + cleanup
- [ ] `docker build -t test-build - <<< 'FROM alpine' && docker rmi test-build` — can build images
- [ ] If firewall enabled: `docker pull alpine` succeeds (Docker Hub allowlisted)

**IDE attach verification (when using a remote IDE workflow):**
- [ ] The chosen editor can reopen the repository in the container
- [ ] GitHub-authenticated development features still work after attach
- [ ] Any required forwarded service ports are reachable from the host

**Firewall verification (Phase 5 only):**

Run with the firewall flag (for example `just dev-shell --firewall <command>`):
- [ ] `curl https://example.com` is rejected (firewall blocks unlisted domains)
- [ ] `gh auth status` still succeeds when GitHub domains are allowlisted
- [ ] `ssh -T git@github.com` succeeds (GitHub SSH is allowed)
- [ ] `firewall-list` shows resolved IPs
- [ ] `whoami` still resolves (gosu dropped to correct UID)
- [ ] `id -u` matches host UID (privilege drop worked)

## Reference

Before generating any file, consult the relevant reference for detailed patterns and code blocks:

- **[references/dockerfile.md](references/dockerfile.md)** — Dockerfile patterns, layer ordering, GitHub tooling, entrypoint
- **[references/devcontainer-json.md](references/devcontainer-json.md)** — devcontainer.json for shared-auth and isolated modes
- **[references/firewall.md](references/firewall.md)** — Network firewall: allowlist, script, Dockerfile/entrypoint additions
- **[references/task-runner.md](references/task-runner.md)** — Task runner recipe with `--firewall` flag
- **[references/common-mistakes.md](references/common-mistakes.md)** — Common mistakes and red flags to avoid
- **[references/docker-support.md](references/docker-support.md)** — Docker CLI + Compose: detection signals, Dockerfile layer, socket GID handling, firewall domains
- **[references/dev-tools.md](references/dev-tools.md)** — Ecosystem dev tools: detection signals, install scope rules, Dockerfile patterns
