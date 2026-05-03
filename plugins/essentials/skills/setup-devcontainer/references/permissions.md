# Copilot CLI Permissions

Use Copilot CLI permissions when the goal is to control what the agent may do while still allowing legitimate documentation and network access. This is the right control surface for Copilot CLI sessions; do not replace it with container-level network filtering.

The official model covers three independent areas:

1. **Tools** — which tools Copilot CLI may run
2. **Paths** — which files and directories it may access
3. **URLs** — which external domains it may fetch

## Tool permissions

Use tool approvals as the first layer of control.

- Approve tools interactively when the need is occasional.
- Use `--allow-tool` for recurring trusted tools.
- Use `--deny-tool` to block dangerous tools or subcommands explicitly.
- Use `--available-tools` when you want to restrict Copilot CLI to a specific allowlist.

Examples:

```shell
copilot --allow-tool='shell'
copilot --allow-tool='write'
copilot --deny-tool='shell(rm)'
copilot --deny-tool='shell(git push)'
copilot --available-tools='shell,write,web_fetch'
```

For `git` and `gh`, deny or allow the specific first-level subcommand when that precision matters.

If MCP servers are configured, include MCP tools in the policy discussion. Allow only the specific MCP server or tool that the workflow needs; do not grant broad MCP access just because a `.mcp.json` file exists.

## Path permissions

By default, Copilot CLI can access the current working directory, its subdirectories, and the system temp directory.

Use path controls to keep the agent inside the intended workspace instead of trying to enforce that through container networking.

- Keep the working directory scoped to the repository you actually want the agent to touch.
- Use the default path verification unless the user explicitly wants broader access.
- Use `--disallow-temp-dir` when temporary-path access is undesirable.
- Reserve `--allow-all-paths` for high-trust sessions only.

Examples:

```shell
copilot --disallow-temp-dir
copilot --allow-all-paths
```

## URL permissions

By default, external URLs require approval before Copilot CLI can access them. That default is usually the right balance for documentation lookup and web research.

Use explicit URL rules only when the workflow repeatedly hits the same trusted domains.

- Use `--allow-url=DOMAIN` to pre-approve recurring documentation, forge, or registry hosts.
- Use `--deny-url=DOMAIN` to block domains the agent should never use.
- Reserve `--allow-all-urls` for high-trust sessions only.

Examples:

```shell
copilot --allow-url=docs.github.com --allow-url=github.com
copilot --allow-url=registry.npmjs.org
copilot --deny-url=example.com
```

When Phase 1 finds recurring hosts such as GitHub Docs, package registries, Docker registries, or a custom Git LFS host, include them in this guidance only if the workflow genuinely needs Copilot CLI to fetch them.

URL permissions apply to Copilot CLI's network-aware tools and supported shell commands. They are not a complete network sandbox for every process in the devcontainer, so do not describe them as container isolation.

## High-trust shortcuts

These flags remove useful safety boundaries. Use them only when the user explicitly wants a fully trusted session:

```shell
copilot --allow-all-tools
copilot --allow-all-paths
copilot --allow-all-urls
copilot --allow-all
```

`--allow-all` and `/allow-all` or `/yolo` combine tool, path, and URL approval bypasses. Do not recommend them as the default.
