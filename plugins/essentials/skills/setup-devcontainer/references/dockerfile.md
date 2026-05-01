# Phase 2: Dockerfile

Create `.devcontainer/Dockerfile` with a GitHub-first baseline that stays compatible with GitHub-authenticated development and GitHub Copilot-capable IDE attach flows.

## 1) Prefer reusing CI images

If the repository already builds a suitable image in GitHub Actions, reuse that image as the devcontainer base.

Look for:
- `.github/workflows/*.yml` jobs that build/publish a toolchain image
- GHCR images referenced by workflow outputs or deployment steps
- repository Dockerfiles that already encode the project toolchain

Prefer a multi-stage `FROM` of the workflow-built image over rebuilding the same toolchain in the devcontainer. If no reusable CI image exists, fall back to an official language/runtime image.

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

## 3) Set `HOME` and `PATH` early

Set `HOME` and `PATH` before any install steps so every installer targets the intended writable location.

```dockerfile
ENV HOME=/tmp/home
ENV PATH="${HOME}/.local/bin:${PATH}"
```

Rules:
- set these before `apt`, `curl | bash`, package managers, or CLI installers
- keep them near the top of the Dockerfile
- do not rely on later `ENV` statements to fix install paths

## 4) Do not set `WORKDIR`

Never set `WORKDIR` in the devcontainer image.

The repository is mounted at the host-native absolute path and may be a git worktree. A fixed image `WORKDIR` breaks that model and can conflict with absolute paths stored in `.git`.

## 5) Keep GitHub tooling available

Install the baseline tools needed for GitHub-authenticated development:
- `gh`
- `git`
- `openssh-client`
- `ca-certificates`

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

## 6) Preserve hardened permissions

Use sticky-bit permissions for shared writable paths:

- `chmod 1777`, not `chmod 777`
- apply it to directories that must be writable by multiple UIDs
- avoid broad world-writable permissions on anything else

If you need a writable home or cache path for runtime tools, make it explicitly sticky-bit world-writable and keep the scope narrow.

## 7) Support arbitrary UID entrypoints

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

## 8) Keep git and SSH ready

Configure git and SSH for non-interactive use:

```dockerfile
RUN git config --system safe.directory '*'
RUN mkdir -p /etc/ssh \
    && ssh-keyscan -t ecdsa,rsa,ed25519 github.com >> /etc/ssh/ssh_known_hosts 2>/dev/null
```

If the project uses another GitHub-hosted Git remote, add that host too. Keep `github.com` present whenever `gh` or GitHub-based auth is expected.

## 9) Practical ordering

Recommended layer order:
1. Base image
2. `ENV HOME` / `ENV PATH`
3. System packages
4. Versioned CLI installs (`gh`, project-wide tools)
5. Git/SSH configuration
6. Non-root user creation
7. Entrypoint and runtime permission setup

This keeps installers using the right paths, reduces rebuild churn, and preserves worktree compatibility.

## 10) What not to do

- Do not add unsupported GitHub Copilot install claims
- Do not bake editor-specific auth secrets into the image
- Do not use `WORKDIR`
- Do not pin floating images without digests
- Do not use `chmod 777` as a shortcut
