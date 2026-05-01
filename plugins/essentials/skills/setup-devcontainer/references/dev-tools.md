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

## Practical examples

- **Python**: `ruff`, `mypy`, `black`, `flake8`, `pylint`, `isort`, `pytest`
- **Go**: `golangci-lint`, `staticcheck`, `gofumpt`
- **JS/TS**: `eslint`, `prettier`, `biome`, `vitest`, `jest`
- **Kubernetes**: `kubectl`, `helm`, `helmfile`, `kustomize`

Use repository evidence to decide which of these are relevant. Do not install a tool just because it is common in the ecosystem.

## GitHub Copilot-specific guidance

Keep the guidance aligned to GitHub repositories and GitHub Actions terminology. If the repo uses GitHub Copilot workflows or related automation, treat them as GitHub-native tooling; do not introduce non-GitHub install or detection language.

## Verification

When you add a global tool install, make sure the Phase 7 checklist includes a `--version` check or equivalent command so the tool is confirmed on `PATH`.
