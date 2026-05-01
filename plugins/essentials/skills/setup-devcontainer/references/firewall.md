# Firewall reference

Use this phase only when the user explicitly wants a restricted outbound network for autonomous or sandboxed execution. Keep it **opt-in**. For normal development, do not enable the firewall.

This reference covers the implementation details that back Phase 5 in `SKILL.md`: allowlist generation, `firewall.sh`, Dockerfile additions, entrypoint changes, IDE/task-runner wiring, and firewall-specific validation.

## 1) Allowlist file

Create `.devcontainer/firewall-allowlist.txt` and resolve it into IP rules at container start.

Example shape:

```text
# Allowed outbound domains — one hostname per line.
# Comments start with #.
# The firewall resolves these domains to IPs at container start.

# GitHub baseline
github.com
api.github.com

# Package registries (populate from Phase 1)
registry.npmjs.org
```

### GitHub-oriented defaults

Start from the GitHub endpoints that the selected workflow genuinely needs:

| Hostname | When to include it |
|----------|--------------------|
| `github.com` | Default Git remote / browser / release download host for GitHub-based repositories |
| `api.github.com` | `gh`, GitHub API automation, and many GitHub-authenticated workflows |
| `uploads.github.com` | Release asset uploads or other GitHub upload APIs |
| `ghcr.io` | Projects that pull from or publish to GitHub Container Registry |
| `api.githubcopilot.com` | Only when the chosen IDE or workflow requires direct Copilot API access from inside the container |

Do **not** add GitHub Copilot-specific hosts by default. Include them only when the selected editor workflow demonstrably uses them.

### Package registry mapping

Populate registry domains from the package managers detected in Phase 1:

| Package manager | Registry domain(s) |
|-----------------|--------------------|
| npm / yarn / pnpm | `registry.npmjs.org` |
| pip / uv | `pypi.org`, `files.pythonhosted.org` |
| cargo | `crates.io`, `static.crates.io`, `index.crates.io` |
| go | `proxy.golang.org`, `sum.golang.org` |
| rubygems | `rubygems.org` |
| Maven / Gradle | `repo1.maven.org` |
| Composer | `repo.packagist.org`, `packagist.org` |

Only include domains the repository actually needs. Do not guess extra registries.

### Additional sources to scan

Add exact hostnames from repository evidence:

- project forge hosts when GitHub Enterprise or another GitHub-compatible host is in use
- registries found in `FROM` lines, compose `image:` fields, or CI registry login steps
- Docker Hub domains when Docker support is enabled: `registry-1.docker.io`, `auth.docker.io`, `production.cloudflare.docker.com`
- Kubernetes API server hosts from kubeconfig `server:` entries
- Helm chart repository hosts from `helmfile.yaml`, `helmfile.yml`, or other checked-in Helm config

Wildcards are not suitable here. The startup script resolves concrete hostnames with `dig`, so convert any cloud-specific wildcard pattern into the actual host the repository uses.

### Git LFS domains

When Git LFS is detected, add the LFS object hosts as well:

| Scenario | Domain handling |
|----------|-----------------|
| GitHub-hosted LFS | Add `github-cloud.s3.amazonaws.com` and `objects.githubusercontent.com` |
| GitHub Enterprise with standard GitHub LFS routing | Start with the repository host, then add any separate object host proven by `.lfsconfig` or `git config lfs.url` |
| Custom LFS server | Parse the hostname from `.lfsconfig` `lfs.url` or `git config lfs.url` |

Git LFS often serves objects from a different host than the git remote. If the repository uses LFS and those domains are missing, `git lfs pull` will fail even though `git fetch` works.

## 2) `firewall.sh`

Create `.devcontainer/firewall.sh` and keep the allowlist external so users edit the domain list, not the shell logic.

```bash
#!/usr/bin/env bash
set -euo pipefail

ALLOWLIST_FILE="${ALLOWLIST_FILE:-/usr/local/share/firewall-allowlist.txt}"
ALLOWED_IPS_SNAPSHOT="${ALLOWED_IPS_SNAPSHOT:-/run/firewall-allowed-ips.txt}"
EXPECTED_DNS_RESOLVER="${EXPECTED_DNS_RESOLVER:-127.0.0.11}"

resolved_dns_resolver="$(awk '/^nameserver[[:space:]]+/ { print $2; exit }' /etc/resolv.conf)"
if [[ "$resolved_dns_resolver" != "$EXPECTED_DNS_RESOLVER" ]]; then
    echo "FIREWALL ERROR: expected Docker embedded DNS at $EXPECTED_DNS_RESOLVER, found ${resolved_dns_resolver:-none}" >&2
    exit 1
fi

# This example is IPv4-only, so disable IPv6 rather than leaving an unfiltered bypass.
sysctl -w net.ipv6.conf.all.disable_ipv6=1 >/dev/null
sysctl -w net.ipv6.conf.default.disable_ipv6=1 >/dev/null
sysctl -w net.ipv6.conf.lo.disable_ipv6=1 >/dev/null

# Preserve Docker's embedded DNS rules before flushing tables.
docker_dns_rules="$(iptables-save | grep -E '127\.0\.0\.11' || true)"

iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

if [[ -n "$docker_dns_rules" ]]; then
    echo "$docker_dns_rules" | iptables-restore --noflush
fi

ipset create allowed-domains hash:net -exist
ipset flush allowed-domains

while IFS= read -r raw_line; do
    domain="${raw_line%%#*}"
    domain="${domain// /}"
    [[ -z "$domain" ]] && continue

    ips="$(dig +short A "$domain" 2>/dev/null | grep -E '^[0-9]+\.' || true)"
    if [[ -z "$ips" ]]; then
        echo "warning: no IPv4 addresses resolved for $domain" >&2
        continue
    fi

    while IFS= read -r ip; do
        [[ -z "$ip" ]] && continue
        ipset add allowed-domains "$ip/32" -exist
    done <<< "$ips"
done < "$ALLOWLIST_FILE"

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP

iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

iptables -A OUTPUT -d "$EXPECTED_DNS_RESOLVER"/32 -p udp --dport 53 -j ACCEPT
iptables -A OUTPUT -d "$EXPECTED_DNS_RESOLVER"/32 -p tcp --dport 53 -j ACCEPT

iptables -A OUTPUT -p tcp --dport 22 -m set --match-set allowed-domains dst -j ACCEPT
iptables -A OUTPUT -p tcp --dport 80 -m set --match-set allowed-domains dst -j ACCEPT
iptables -A OUTPUT -p tcp --dport 443 -m set --match-set allowed-domains dst -j ACCEPT

iptables -A OUTPUT -j REJECT --reject-with icmp-admin-prohibited

mkdir -p "$(dirname "$ALLOWED_IPS_SNAPSHOT")"
ipset list allowed-domains | grep -E '^[0-9]' > "$ALLOWED_IPS_SNAPSHOT" || true

if curl -sf --max-time 3 https://example.com >/dev/null 2>&1; then
    echo "FIREWALL ERROR: example.com should be blocked but is reachable" >&2
    exit 1
fi

echo "Firewall active — $(wc -l < "$ALLOWED_IPS_SNAPSHOT" | tr -d ' ') IPs allowed"
```

Implementation notes:

- This example assumes Docker's embedded DNS at `127.0.0.11` and fails closed if the container is using a different resolver. Do not replace this with a broad "allow any DNS" rule.
- Preserve Docker DNS rules before flushing `iptables`, or container DNS may stop working.
- Disable IPv6 in this mode because the example only applies IPv4 filtering. If the workload genuinely needs IPv6, add equivalent `ip6tables` / `ipset` rules before treating the container as restricted.
- Keep the default outbound policy at `DROP`.
- Allow only Docker DNS plus SSH/HTTP/HTTPS to allowlisted destinations.
- Resolve domains once at startup; if CDN IPs rotate, restart the container.
- Snapshot the resolved IPs before dropping privileges so a helper such as `firewall-list` can display them later.

## 3) Dockerfile additions

Add the firewall dependencies in the same system-packages layer that already installs `gosu` for the root-to-user privilege drop. Do not add a second `gosu` install path here.

```dockerfile
# -- Firewall packages (optional, firewalled mode only) -------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
        iptables ipset iproute2 dnsutils procps curl \
    && rm -rf /var/lib/apt/lists/*
```

If the Dockerfile already follows [the base reference](dockerfile.md), fold these packages into that existing `apt-get install` block and leave `gosu` there.

Copy the runtime files into the image:

```dockerfile
COPY firewall.sh /usr/local/bin/firewall.sh
RUN chmod +x /usr/local/bin/firewall.sh

COPY firewall-allowlist.txt /usr/local/share/firewall-allowlist.txt

RUN printf '#!/usr/bin/env bash\ncat /run/firewall-allowed-ips.txt 2>/dev/null || echo "Firewall not active"\n' \
        > /usr/local/bin/firewall-list \
    && chmod +x /usr/local/bin/firewall-list
```

Why `gosu` is required:

- firewalled mode starts as root so it can program `iptables` / `ipset`
- the shell or IDE session should still run as the target non-root UID afterwards
- `gosu` performs that final privilege drop cleanly without leaving the container session running as root

## 4) Entrypoint changes

Update `.devcontainer/entrypoint.sh` so it supports both normal mode and firewalled mode.

```bash
#!/usr/bin/env bash
set -euo pipefail

DEFAULT_USER="${DEFAULT_USER:-dev}"
DEFAULT_HOME="${DEFAULT_HOME:-${HOME:-/tmp/home}}"
target_uid="${DEVCONTAINER_UID:-$(id -u)}"
target_gid="${DEVCONTAINER_GID:-$(id -g)}"
target_user="${DEFAULT_USER}"

if [[ "$(id -u)" = "0" ]] && [[ -z "${DEVCONTAINER_UID:-}" ]]; then
    workspace="${DEVCONTAINER_WORKSPACE:-$(pwd)}"
    if [[ -d "$workspace" ]]; then
        target_uid="$(stat -c '%u' "$workspace")"
        target_gid="$(stat -c '%g' "$workspace")"
    fi
fi

if [[ "$(id -u)" = "0" ]] && ! getent group "$target_gid" >/dev/null 2>&1; then
    echo "devcontainer-${target_gid}:x:${target_gid}:" >> /etc/group
fi

if existing_passwd="$(getent passwd "$target_uid")"; then
    target_user="${existing_passwd%%:*}"
elif [[ "$(id -u)" = "0" ]]; then
    if getent passwd "$DEFAULT_USER" >/dev/null 2>&1; then
        target_user="devcontainer-${target_uid}"
    fi
    echo "${target_user}:x:${target_uid}:${target_gid}:dev:${DEFAULT_HOME}:/bin/bash" >> /etc/passwd
fi

configure_docker_socket_access() {
    local socket_path="/var/run/docker.sock"
    local socket_gid socket_group

    [[ -S "$socket_path" ]] || return 0

    socket_gid="$(stat -c '%g' "$socket_path" 2>/dev/null || true)"
    if [[ -z "$socket_gid" ]]; then
        echo "warning: skipping Docker socket GID mapping; could not read ${socket_path}" >&2
        return 0
    fi

    socket_group="$(getent group "$socket_gid" | cut -d: -f1 || true)"
    if [[ -z "$socket_group" ]]; then
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

if [[ "$(id -u)" = "0" ]]; then
    mkdir -p "$DEFAULT_HOME"
    chown "${target_uid}:${target_gid}" "$DEFAULT_HOME"
    configure_docker_socket_access
fi

if [[ "${DEVCONTAINER_FIREWALL:-}" = "1" ]] && [[ "$(id -u)" = "0" ]]; then
    /usr/local/bin/firewall.sh
fi

if [[ "$(id -u)" = "0" ]] && [[ "$target_uid" != "0" ]]; then
    exec gosu "${target_user}:${target_gid}" "$@"
fi

exec "$@"
```

Mode behaviour:

- **normal CLI/task-runner mode**: the container starts through the root entrypoint, typically with `DEVCONTAINER_UID` / `DEVCONTAINER_GID` set by the task runner; because `DEVCONTAINER_FIREWALL` is unset, the firewall path is skipped and the entrypoint drops privileges normally
- **IDE attach without explicit UID mapping**: container may start as root, infer the target UID from the workspace owner, map the Docker socket group if present, then drop privileges
- **firewalled mode**: the launcher adds `NET_ADMIN` / `NET_RAW` and sets `DEVCONTAINER_FIREWALL=1`; the entrypoint applies firewall rules as root, then drops to the target UID via `gosu`

## 5) `devcontainer.json` additions

If the user wants the IDE itself to launch the firewalled path, add the firewall capabilities and explicit environment variables.

```json
{
  "runArgs": ["--cap-add=NET_ADMIN", "--cap-add=NET_RAW"],
  "containerEnv": {
    "DEVCONTAINER_FIREWALL": "1",
    "DEVCONTAINER_UID": "${localEnv:UID}",
    "DEVCONTAINER_GID": "${localEnv:GID}"
  }
}
```

Notes:

- keep these additions conditional on the user opting into firewalled mode
- `${localEnv:UID}` / `${localEnv:GID}` depend on the host environment exporting them
- the task runner path is often more reliable because it can call `id -u` / `id -g` directly and set `DEVCONTAINER_FIREWALL=1` itself when needed

## 6) Validation

Revalidate both the restrictive and the allowed paths:

- `curl https://example.com` must fail
- `sysctl net.ipv6.conf.all.disable_ipv6` must report `1` in this mode
- `gh auth status` must still work when the required GitHub hosts are allowlisted
- `ssh -T git@github.com` must work when the repository uses SSH remotes
- `awk '/^nameserver[[:space:]]+/ { print $2; exit }' /etc/resolv.conf` should print `127.0.0.11`
- `firewall-list` should show resolved IPs
- `git lfs pull` should work when LFS is enabled and its object hosts were allowlisted
- pulls from GHCR or Docker Hub should still work when those registries were included deliberately

If an expected operation fails, add the missing domain based on repository evidence and rerun the check. Do not broaden the allowlist with speculative hosts.
