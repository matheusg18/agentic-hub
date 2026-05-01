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
6. optionally enable `--firewall` mode
7. detect TTY presence so scripts do not fail in CI or non-interactive shells

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

If Docker support is enabled and `/var/run/docker.sock` exists, mount it explicitly and add the socket GID to the container’s supplementary groups.

Keep this conditional. If the socket is missing, run without Docker access.

## 6) Optional firewall mode

If the skill’s firewall phase was selected, expose a `--firewall` flag on the recipe.

- Without `--firewall`, run in normal development mode
- With `--firewall`, start the firewalled path and pass the flag through to the container entrypoint

Do not make firewall mode the default.

## 7) Example `just` shape

```just
[positional-arguments]
dev-shell *args:
    #!/usr/bin/env bash
    set -euo pipefail

    run_args=(
        --rm
        -v "$(pwd):$(pwd)"
        -w "$(pwd)"
    )

    if [[ -t 0 ]]; then
        run_args+=(-it)
    else
        run_args+=(-i)
    fi

    [[ -f "$HOME/.gitconfig" ]] && run_args+=(-v "$HOME/.gitconfig:/tmp/home/.gitconfig:ro")
    [[ -d "$HOME/.config/gh" ]] && run_args+=(-v "$HOME/.config/gh:/tmp/home/.config/gh")

    if [[ -S /var/run/docker.sock ]]; then
        run_args+=(-v /var/run/docker.sock:/var/run/docker.sock)
        run_args+=(--group-add "$(stat -c '%g' /var/run/docker.sock)")
    fi

    if [[ "${1:-}" == "--firewall" ]]; then
        shift
        run_args+=(--cap-add=NET_ADMIN --cap-add=NET_RAW)
    fi

    if [[ $# -eq 0 ]]; then
        exec docker run "${run_args[@]}" <image> bash
    else
        exec docker run "${run_args[@]}" <image> "$@"
    fi
```

## 8) Key rules

- Keep the entry point short and local
- Prefer one recipe over separate isolated/full variants
- Mount only existing host paths
- Reuse GitHub auth explicitly and safely
- Keep Docker and firewall support opt-in
