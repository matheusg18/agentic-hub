# Task Runner Reference

Wrap the devcontainer launch in one stable local entry point so users do not need to remember a long `docker run` command.

Prefer a single recipe such as `just dev-shell`, then adapt the same shape to `make dev-shell`, `task dev-shell`, or a package manager script when the repository already uses that tool.

## 1) Detect the existing task runner

Check the project root for:

| File | Tool |
|------|------|
| `justfile` / `.justfile` | `just` |
| `Makefile` / `GNUmakefile` | `make` |
| `Taskfile.yml` | `task` |
| `package.json` scripts | npm / yarn / pnpm |

If a task runner already exists, ask before adding recipes. Keep the repository’s naming conventions and do not introduce a second parallel launcher unless the user explicitly wants it.

## 2) What the recipe should do

The recipe should:

1. build the devcontainer image when stale
2. launch an interactive shell by default
3. accept pass-through arguments for specific commands
4. add conditional mounts only when the host path exists
5. optionally enable Docker socket access
6. detect TTY presence so scripts do not fail in CI or non-interactive shells

For staleness checks, prefer the runner’s native file tracking when it exists. If the runner cannot do that cleanly, use a simple sentinel file or always rebuild and rely on Docker layer caching.

## 3) Stable entry point

Use one command as the standard local entry point:

- `just dev-shell`

Equivalent names are fine in other runners, but keep the command short and memorable.

## 4) Mount rules

Mount only host files and directories that actually exist.

Good candidates:

- `~/.gitconfig` when the file exists
- `~/.config/gh` when GitHub CLI auth should be reused
- `~/.ssh` only when the repository explicitly requires direct key files
- `~/.kube` only when Kubernetes tooling is enabled

Do not rely on hidden tool-specific directories that are not part of the repository’s agreed auth flow. For GitHub auth, prefer explicit reuse of `~/.config/gh` and `~/.gitconfig` rather than inventing new state locations.

## 5) Optional Docker socket support

If Docker support was enabled in Phase 1, gate the socket check on that choice; only then, if `/var/run/docker.sock` exists, mount it explicitly and add the socket GID to the container’s supplementary groups.

Keep this conditional. If Docker support was not selected or the socket is missing, run without Docker access.

## 6) Example `just` shape

```just
[positional-arguments]
dev-shell *args:
    #!/usr/bin/env bash
    set -euo pipefail

    run_args=(
        --rm
        -v "$(pwd):$(pwd)"
        -w "$(pwd)"
        -e "DEVCONTAINER_UID=$(id -u)"
        -e "DEVCONTAINER_GID=$(id -g)"
    )
    docker_support_enabled=false # set from the Phase 1 Docker support choice
    container_args=()

    if [[ -t 0 ]]; then
        run_args+=(-it)
    else
        run_args+=(-i)
    fi

    [[ -f "$HOME/.gitconfig" ]] && run_args+=(-v "$HOME/.gitconfig:/tmp/home/.gitconfig:ro")
    [[ -d "$HOME/.config/gh" ]] && run_args+=(-v "$HOME/.config/gh:/tmp/home/.config/gh:ro")

    if [[ "$docker_support_enabled" == true && -S /var/run/docker.sock ]]; then
        run_args+=(-v /var/run/docker.sock:/var/run/docker.sock)
        docker_sock_gid="$(
            if stat -c '%g' /var/run/docker.sock >/dev/null 2>&1; then
                stat -c '%g' /var/run/docker.sock
            else
                stat -f '%g' /var/run/docker.sock
            fi
        )"
        if [[ -z "$docker_sock_gid" ]]; then
            echo "error: could not determine Docker socket GID" >&2
            exit 1
        fi
        run_args+=(--group-add "$docker_sock_gid")
    fi

    container_args+=("$@")

    if [[ ${#container_args[@]} -eq 0 ]]; then
        exec docker run "${run_args[@]}" <image> bash
    else
        exec docker run "${run_args[@]}" <image> "${container_args[@]}"
    fi
```

## 7) Key rules

- Keep the entry point short and local
- Prefer one recipe over separate isolated/full variants
- Mount only existing host paths
- Reuse GitHub auth explicitly and safely
- Keep Docker socket support gated on the Phase 1 choice
- Let the entrypoint handle UID/GID remapping; do not bypass it with `--user` in this pattern
