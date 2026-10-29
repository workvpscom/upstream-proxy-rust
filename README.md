# Upstream Proxy

Upstream Proxy provides an HTTP/1.1 proxy endpoint over iroh. Client usernames select upstream proxy mappings stored in Redis. The server supports HTTP, CONNECT and Upgrade traffic.

## Download and verify

Download `install.sh`, `SHA256SUMS` and the matching binary from the same GitHub Release. Linux targets are `amd64` and `arm64`, with `gnu` (glibc) and `musl` builds. Verify the checksum before running any downloaded file:

```sh
sha256sum -c SHA256SUMS --ignore-missing
```

A checksum downloaded from the same release detects corruption but is not an independent signature. Releases may be rebuilt for the same source commit; record the checksum or image digest you deploy.

## Install

Run the installer interactively after reviewing it:

```sh
sh install.sh
```

The installer asks for a stable release version and confirmation before changing an existing binary. User installations use `~/.local/bin/upstream-proxy`; system installations use `/usr/local/bin/upstream-proxy` and require root. User service setup supports systemd, OpenRC and an existing runit supervisor. System service setup supports systemd, OpenRC, runit and a compatible SysV backend. Availability depends on the host's service manager and required tools. If no user service manager is available, choose binary-only installation when prompted.

The installer verifies the downloaded binary before installation. For upgrades it keeps a backup and asks before rollback when service setup fails. Managed systemd services can be restarted during upgrade; existing OpenRC, runit or SysV services may require manual inspection before upgrade. It does not silently replace an existing identity. Inspect service state after rollback; a running process alone does not establish proxy readiness.

## Configure and operate

Provide a service name, HTTPS relay URL, Redis URL and a non-empty default password. Provision Redis mappings separately; do not put runtime secrets in release assets or container images. The iroh identity is created once and must be retained across restarts and upgrades: replacing it changes the endpoint identity. Protect the identity and configuration files and back them up securely.

Validate configuration before starting:

```sh
upstream-proxy --check-config /path/to/server.toml
upstream-proxy --check-runtime /path/to/server.toml
```

These commands do not verify an end-to-end proxy request. Confirm readiness with a real client through the configured iroh access, Redis and upstream proxy.

The default drain timeout is 30 seconds. Configure the service manager's stop grace above the configured drain timeout plus cleanup and scheduling margin; increase the grace whenever `shutdown_timeout_ms` increases.

## Container image

The release pipeline is designed to publish a multi-architecture image to `ghcr.io/workvpscom/upstream-proxy-rust`. Mount configuration and identity as protected read-only files; do not put secrets in image layers. The image runs as a non-root user and requires a stop timeout appropriate for the configured drain.

Publication and installation behavior must be verified against the actual release before production use.
