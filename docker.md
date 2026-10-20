# Docker

The image `ofkindness/xenia` holds the same static binary as the [Linux archives](index.md#installing), for amd64 and arm64, and little else: no shell, no package manager. It runs as its own user (uid 65532) and needs no added privileges or capabilities.

```sh
docker run -d --name xenia -p 80:8080 ofkindness/xenia:0.15.1
curl http://localhost/          # hello from xenia: the built-in route
```

| | |
| --- | --- |
| Binary | `/usr/local/bin/xenia`, the entrypoint |
| Config | `/etc/xenia/xenia.conf.js`, the working directory; without one, the built-in route |
| Ports | 8080 (HTTP), 8443 (HTTPS, with `--listen-tls`) |
| Admin socket | `/run/xenia/admin.sock`; `XENIA_ADMIN` points `xenia ctl` at it |
| Access log | JSON lines on standard output (`--access-log -`), after the `listening on` lines |
| Error log | standard error |
| CA certificates | `/etc/ssl/certs/ca-certificates.crt`, Mozilla's, for upstreams' `tls.ca` |
| Licenses | `/licenses/` |

The default command is `--listen 0.0.0.0:8080 --admin /run/xenia/admin.sock --access-log -`. Arguments given to `docker run` replace it, so keep `--admin` when changing it: the healthcheck uses it.

## A config

Mount the project's directory, the config with the modules it imports, at `/etc/xenia`, read-only. Its files must be readable by uid 65532, as every reload reads them again (see [Deployment](deployment.md#who-reads-what)).

```sh
docker run -d --name xenia -p 80:8080 \
  -v "$PWD:/etc/xenia:ro" \
  ofkindness/xenia:0.15.1
docker exec xenia xenia check   # validate the config the server would load
```

Paths in the config are relative to it, so certificates and served files can sit in the same directory, or in other mounts named by absolute paths.

## HTTPS

```sh
docker run -d --name xenia -p 80:8080 -p 443:8443 \
  -v "$PWD:/etc/xenia:ro" \
  ofkindness/xenia:0.15.1 \
  --listen 0.0.0.0:8080 --listen-tls 0.0.0.0:8443 \
  --admin /run/xenia/admin.sock --access-log -
```

The config's `tls.certificates` name the certificate files (see [TLS](configuration.md#tls)). With `tls.redirect`, plain requests are sent to the first `--listen-tls` port, here 8443, which is not the one clients reach. For redirects, listen on 80 and 443 inside the container as well: Docker lets a non-root process bind them (`-p 80:80 -p 443:443` with `--listen 0.0.0.0:80 --listen-tls 0.0.0.0:443`), as do Kubernetes clusters that allow the `net.ipv4.ip_unprivileged_port_start` sysctl.

## Reloads and logs

```sh
docker kill -s HUP xenia          # recompile the config and publish it
docker exec xenia xenia ctl reload   # the same, waiting for the outcome
docker exec xenia xenia ctl status
docker logs -f xenia              # access log lines (JSON) and errors
```

A reload that fails leaves the running config in place and says why in the log and in `xenia ctl status`.

## Stopping

`docker stop` sends `SIGTERM`: xenia stops accepting and lets responses in flight finish, for up to `server.shutdownTimeout` (10 s by default). Docker waits 10 s before it kills, so give it longer than the drain: `docker stop -t 30`, or `stop_grace_period` in compose.

## Health

The image's `HEALTHCHECK` runs `xenia ctl status` every 30 s against the admin socket. Elsewhere, probe a route of the config, or `GET /status` on an `--admin-listen` address with its token.

## Compose

```yaml
services:
  xenia:
    image: ofkindness/xenia:0.15.1
    ports: ["80:8080", "443:8443"]
    volumes: ["./site:/etc/xenia:ro"]
    command: ["--listen", "0.0.0.0:8080", "--listen-tls", "0.0.0.0:8443",
              "--admin", "/run/xenia/admin.sock", "--access-log", "-"]
    stop_grace_period: 30s
    read_only: true
    tmpfs: ["/run/xenia:uid=65532,gid=65532"]
    restart: unless-stopped
```

With `read_only`, `/run/xenia` needs a tmpfs of its own, owned by uid 65532, for the admin socket; nothing else is written.

## Kubernetes

```yaml
containers:
  - name: xenia
    image: ofkindness/xenia:0.15.1
    ports: [{ containerPort: 8080 }]
    volumeMounts:
      - { name: config, mountPath: /etc/xenia, readOnly: true }
      - { name: run, mountPath: /run/xenia }
    securityContext:
      runAsNonRoot: true
      runAsUser: 65532
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities: { drop: [ALL] }
    readinessProbe: { httpGet: { path: /, port: 8080 } }
volumes:
  - { name: config, configMap: { name: xenia-config } }
  - { name: run, emptyDir: {} }
terminationGracePeriodSeconds: 30
```

A changed ConfigMap reaches the mounted files after a while, but xenia reloads only when told: `kubectl exec ... -- xenia ctl reload`, or roll the pods. Behind an ingress or load balancer, list it in `server.trustedProxies` (see [Deployment](deployment.md#behind-a-load-balancer)).

## Verifying the image

Each release's image is signed with [Sigstore](https://www.sigstore.dev) by the workflow that publishes it, and carries an SPDX SBOM of xenia and the libraries in it as an attestation. With [cosign](https://docs.sigstore.dev/cosign/system_config/installation/):

```sh
cosign verify ofkindness/xenia:0.15.1 \
  --certificate-identity https://github.com/xenia-platform/xenia/.github/workflows/publish.yml@refs/heads/main \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
cosign verify-attestation --type spdxjson ofkindness/xenia:0.15.1 \
  --certificate-identity https://github.com/xenia-platform/xenia/.github/workflows/publish.yml@refs/heads/main \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

## Tags

`X.Y.Z` for a release, `X.Y` for the newest of its minor version, and `latest` for the newest release. Pin `X.Y.Z` in production.
