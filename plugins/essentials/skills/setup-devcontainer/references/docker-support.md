# Docker Support (Optional)

Docker CLI + Compose support is an optional cross-cutting feature for GitHub-hosted development workflows. Enable it only when the project actually needs to build images, run Compose stacks, or use Docker-aware test tooling, or when the user explicitly asks for it.

Do not install a Docker daemon in the devcontainer. The host daemon should be accessed through the Docker socket when support is enabled.

## Detection signals

Scan the repository during Phase 1 to decide whether Docker support should be offered.

**Strong signals**:
- `docker-compose.yml`, `docker-compose.yaml`, `compose.yml`, `compose.yaml`
- `Dockerfile` or `Containerfile` outside `.devcontainer/`
- task runner commands that invoke `docker build`, `docker compose`, `docker run`, or similar
  - examples: `Makefile`, `justfile`, `Taskfile.yml`, `package.json` scripts
- Docker-aware test dependencies such as Testcontainers
  - Java: `org.testcontainers`
  - Python: `testcontainers`
  - Node: `testcontainers`
  - Go: `testcontainers-go`
  - Rust: `testcontainers`

**Weak signals**:
- `.dockerignore`
- CI workflows that build or publish container images
- `DOCKER_HOST` or similar Docker env vars in repo config

Present Docker support as an option when strong signals exist. If nothing points to Docker, keep it optional and ask before adding it.

## What to install

When Docker support is enabled, install only the client-side tooling needed by the workflow:
- Docker CLI
- Compose plugin
- buildx plugin when image builds are part of the workflow

Install these only when needed or explicitly requested. Do not add them by default just because the repository uses GitHub Copilot or a devcontainer.

## Socket GID mapping

Handle the Docker socket group safely.

- Only touch the socket path when `/var/run/docker.sock` exists
- Only perform GID mapping when the entrypoint is running as root
- Look up the socket GID at runtime and add the user to that group only if it does not already exist
- Keep the logic conditional so containers still work without Docker access when the socket is absent

Use the Dockerfile/entrypoint path for the IDE-managed launch flow. For the task-runner launch path, follow the dedicated guidance in [references/task-runner.md](task-runner.md) so the `--group-add` handling stays consistent.

## Firewall allowlist

Only add Docker registry domains to the firewall allowlist when Docker support is enabled **and** the firewall feature is enabled.

Scan `FROM` directives in Dockerfiles and `image:` fields in Compose files to identify registries.

Common registry domains:
- Docker Hub: `registry-1.docker.io`, `auth.docker.io`, `production.cloudflare.docker.com`
- GHCR: `ghcr.io`
- GitLab: `registry.gitlab.com`
- GCR: `gcr.io`
- GAR: `*-docker.pkg.dev`
- ECR: `*.dkr.ecr.*.amazonaws.com`

If the project uses a private registry, only add the exact hostnames the repository actually references.
