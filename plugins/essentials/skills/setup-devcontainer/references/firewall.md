# Firewall reference

## When to use

Use the firewall only for autonomous or sandboxed modes where outbound network access must be tightly constrained. Keep it **opt-in**. For normal development, do not enable the firewall.

If the workflow does not explicitly require a restricted network mode, stay with the standard devcontainer setup.

## Allowlist defaults

Start from a GitHub-focused allowlist and then add only what the project actually needs.

Default entries:
- `github.com`
- `api.github.com`
- package registries required by the detected toolchain
- container registries required by the project or by Docker support

Typical registry examples:
- npm: `registry.npmjs.org`
- PyPI: `pypi.org`, `files.pythonhosted.org`
- Cargo: `crates.io`, `static.crates.io`, `index.crates.io`
- Go: `proxy.golang.org`, `sum.golang.org`
- RubyGems: `rubygems.org`
- Docker Hub: `registry-1.docker.io`, `auth.docker.io`, `production.cloudflare.docker.com`
- GHCR and other project registries: add the exact hostnames the project already uses

Do not guess additional domains. Only include hosts that are required by the repository, its package manager, or its container image sources.

## Firewall behavior

Use a default **DROP policy** for outbound traffic and allow only the minimum required network paths:
- DNS for name resolution
- SSH for git operations when the project uses SSH remotes
- HTTP/HTTPS only for allowlisted hosts

The firewall should resolve allowlisted domains to IPs at startup and then permit only those destinations. Everything else stays blocked.

## Self-test

After applying the rules, verify both sides of the policy:
- a blocked domain such as `example.com` must fail
- an allowlisted domain such as `github.com` must succeed

This confirms the firewall is actually active and that the allowlist was populated correctly.
