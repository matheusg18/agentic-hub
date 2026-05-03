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

Keep the Docker tooling in the normal global CLI install layer alongside other host-facing tools. Do not add Docker daemon packages or services inside the devcontainer.

Use inclusion rules, not guesses:

- add the Compose plugin when the repository uses `docker compose` or checked-in Compose files
- add buildx only when the repository builds images or invokes `docker buildx`
- skip both when the repo only needs unrelated container references in docs or CI metadata

If Docker support is enabled, the image still stays client-only. The host daemon comes from `/var/run/docker.sock`; the container should never run a daemon of its own.

## Dockerfile additions

Add Docker support in the same global CLI layer as the other host-facing tools, but keep it client-only.

On Debian/Ubuntu-style bases, add the Docker APT repository first; the package names below come from that repository. For other base images, use the equivalent client-only package source for that distro.

- `docker-ce-cli` for the CLI
- `docker-compose-plugin` when the repository uses `docker compose` or checked-in Compose files
- `docker-buildx-plugin` when the repository builds images or invokes `docker buildx`

Do **not** install daemon packages or services such as `docker-ce`, `dockerd`, or `containerd`. The devcontainer should talk to the host daemon through `/var/run/docker.sock`, not start its own daemon.

Keep the socket-GID path writable during the root phase of the entrypoint. If the runtime needs to append a missing socket GID to `/etc/group`, do not replace that file with a read-only mount or otherwise block the root user from updating it before privilege drop.

```dockerfile
# -- Docker client (optional, Docker support only) ------------------------------
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl gnupg \
    && install -m 0755 -d /etc/apt/keyrings \
    && . /etc/os-release \
    && curl -fsSL https://download.docker.com/linux/${ID}/gpg \
        | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
    && chmod a+r /etc/apt/keyrings/docker.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/${ID} ${VERSION_CODENAME} stable" \
        > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        docker-ce-cli \
        docker-compose-plugin \
        docker-buildx-plugin \
    && rm -rf /var/lib/apt/lists/*
```

Omit the Compose and buildx packages when the inclusion rules above do not apply. Keep Docker support optional and add only the client pieces the repository actually needs.

## Socket GID mapping

Handle the Docker socket group safely.

- Only touch the socket path when `/var/run/docker.sock` exists
- Only perform GID mapping when the entrypoint is running as root
- Look up the socket GID at runtime, create a matching group only when needed, and add the target login user to it before dropping privileges
- Keep the logic conditional so containers still work without Docker access when the socket is absent

If the entrypoint needs to append a missing socket GID to `/etc/group`, that file must be writable during the root phase of the entrypoint. The write happens before dropping privileges, so the container can safely map the socket group without running the whole session as root.

For IDE-managed attach flows, make the entrypoint do the mapping and continue without failing the whole container when Docker is unavailable:

```bash
configure_docker_socket_access() {
  local socket_path="/var/run/docker.sock"
  local socket_gid socket_group

  [ -S "$socket_path" ] || return 0

  socket_gid="$(stat -c '%g' "$socket_path" 2>/dev/null || true)"
  if [ -z "$socket_gid" ]; then
    echo "warning: skipping Docker socket GID mapping; could not read ${socket_path}" >&2
    return 0
  fi

  socket_group="$(getent group "$socket_gid" | cut -d: -f1 || true)"
  if [ -z "$socket_group" ]; then
    socket_group="docker-host-${socket_gid}"
    echo "${socket_group}:x:${socket_gid}:" >> /etc/group
  fi

  if command -v gpasswd >/dev/null 2>&1; then
    if ! id -nG "$target_user" | tr ' ' '\n' | grep -Fx "$socket_group" >/dev/null; then
      gpasswd -a "$target_user" "$socket_group" >/dev/null 2>&1
    fi
  else
    echo "warning: gpasswd not available; skipping Docker socket GID mapping" >&2
  fi
}
```

Run that helper after the target passwd entry exists and before `gosu` drops privileges. This is the IDE-managed path; the separate task-runner path should still use [`--group-add`](task-runner.md).

Use the Dockerfile/entrypoint path for the IDE-managed launch flow. For the task-runner launch path, follow the dedicated guidance in [references/task-runner.md](task-runner.md) so the `--group-add` handling stays consistent.

## Copilot CLI URL permissions

If the chosen workflow uses Copilot CLI URL allow/deny rules, scan `FROM` directives in Dockerfiles and `image:` fields in Compose files, then extract the literal hostname from each registry reference.

Use only exact hostnames that the repository actually references. Typical examples:

- `registry-1.docker.io`
- `auth.docker.io`
- `production.cloudflare.docker.com`
- `ghcr.io`
- `registry.gitlab.com`
- `gcr.io`
- `us-central1-docker.pkg.dev`
- `123456789012.dkr.ecr.us-east-1.amazonaws.com`

Those hosts belong in URL permission guidance only when Copilot CLI is actually expected to fetch them during the workflow.
