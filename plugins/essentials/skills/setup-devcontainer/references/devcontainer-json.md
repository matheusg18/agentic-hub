# Phase 3: devcontainer.json

**Ask the user:** should the container share host GitHub auth and local IDE settings, or start isolated?

Use the Dev Container spec for GitHub-authenticated development and GitHub Copilot-compatible IDE attach flows. Keep the configuration generic enough for VS Code, JetBrains, DevPod, and `devcontainer` CLI use.

## Path A: Shared host auth and settings

Use this when the host already has GitHub CLI auth, SSH agent access, and any local developer settings you want to reuse.
Start with the minimal shared-host example below, then add only the optional mounts you actually need.

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
  "remoteEnv": {
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
- Mount `~/.gitconfig` read-only when the file exists on the host so the container can read identity and aliases without mutating host config.
- Do not add any standalone tool-specific config mount. GitHub Copilot should use the IDE’s normal attach/auth flow, not a separate in-container tool path.
- SSH agent forwarding is enough; do not mount `~/.ssh` unless the repository explicitly needs a separate key file.

Optional shared-host snippets:

```json
{
  "mounts": [
    "source=${localEnv:HOME}/.gitconfig,target=/tmp/home/.gitconfig,type=bind,readonly"
  ]
}
```

Use this only when the host `~/.gitconfig` file exists.

```json
{
  "mounts": [
    "source=${localEnv:HOME}/.config/gh,target=/tmp/home/.config/gh,type=bind,readonly"
  ]
}
```

Use this only when the host `~/.config/gh` directory exists and you want to reuse GitHub CLI auth state.

```json
{
  "mounts": [
    "source=${localEnv:SSH_AUTH_SOCK},target=/tmp/ssh-agent.sock,type=bind"
  ],
  "remoteEnv": {
    "SSH_AUTH_SOCK": "/tmp/ssh-agent.sock"
  }
}
```

Use this only when the host SSH agent is available.

```json
{
  "mounts": [
    "source=${localEnv:HOME}/.kube,target=/tmp/home/.kube,type=bind,readonly"
  ],
  "remoteEnv": {
    "KUBECONFIG": "/tmp/home/.kube/config"
  }
}
```

Use this only when host Kubernetes config should be reused.

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
    "source=gh-config-${devcontainerId},target=/tmp/home/.config/gh,type=volume"
  ],
  "remoteEnv": {
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
- On first run, authenticate inside the container with `gh auth login --hostname github.com --web --git-protocol https`, then run `gh auth setup-git` if you want GitHub CLI to configure git credentials.
- Keep isolated mode free of host auth, user config, and SSH agent mounts; do not reuse the host SSH agent socket or any host auth/user config. The named `gh` volume is the only persisted auth state.
- Do not invent host-side Copilot mounts or any other tool-specific config directories in isolated mode.

## Optional Kubernetes config

If the host Kubernetes context should be reused, add the optional shared-host kube snippet above. Keep it out of isolated mode unless the host kubeconfig is explicitly meant to be shared.

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
