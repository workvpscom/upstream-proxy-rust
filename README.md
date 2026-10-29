# Upstream Proxy

A Linux server that routes HTTP/1.1 proxy traffic from `iroh-tunnel access` through an HTTP upstream selected per client username from Redis. It supports HTTP, CONNECT and client-requested Upgrade. This guide covers v0.1.0 installation, configuration, containers, Kubernetes and an AI-agent deployment checklist. You must supply Redis, an HTTPS iroh relay, upstream proxies and a compatible `iroh-tunnel access` client. The server is not a conventional public HTTP proxy listener.

- [Choose a deployment](#choose-a-deployment)
- [Prepare identity and configuration](#prepare-identity-and-configuration)
- [Configure Redis mappings and clients](#configure-redis-mappings-and-clients)
- [Install on Linux](#install-on-linux)
- [Run the container](#run-the-container)
- [Deploy on Kubernetes](#deploy-on-kubernetes)
- [Configuration reference](#configuration-reference)
- [AI agent deployment runbook](#ai-agent-deployment-runbook)
- [Troubleshooting](#troubleshooting)

## Choose a deployment

| Environment | Supported path |
| --- | --- |
| Linux amd64 or arm64 | Release binary: GNU/glibc or musl; interactive installer or manual installation |
| Linux containers | Multi-architecture GHCR image for amd64 and arm64 |
| Kubernetes on Linux nodes | Container image and externally managed Redis, relay and secrets; example below |
| macOS or Windows | Run the Linux server in a compatible Linux VM or Linux container; on Windows, WSL2 is another Linux environment. Native macOS/Windows server binaries are not provided. |
| Other platforms or architectures | No release artifact is currently provided; do not assume compatibility. |

[Download v0.1.0](https://github.com/workvpscom/upstream-proxy-rust/releases/tag/v0.1.0). The image is `ghcr.io/workvpscom/upstream-proxy-rust:v0.1.0`. If the package is private, an authorized GHCR login or Kubernetes image pull secret is required. A checksum from the same release detects corruption, not an independently signed provenance statement. Record the checksum and image digest you deploy.

## Prepare identity and configuration

The iroh identity determines the server's stable `EndpointId`. Generate it **once**, store it securely, and reuse it across restarts, upgrades and replacements. Never generate a new identity merely because a mount failed. Only one active instance should use a given identity unless the operator has independently established a safe topology.

On a secure Linux host, for a new deployment only:

```sh
umask 077
mkdir -p ./secure
# Refuse to overwrite an existing key.
( set -C; openssl rand -base64 32 > ./secure/identity )
chmod 0400 ./secure/identity
```

The file contains standard base64 encoding of exactly 32 secret-key bytes. Do not print it, put it in Git, command arguments or logs. Back it up in an access-controlled secret store. Restore the same bytes after replacement.

Create a private `server.toml`, replacing every example value with real values:

```toml
identity_file = "/var/lib/upstream-proxy/identity"
service_name = "proxy"
relay_urls = ["https://relay.example.com/"]
redis_url = "rediss://redis.example.com:6379/0"
default_password = "REPLACE_WITH_A_SECRET"

# Optional Redis-only extra CA bundle, if required:
# redis_ca_file = "/etc/upstream-proxy/redis-ca.pem"
cache_ttl_ms = 30000
cache_capacity = 1024
redis_timeout_ms = 2000
upstream_timeout_ms = 10000
header_timeout_ms = 15000
upstream_response_header_timeout_ms = 300000
shutdown_timeout_ms = 30000
```

Use a private file or secret manager; the config can contain Redis credentials and a proxy password. `rediss://` provides Redis TLS. An optional `redis_ca_file` adds trust only for Redis, not for the iroh relay. Relay URLs must use HTTPS with certificates trusted by the binary's compiled-in roots; mounting a custom OS CA does not add relay trust in v0.1.0. Provide DNS and outbound connectivity to the relay, Redis and upstreams; permit UDP for direct iroh connectivity where applicable. No fallback relay is configured automatically.

## Configure Redis mappings and clients

Provision the Redis hash `upstream-proxy:mappings` separately. Each field is a client username; its value is an HTTP upstream proxy URL, for example `http://upstream.example.com:3128` or `http://UPSTREAM_USER:UPSTREAM_PASSWORD@upstream.example.com:3128`. Percent-encode reserved characters in URL credentials. Use your approved secret-provisioning method rather than putting real credentials in shell history or process arguments. The server reads mappings; it does not provide an admin API or create them. For example, with an approved Redis connection configured securely, `redis-cli -x HSET upstream-proxy:mappings CLIENT_USERNAME < /secure/path/upstream-url` reads the value from a protected file rather than an argument. Configure Redis authentication using your approved secure mechanism; do not put passwords in command arguments. The server needs `HGET` access to the hash and `SELECT` when using a nonzero database, plus authentication where configured.

Clients authenticate with HTTP Basic proxy authentication: username selects the mapping. The client password must equal the upstream URL password when one is present; otherwise it must equal `default_password`. An HTTP upstream does **not** encrypt that upstream hop, including credentials. Restrict Redis and upstream access. Mapping cache can delay revocation for up to `cache_ttl_ms`; already-open tunnels are not revoked midstream.

Only `http://` upstream URLs are supported; SOCKS and HTTPS upstream proxies are not supported. After startup, capture the server's `EndpointId:` from its logs. Install a compatible [iroh-tunnel access v0.4.2](https://github.com/naicoi92/iroh-tunnel/releases/tag/v0.4.2) client for its supported platform. Configure access with that endpoint, the same `service_name`, and the appropriate relay configuration. Use that client's local HTTP proxy endpoint in your application; use the client-specific proxy credentials above. Consult the access client's own documentation for its platform-specific commands and configuration. An EndpointId is an identifier, not an authentication secret. For example, on the access host, using the local port `18080` and replacing the endpoint ID:

```sh
iroh-tunnel access config add --name proxy --protocol tcp --node_id YOUR_ENDPOINT_ID --host 127.0.0.1 --port 18080
iroh-tunnel access run
```

The service name `proxy` must match `service_name` above. To configure the access relay explicitly, edit the access config using its documented `iroh-tunnel access config edit` command. The following v0.4.2 example uses the server's configured relay and a loopback listener; replace the endpoint ID and relay:

```toml
[node]
relay_urls = ["https://relay.example.com/"]

[[services]]
name = "proxy"
node_id = "YOUR_ENDPOINT_ID"
protocol = "tcp"
host = "127.0.0.1"
port = 18080
multiplex = true
```

Access may create and persist its own identity on first run; protect its config and keep it if a stable access identity is needed. Configure an optional relay token according to the access documentation if your relay requires it. In a separate terminal, prepare a private curl config (mode `0600`) with `proxy = "http://127.0.0.1:18080"`, `proxy-user = "USERNAME:PASSWORD"` and `url = "https://YOUR_TEST_DESTINATION/"`; then run `curl --config /secure/path/proxy-test.curl`. Supply credentials through an approved secure channel, not command arguments. Verify the intended upstream and destination were reached; a running process is not proof of readiness.

## Install on Linux

Choose the matching binary and download `install.sh`, `SHA256SUMS` and that binary from the **same** release. For example, Linux amd64 glibc:

```sh
base=https://github.com/workvpscom/upstream-proxy-rust/releases/download/v0.1.0
curl -fLO "$base/SHA256SUMS"
curl -fLO "$base/install.sh"
curl -fLO "$base/upstream-proxy-v0.1.0-linux-amd64-gnu"
sha256sum -c SHA256SUMS --ignore-missing
```

`--ignore-missing` checks only downloaded files; ensure both selected files appear as `OK`. For other hosts use `arm64` instead of `amd64`, and `musl` instead of `gnu` where appropriate. On systems without GNU `sha256sum`, use a compatible SHA-256 checker and compare the exact filenames and hashes.

For interactive installation, review `install.sh` first, then run `sh install.sh`. It asks for a stable version and confirmations. User installations use `~/.local/bin/upstream-proxy`; system installations use `/usr/local/bin/upstream-proxy` and require root. Service integration depends on the host: user mode supports systemd, OpenRC or an existing runit supervisor; system mode supports systemd, OpenRC, runit and compatible SysV. If unavailable, choose binary-only. Inspect existing OpenRC, runit or SysV services before upgrading; they may require manual intervention. The installer backs up an existing binary and offers rollback on service setup failure.

For manual binary-only installation, install the verified file as `upstream-proxy` in a suitable executable directory. Prepare the config and identity at paths readable only by the service account. On Unix, the identity file must have **no group or other permissions**; mode `0400` owned by the service account is appropriate. The config must also be protected. Then validate and run:

```sh
upstream-proxy --check-config /path/to/server.toml
upstream-proxy --check-identity /path/to/identity
upstream-proxy --check-runtime /path/to/server.toml
upstream-proxy --stop-grace-seconds /path/to/server.toml
upstream-proxy /path/to/server.toml
```

The checks validate files, not network readiness. `--check-runtime` checks config and configured identity. `--stop-grace-seconds` returns the drain timeout rounded up plus 10 seconds. Start the foreground process as the intended non-root user. Capture `EndpointId` and perform the end-to-end check above. Stop with SIGTERM or SIGINT. Preserve config and identity before upgrading; rollback by restoring the previous verified binary and restarting with the same files. Uninstall the binary/service using the relevant service manager only after stopping it; retain or securely dispose of secrets according to your policy.

### Install a systemd service manually

On an authorized systemd host, use a verified binary and securely prepared `server.toml` and `identity` with the paths shown above. These commands create a dedicated account and install files; do not run them over an existing deployment without checking ownership and backing up its identity first.

```sh
sudo groupadd --system upstream-proxy
sudo useradd --system --gid upstream-proxy --no-create-home --home-dir /nonexistent --shell /usr/sbin/nologin upstream-proxy
sudo install -o root -g root -m 0555 ./upstream-proxy-v0.1.0-linux-amd64-gnu /usr/local/bin/upstream-proxy
sudo install -d -o root -g upstream-proxy -m 0750 /etc/upstream-proxy /var/lib/upstream-proxy
sudo install -o upstream-proxy -g upstream-proxy -m 0400 /secure/path/server.toml /etc/upstream-proxy/server.toml
sudo install -o upstream-proxy -g upstream-proxy -m 0400 /secure/path/identity /var/lib/upstream-proxy/identity
sudo -u upstream-proxy /usr/local/bin/upstream-proxy --check-runtime /etc/upstream-proxy/server.toml
```

Select the binary for your architecture and libc. Install an optional Redis CA at the configured path with appropriate ownership. Create `/etc/systemd/system/upstream-proxy.service` as root:

```ini
[Unit]
Description=Upstream Proxy (iroh)
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
Type=exec
User=upstream-proxy
Group=upstream-proxy
ExecStart=/usr/local/bin/upstream-proxy /etc/upstream-proxy/server.toml
WorkingDirectory=/
Restart=on-failure
RestartSec=5s
KillSignal=SIGTERM
KillMode=mixed
TimeoutStopSec=40s
UMask=0077
NoNewPrivileges=true
CapabilityBoundingSet=
AmbientCapabilities=
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectKernelLogs=true
ProtectControlGroups=true
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6 AF_NETLINK
RestrictSUIDSGID=true
RestrictRealtime=true
LockPersonality=true
RemoveIPC=true

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now upstream-proxy
systemctl status upstream-proxy --no-pager
journalctl -u upstream-proxy -n 50 --no-pager
```

Before upgrade, back up the binary, config and identity securely. Stop the service, install the verified new binary, validate runtime, and start it again. Roll back by stopping it and restoring the previous binary; preserve identity throughout. For uninstall, disable and stop the service, remove its unit and binary, then run `systemctl daemon-reload`; retain or securely dispose of runtime secrets separately. Increase `TimeoutStopSec` if the configured shutdown drain increases. Systemd deployment has not been runtime-verified as part of this release.

## Run the container

The image is distroless: no shell or package manager. It runs as UID/GID `65532`; its default command reads `/etc/upstream-proxy/server.toml`. Prepare directories containing the private config and identity. The identity must be owned by the effective container UID, mode `0400`; account for user-namespace remapping. Mount both directories read-only, with config paths matching the example above.

```sh
docker run -d --name upstream-proxy \
  --user 65532:65532 --read-only --cap-drop=ALL \
  --security-opt=no-new-privileges:true --stop-timeout=40 \
  --mount type=bind,src=/secure/upstream-proxy/etc,dst=/etc/upstream-proxy,readonly \
  --mount type=bind,src=/secure/upstream-proxy/state,dst=/var/lib/upstream-proxy,readonly \
  ghcr.io/workvpscom/upstream-proxy-rust:v0.1.0
docker logs upstream-proxy
```

Replace the host paths with your secure directories. No HTTP port mapping is needed for this server. If you change `shutdown_timeout_ms`, increase the runtime stop timeout above the drain plus cleanup and scheduling margin; 40 seconds is intended for the default 30-second drain. Inspect logs without exposing config or identity. Restart with the same mounts. Do not expect `docker exec ... sh` to work in a distroless image.

## Deploy on Kubernetes

This example uses one replica and a pre-existing Kubernetes Secret named `upstream-proxy-runtime` containing keys `server.toml` and `identity`. It stages files into a memory-backed volume with the ownership and `0400` identity mode required by the application. The init container runs as root solely to set ownership; clusters enforcing restricted Pod Security may require an approved alternative secret delivery mechanism that preserves UID `65532` and mode `0400`. Protect Kubernetes Secrets with RBAC and encryption at rest; base64 is not encryption. Do not commit the Secret manifest or its values.

Prepare `server.toml` with the paths shown above and create the Secret through your organization's approved tooling. If GHCR is private, configure a namespace-local registry pull Secret and add `imagePullSecrets` to the Pod spec according to your cluster policy. If using `kubectl create secret generic --from-file`, ensure local source files and shell history are protected. The example needs egress to DNS, relay HTTPS, Redis, upstream proxy and, where available, direct UDP. Adapt NetworkPolicies to your environment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: upstream-proxy
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: upstream-proxy
  template:
    metadata:
      labels:
        app: upstream-proxy
    spec:
      terminationGracePeriodSeconds: 40
      initContainers:
        - name: stage-secrets
          image: busybox:1.36
          command: ["sh", "-ec"]
          args:
            - >-
              cp /input/server.toml /runtime/server.toml;
              cp /input/identity /runtime/identity;
              chmod 0400 /runtime/server.toml /runtime/identity;
              chown 65532:65532 /runtime/server.toml /runtime/identity
          securityContext:
            runAsUser: 0
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
              add: [CHOWN]
          volumeMounts:
            - name: input
              mountPath: /input
              readOnly: true
            - name: runtime
              mountPath: /runtime
      containers:
        - name: upstream-proxy
          image: ghcr.io/workvpscom/upstream-proxy-rust:v0.1.0
          securityContext:
            runAsNonRoot: true
            runAsUser: 65532
            runAsGroup: 65532
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
          volumeMounts:
            - name: runtime
              mountPath: /etc/upstream-proxy/server.toml
              subPath: server.toml
              readOnly: true
            - name: runtime
              mountPath: /var/lib/upstream-proxy/identity
              subPath: identity
              readOnly: true
      volumes:
        - name: input
          secret:
            secretName: upstream-proxy-runtime
            defaultMode: 0400
        - name: runtime
          emptyDir:
            medium: Memory
```

If using a private Redis CA, also stage and mount it at the configured `redis_ca_file` path. Staged Secret changes require a Pod restart. Do not use a ConfigMap for secret-bearing TOML. Pin images by digest for production and use an approved pinned init image. Set `terminationGracePeriodSeconds` above the configured drain plus cleanup and margin.

Apply only to an authorized cluster and namespace:

```sh
kubectl create namespace YOUR_NAMESPACE
kubectl -n YOUR_NAMESPACE create secret generic upstream-proxy-runtime \
  --from-file=server.toml=/secure/path/server.toml \
  --from-file=identity=/secure/path/identity
kubectl -n YOUR_NAMESPACE apply -f upstream-proxy.yaml
kubectl -n YOUR_NAMESPACE rollout status deployment/upstream-proxy
kubectl -n YOUR_NAMESPACE logs deployment/upstream-proxy
# After verifying the access path, exercise an authenticated proxy request.
# To return to the previous deployment revision if necessary:
kubectl -n YOUR_NAMESPACE rollout undo deployment/upstream-proxy
```

There is no server HTTP health endpoint or fixed public HTTP listener: an HTTP readiness probe, HTTP Ingress or Service targeting a guessed port would be misleading. Kubernetes rollout status only proves container lifecycle, not Redis, relay or end-to-end proxy readiness. Validate with a real access client and proxy request. The example has not been verified in a Kubernetes cluster; validate security policy, networking and connectivity in your own environment before production.

## Configuration reference

The server reads one TOML file, at most 16 KiB. Unknown fields are rejected. All numeric values are positive integers; `_ms` values are milliseconds.

| Field | Required / default | Meaning and constraints |
| --- | --- | --- |
| `identity_file` | Required | Non-empty path to the protected base64 32-byte iroh secret key. |
| `service_name` | Required | 1–63 lowercase ASCII letters, digits or hyphens; must match access configuration. |
| `relay_urls` | Required | Array of 1–8 HTTPS URLs with a host; no URL credentials, query or fragment. |
| `redis_url` | Required | `redis://` or `rediss://` URL with host; optional numeric database path; no query or fragment. Credentials, if used, belong in protected config. |
| `redis_ca_file` | Optional | PEM file with additional Redis TLS roots; maximum 64 KiB and 128 certificates. Not a relay CA. |
| `default_password` | Required | Non-empty client password for mappings without an upstream password. |
| `cache_ttl_ms` | `30000` | Mapping cache lifetime. |
| `cache_capacity` | `1024` | Maximum number of cached mappings. |
| `redis_timeout_ms` | `2000` | Redis operation deadline; maximum 300000 ms. |
| `upstream_timeout_ms` | `10000` | Upstream connect/CONNECT handshake deadline; maximum 300000 ms. |
| `header_timeout_ms` | `15000` | Client request-header deadline; maximum 300000 ms. |
| `upstream_response_header_timeout_ms` | `300000` | Wait for upstream final response headers after request upload; not an upload, body or tunnel idle timeout. |
| `shutdown_timeout_ms` | `30000` | Graceful drain deadline; adjust service/container termination grace as well. |

## AI agent deployment runbook

1. **Inventory and authorization:** identify target OS, CPU architecture, libc and deployment runtime; confirm the operator-authorized host or cluster and namespace. Use Linux binaries or Linux containers only; do not claim unsupported native platforms. Do not provision infrastructure or modify firewall rules without authorization.
2. **Collect prerequisites:** obtain a compatible access client, relay URL, Redis endpoint/TLS trust, upstream mappings and secret storage. Ask for missing values through an approved secure channel; never request secrets in chat or print them.
3. **Preserve state:** identify an existing deployment, back up its config and identity securely, record the existing binary checksum/image digest and rollback method. Never replace an existing identity silently.
4. **Select and verify artifacts:** download the correct v0.1.0 binary or image; verify release checksums and record the image digest. If GHCR is private, use an authorized pull credential without exposing it.
5. **Configure:** render the complete TOML above using approved values, provision Redis mappings securely, ensure paths, ownership and mode are correct. Run `--check-config`, `--check-identity` and `--check-runtime` in the target environment when possible; they do not test network readiness.
6. **Start and verify:** deploy only to the approved target, capture `EndpointId`, configure access with matching service/relay settings, and execute an authenticated request through access to the expected upstream and destination. Verify restart retains EndpointId. Record failures without logging secrets.
7. **Report:** state target, version, digest/checksum, deployment path, verification actually performed and any limitations. On failure, stop and use the agreed rollback; do not declare E2E success from static checks, a running Pod or a successful image pull.

## Troubleshooting

- **Invalid configuration:** check required fields, TOML syntax, size and constraints above; do not paste secrets into tickets.
- **Invalid identity:** verify that the file exists, contains base64 for 32 bytes, is owned by the effective user and has no group/other permissions. Preserve the original identity.
- **Proxy authentication failure:** verify the client username, Redis hash field and password rule. Allow for mapping-cache TTL after changes.
- **Relay or Redis connection failure:** verify DNS, egress, TLS certificates and the correct CA scope. No public relay fallback is implied.
- **Container image pull fails:** check GHCR visibility and registry credentials. A private package needs authorized access.
- **Pod is running but requests fail:** inspect logs and test the complete access → server → Redis → upstream → destination path; there is no built-in HTTP readiness endpoint.
