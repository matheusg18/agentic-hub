# Dockerfile Reference

Create `.devcontainer/Dockerfile` with a GitHub-first baseline that stays compatible with GitHub-authenticated development and GitHub Copilot-capable IDE attach flows.

## 1) Prefer reusing CI and job container images

If the repository already builds a suitable image in GitHub Actions, reuse that image as the devcontainer base.

Look for:
- `.github/workflows/*.yml` jobs that build/publish a toolchain image
- jobs that already run inside a `container:` image
- GHCR images referenced by workflow outputs or deployment steps
- repository Dockerfiles that already encode the project toolchain

Prefer a multi-stage `FROM` of the workflow-built image over rebuilding the same toolchain in the devcontainer. If a workflow uses `container:`, reuse that same image (tag + digest) so local devcontainer runs and CI execute against the same filesystem and toolchain baseline. If no reusable CI image exists, fall back to an official language/runtime image.

Always pin the base image with both tag and digest:

```dockerfile
FROM ghcr.io/org/project/toolchain:1.2.3@sha256:...
```

## 2) Pin every versioned dependency

Every versioned install should be explicit and Renovate-friendly:

```dockerfile
# renovate: datasource=github-releases depName=cli/cli
ARG GH_VERSION="2.60.1"
```

Use the appropriate datasource for the source being installed. Keep the version in an `ARG` so Renovate can update it without editing the install logic.

## 3) Preserve npm hardening when Node is involved

If the project uses npm, keep installs reproducible and minimize supply-chain surface:

- prefer `npm ci` over `npm install` in build steps
- set `npm config set ignore-scripts true` during dependency installation unless the project explicitly requires lifecycle scripts
- run any required package scripts in explicit, reviewed steps rather than during the dependency fetch phase

## 4) Set `HOME` and `PATH` early

Set `HOME` and `PATH` before any install steps so every installer targets the intended writable location.

```dockerfile
ENV HOME=/tmp/home
ENV PATH="${HOME}/.local/bin:${PATH}"
```

Rules:
- set these before `apt`, `curl | bash`, package managers, or CLI installers
- keep them near the top of the Dockerfile
- do not rely on later `ENV` statements to fix install paths

## 5) Do not set `WORKDIR`

Never set `WORKDIR` in the devcontainer image.

The repository is mounted at the host-native absolute path and may be a git worktree. A fixed image `WORKDIR` breaks that model and can conflict with absolute paths stored in `.git`.

## 6) Keep GitHub tooling available

Install the baseline tools needed for GitHub-authenticated development:
- `gh`
- `git`
- `openssh-client`
- `ca-certificates`

On Debian/Ubuntu-style bases, keep the system package layer explicit and include `gosu` there when you need runtime privilege drop:

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        git \
        gosu \
        openssh-client \
    && rm -rf /var/lib/apt/lists/*
```

Adapt the package-manager command to the chosen base image, but keep `gosu` installed before the entrypoint depends on it.

A typical GitHub CLI install pattern is:

```dockerfile
# renovate: datasource=github-releases depName=cli/cli
ARG GH_VERSION="2.60.1"
RUN arch="$(dpkg --print-architecture)" \
    && curl -fsSL "https://github.com/cli/cli/releases/download/v${GH_VERSION}/gh_${GH_VERSION}_linux_${arch}.deb" -o /tmp/gh.deb \
    && dpkg -i /tmp/gh.deb \
    && rm /tmp/gh.deb \
    && gh --version
```

If the build runs `gh` during image creation, clean up any root-owned config directory it creates and recreate it world-writable for arbitrary UID support:

```dockerfile
ENV GH_CONFIG_DIR=/tmp/gh-config
RUN rm -rf /tmp/gh-config && mkdir -m 1777 /tmp/gh-config
```

Do not describe or invent a standalone GitHub Copilot installer for the container. Copilot is an IDE-side capability, not a Dockerfile-managed binary here.

If Phase 1 detects Git LFS usage, extend the same system package layer and initialize it system-wide:

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        git \
        git-lfs \
        gosu \
        openssh-client \
    && git lfs install --system \
    && rm -rf /var/lib/apt/lists/* \
    && git lfs version
```

Keep `git-lfs` in the Dockerfile only when the repository actually uses LFS. If `.lfsconfig` points at a custom LFS server, note that hostname for any later Copilot CLI URL permission guidance.

## 7) Preserve hardened permissions

Use sticky-bit permissions for shared writable paths:

- `chmod 1777`, not `chmod 777`
- apply it to directories that must be writable by multiple UIDs
- avoid broad world-writable permissions on anything else

If you need a writable home or cache path for runtime tools, make it explicitly sticky-bit world-writable and keep the scope narrow.

## 8) Support arbitrary UID entrypoints

Create exactly one non-root user in the image, then drop privileges at runtime when the container starts as root.

```dockerfile
RUN groupadd --gid 1000 dev \
 && useradd --uid 1000 --gid 1000 --no-create-home --home-dir /tmp/home --shell /bin/bash dev
```

Use an entrypoint that:
- resolves the workspace owner UID/GID when possible
- copies or writes any required passwd entries
- drops from root to the target UID with `gosu` when needed

Keep the privilege-drop path compatible with both IDE-managed UID remapping and direct container runs.

A practical `.devcontainer/entrypoint.sh` template for arbitrary-UID support:

```bash
#!/usr/bin/env bash
set -euo pipefail

DEFAULT_USER="${DEFAULT_USER:-dev}"
DEFAULT_UID="${DEFAULT_UID:-1000}"
DEFAULT_GID="${DEFAULT_GID:-1000}"
DEFAULT_HOME="${DEFAULT_HOME:-${HOME:-/tmp/home}}"
WORKSPACE_DIR="${LOCAL_WORKSPACE_FOLDER:-${WORKSPACE_FOLDER:-$PWD}}"

target_uid="${DEFAULT_UID}"
target_gid="${DEFAULT_GID}"
target_user="${DEFAULT_USER}"

if [ -d "${WORKSPACE_DIR}" ]; then
  workspace_uid="$(stat -c '%u' "${WORKSPACE_DIR}")"
  workspace_gid="$(stat -c '%g' "${WORKSPACE_DIR}")"

  if [ "${workspace_uid}" != "0" ]; then
    target_uid="${workspace_uid}"
    target_gid="${workspace_gid}"
  fi
fi

if [ "$(id -u)" != "0" ]; then
  exec "$@"
fi

if ! getent group "${target_gid}" >/dev/null; then
  echo "devcontainer-${target_gid}:x:${target_gid}:" >> /etc/group
fi

if existing_passwd="$(getent passwd "${target_uid}")"; then
  target_user="${existing_passwd%%:*}"
else
  if getent passwd "${DEFAULT_USER}" >/dev/null; then
    target_user="devcontainer-${target_uid}"
  fi
  echo "${target_user}:x:${target_uid}:${target_gid}:Dev Container User:${DEFAULT_HOME}:/bin/bash" >> /etc/passwd
fi

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

mkdir -p "${DEFAULT_HOME}"
chown "${target_uid}:${target_gid}" "${DEFAULT_HOME}"

configure_docker_socket_access

exec gosu "${target_user}:${target_gid}" "$@"
```

Pair it with the Dockerfile so the runtime contract is explicit:

```dockerfile
COPY .devcontainer/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod 0755 /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["sleep", "infinity"]
```

Notes:
- use the mounted workspace owner when available so git writes land with the host UID/GID
- keep the baked-in `dev` user as the fallback for direct container runs and hosts that already remap the UID
- only append passwd/group entries when they do not already exist
- when Docker support is enabled, map the socket GID in the root phase and add the target login user to that group before `gosu`
- `gosu` must already be installed in the image before this entrypoint runs
- Debian/Ubuntu bases usually provide `gpasswd`; on minimal bases, install the equivalent account-management tool if the entrypoint needs to edit group membership

## 9) Keep git and SSH ready

Configure git and SSH for non-interactive use:

```dockerfile
RUN git config --system safe.directory '*'
RUN mkdir -p /etc/ssh \
    && ssh-keyscan -t ecdsa,rsa,ed25519 github.com >> /etc/ssh/ssh_known_hosts 2>/dev/null
```

If the project uses another GitHub-hosted Git remote, add that host too. Keep `github.com` present whenever `gh` or GitHub-based auth is expected.

## 10) Practical ordering

Recommended layer order:
1. Base image
2. `ENV HOME` / `ENV PATH`
3. System packages
4. Versioned CLI installs (`gh`, project-wide tools)
5. Git/SSH configuration
6. Non-root user creation
7. Entrypoint and runtime permission setup

This keeps installers using the right paths, reduces rebuild churn, and preserves worktree compatibility.

## 11) What not to do

- Do not add unsupported GitHub Copilot install claims
- Do not bake editor-specific auth secrets into the image
- Do not use `WORKDIR`
- Do not pin floating images without digests
- Do not use `chmod 777` as a shortcut
