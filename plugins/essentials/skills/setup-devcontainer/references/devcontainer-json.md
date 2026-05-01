# Phase 3: devcontainer.json

**Ask the user:** should the container share host GitHub auth and local IDE settings, or start isolated?

Use the Dev Container spec for GitHub-authenticated development and GitHub Copilot-compatible IDE attach flows. Keep the configuration generic enough for VS Code, JetBrains, DevPod, and `devcontainer` CLI use.

## Path A: Shared host auth and settings

Use this when the host already has GitHub CLI auth, SSH agent access, and any local developer settings you want to reuse.

```json
{
  "name": "<project-name>",
  "build": { "dockerfile": "Dockerfile" },
  "init": true,
  "containerUser": "dev",
  "remoteUser": "dev",
  "updateRemoteUserUID": true,
  "workspaceFolder": "${localWorkspaceFolder}",
  "workspaceMount": "source=${localWorkspaceFolder},target=${localWorkspaceFolder},type=bind",
  "mounts": [
    "source=${localEnv:HOME}/.gitconfig,target=/tmp/home/.gitconfig,type=bind,readonly",
    "source=${localEnv:HOME}/.config/gh,target=/tmp/home/.config/gh,type=bind,readonly",
    "source=${localEnv:SSH_AUTH_SOCK},target=/tmp/ssh-agent.sock,type=bind"
  ],
  "remoteEnv": {
    "SSH_AUTH_SOCK": "/tmp/ssh-agent.sock",
    "GIT_SSH_COMMAND": "ssh -o UserKnownHostsFile=/etc/ssh/ssh_known_hosts",
    "COLORTERM": "${localEnv:COLORTERM}"
  },
  "containerEnv": {
    "DEVCONTAINER_WORKSPACE": "${localWorkspaceFolder}"
  }
}
```

Key points:
- Keep the project mounted at the same absolute path as on the host so git worktree paths keep working.
- Mount `~/.gitconfig` read-only so the container can read identity and aliases without mutating host config.
- Only mount `~/.config/gh` when the host path actually exists. Keep it read-only when reusing host auth state; if the host path does not exist, omit the mount and use the isolated path instead.
- Do not add any standalone tool-specific config mount. GitHub Copilot should use the IDE’s normal attach/auth flow, not a separate in-container tool path.
- SSH agent forwarding is enough; do not mount `~/.ssh` unless the repository explicitly needs a separate key file.

## Path B: Isolated container state

Use this for shared team containers, CI-like setups, or when host GitHub auth should not be reused.

```json
{
  "name": "<project-name>",
  "build": { "dockerfile": "Dockerfile" },
  "init": true,
  "containerUser": "dev",
  "remoteUser": "dev",
  "updateRemoteUserUID": true,
  "workspaceFolder": "${localWorkspaceFolder}",
  "workspaceMount": "source=${localWorkspaceFolder},target=${localWorkspaceFolder},type=bind",
  "mounts": [
    "source=${localEnv:HOME}/.gitconfig,target=/tmp/home/.gitconfig,type=bind,readonly",
    "source=gh-config-${devcontainerId},target=/tmp/home/.config/gh,type=volume",
    "source=${localEnv:SSH_AUTH_SOCK},target=/tmp/ssh-agent.sock,type=bind"
  ],
  "remoteEnv": {
    "SSH_AUTH_SOCK": "/tmp/ssh-agent.sock",
    "GIT_SSH_COMMAND": "ssh -o UserKnownHostsFile=/etc/ssh/ssh_known_hosts",
    "COLORTERM": "${localEnv:COLORTERM}"
  },
  "containerEnv": {
    "DEVCONTAINER_WORKSPACE": "${localWorkspaceFolder}"
  }
}
```

Key points:
- Use a named volume for `~/.config/gh` so GitHub CLI auth persists across rebuilds without depending on host state.
- Keep `~/.gitconfig` read-only in both modes; identity should be visible but not rewritten by the container.
- Do not invent host-side Copilot mounts or any other tool-specific config directories in isolated mode.

## Optional Docker socket support

If Docker support was enabled in Phase 1, add a Docker socket bind mount only when the host socket exists:

```json
"source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind"
```

Notes:
- Keep this optional and explicit.
- Do not add `runArgs` for socket GID handling here; the Dockerfile/entrypoint path should handle that.
- If the socket is missing, skip the mount and keep the container Docker-free.

## Optional firewall mode

If the firewall mode is selected elsewhere in the skill, keep the devcontainer JSON otherwise unchanged. Firewall behavior should come from the dedicated wrapper and entrypoint logic, not from extra devcontainer-specific hacks.

## Key decisions

- `"init": true` should always be set.
- `workspaceFolder` and `workspaceMount` should both use `${localWorkspaceFolder}` for worktree compatibility.
- `containerUser`, `remoteUser`, and `updateRemoteUserUID` should stay aligned with the Dockerfile user setup.
- `COLORTERM` forwarding keeps terminal color behavior consistent across IDEs and headless use.
- GitHub Copilot is a compatibility target for the IDE attach flow, not a separate containerized tool with its own config directory.
