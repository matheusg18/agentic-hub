# Getting Started

Use this reference before committing to a specific `.devcontainer/` shape. It covers the general devcontainer decisions that should happen before the GitHub-authenticated hardening flow in `SKILL.md`.

## Source of truth

Before recommending structure or "best practices", check the current official sources that match the chosen workflow:

- Dev Container spec and `devcontainer.json` reference
- official Templates catalog
- official Features catalog
- editor workflow docs for the chosen host when behavior differs
- current behavior and included tooling of the base image you plan to use

Do not rely on memory for lifecycle semantics, Feature availability, Template coverage, or base-image contents.

## Choose the right starting point

| Starting point | Use when | Avoid when |
|---|---|---|
| **Template** | Greenfield setup, common language/runtime stack, team wants a known-good baseline quickly | The repository already has good Docker/Compose assets worth extending |
| **Image** | Single-container workflow, very light customization, base image already matches the toolchain | You need durable OS packages, trust store changes, or stable global tooling |
| **Dockerfile** | You need durable image changes such as OS packages, CA/proxy trust, stable CLIs, or explicit non-root defaults | The real workflow depends on multiple side services |
| **Docker Compose** | Local development needs DB/cache/search/vector/queue side services | The workflow is really single-container |
| **Extend existing Compose** | The repository already has working checked-in Compose assets that model local development | Existing Compose is broken or production-only in ways that actively hurt local development |

**Practical default:** prefer **Template** for greenfield, **extend existing Compose** for mature multi-service repos, and **Dockerfile** for single-service custom setups.

## Official accelerators

- Use official/community **Templates** before hand-rolling a greenfield baseline.
- Use official **Features** before writing custom install logic for common tooling.
- Reuse existing repository Dockerfiles or Compose assets when they already match the development workflow.

If the environment has fragile network access, corp proxy constraints, or credentials that make build-time installs brittle, a runtime bootstrap may be safer than a Feature. Document that tradeoff instead of cargo-culting the Feature path.

## File-by-file responsibilities

| File | Responsibility |
|---|---|
| `.devcontainer/devcontainer.json` | Top-level container wiring, lifecycle hooks, forwarded ports, editor ergonomics |
| `.devcontainer/Dockerfile` | Durable image content: base image, CA trust, OS packages, stable tooling |
| `.devcontainer/docker-compose.yml` | Workspace plus side-service orchestration, healthchecks, internal networking |
| `.devcontainer/initialize.sh` | Host-side prep before build or attach |
| `.devcontainer/post-create.sh` | Repo-aware bootstrap after the workspace is mounted |
| `.devcontainer/devcontainer.env.example` | Non-secret local env contract for hostnames, ports, and expected variables |
| `.devcontainer/README.md` | Operational source of truth for lifecycle, troubleshooting, stack notes, and onboarding |

Keep the root `README.md` short when the setup is non-trivial; link to `.devcontainer/README.md` for detailed behavior.
