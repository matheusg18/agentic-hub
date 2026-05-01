# Ecosystem Dev Tools

Use this reference when Phase 1 signals that the repository expects local dev tools in the devcontainer.

## What to inspect

Scan both CI and project metadata:

- **CI commands**: GitHub Actions workflows, Makefile targets, justfiles, Taskfiles, shell scripts, and other automation entrypoints.
- **Dependency manifests**: `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, and similar project-managed dependency files.
- **Tool config files**: `ruff.toml`, `.ruff.toml`, `.golangci.yml`, `.prettierrc`, `biome.json`, `helmfile.yaml`, `Chart.yaml`, `kustomization.yaml`, and other ecosystem-standard config files.

Prefer the tool name and version from CI when the repo is explicit. If the repo only signals a tool through config or dependency manifests, use that as the source of truth.

## Install scope

A dev tool must be installed in one of two places:

- **Project-managed**: install through the repo's own package manager or toolchain flow.
- **Globally in the devcontainer**: install in the Dockerfile when the repo uses the tool in CI but does not manage it in project dependencies.

Rules:

1. If the tool appears in project dependencies or the standard dev install flow, do **not** add a global install.
2. If the tool is a language toolchain component, keep it with the toolchain rather than the package manager.
3. If CI uses the tool and the repo does not manage it, install it globally so local and CI behavior match.

## Dockerfile patterns by ecosystem

Use explicit `ARG` values plus Renovate annotations for every versioned global install.

### Python tools

- Keep `pipx`/`uv tool install`/`python -m pip install` layers separate from apt packages.
- Use one layer for Python tool installers after `ENV HOME` / `ENV PATH` are set.
- Prefer project-managed installs from `pyproject.toml` or requirements files; only global-install tools like `ruff` or `mypy` when CI calls them outside the project environment.

Example pattern:

```dockerfile
# renovate: datasource=pypi depName=ruff
ARG RUFF_VERSION="0.6.9"
RUN pipx install "ruff==${RUFF_VERSION}"
```

### JavaScript / TypeScript tools

- If the repo already declares the tool in `package.json`, let the package manager handle it.
- For genuinely global JS tooling, keep npm-based CLI installs in their own layer after Node is available.
- Preserve npm hardening guidance from `references/dockerfile.md` when using npm in the image.

Example pattern:

```dockerfile
# renovate: datasource=npm depName=pnpm
ARG PNPM_VERSION="9.12.3"
RUN npm install -g "pnpm@${PNPM_VERSION}"
```

### Go tools

- Install Go-based CLIs with `go install` after the Go toolchain is on `PATH`.
- Keep Go CLI installs in a dedicated layer so tool bumps do not invalidate unrelated system packages.
- Use the module path plus version in the install command.

Example pattern:

```dockerfile
# renovate: datasource=go depName=honnef.co/go/tools
ARG STATICCHECK_VERSION="v2024.1.1"
RUN go install honnef.co/go/tools/cmd/staticcheck@${STATICCHECK_VERSION}
```

### Rust tools

- Prefer `cargo install --locked` for global Rust CLIs the repo does not vendor.
- Keep Rust CLI installs in a dedicated layer after Rust toolchain setup.
- Use `--locked` to preserve upstream lockfile behavior where supported.

Example pattern:

```dockerfile
# renovate: datasource=crate depName=cargo-nextest
ARG CARGO_NEXTEST_VERSION="0.9.81"
RUN cargo install --locked cargo-nextest --version "${CARGO_NEXTEST_VERSION}"
```

### Kubernetes / platform tooling

- Install release binaries such as `kubectl`, `helm`, `helmfile`, or `kustomize` in explicit CLI layers.
- Keep them out of language-package layers because they are repo-wide infrastructure tools.
- Reuse the GitHub-release download style from `references/dockerfile.md` when the repo does not manage them another way.

## Renovate annotations

Every versioned global tool install should follow the same structure:

1. `# renovate: ...` comment immediately above the `ARG`
2. `ARG TOOL_VERSION="..."`
3. Install command referencing that `ARG`

Pick the datasource that matches the upstream source (`github-releases`, `npm`, `pypi`, `go`, `crate`, and so on). This keeps tool updates visible and reviewable.

## Layer placement

Fit ecosystem tool installs into the broader Dockerfile order from `references/dockerfile.md`:

1. Base image
2. `ENV HOME` / `ENV PATH`
3. System packages
4. Versioned global CLI installs (`gh`, repo-wide tools, Kubernetes binaries)
5. Ecosystem-specific global tool layers (Python, JS/TS, Go, Rust) only when the repo does not manage them
6. Git/SSH configuration
7. Non-root user creation
8. Entrypoint and runtime permission setup

This keeps path-sensitive installers working, reduces rebuild churn, and makes repo-wide tools easy to audit.

## Practical examples

- **Python**: `ruff`, `mypy`, `black`, `flake8`, `pylint`, `isort`, `pytest`
- **Go**: `golangci-lint`, `staticcheck`, `gofumpt`
- **JS/TS**: `eslint`, `prettier`, `biome`, `vitest`, `jest`
- **Kubernetes**: `kubectl`, `helm`, `helmfile`, `kustomize`

Use repository evidence to decide which of these are relevant. Do not install a tool just because it is common in the ecosystem.

## GitHub Copilot-specific guidance

Keep the guidance aligned to GitHub repositories and GitHub Actions terminology. Limit GitHub Copilot guidance to supported IDE attach behavior, authentication compatibility, and repository-evidenced tooling needs. Do **not** invent a separate GitHub Copilot workflow surface, installer, or detection namespace inside the container.

## Verification

When you add a global tool install, make sure the Phase 7 checklist includes a `--version` check or equivalent command so the tool is confirmed on `PATH`.
