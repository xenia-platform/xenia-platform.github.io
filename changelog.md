# Changelog

What changed for people who run xenia, newest first. Each release's notes, with the details, are its tag message and its page on [the releases repo](https://github.com/xenia-platform/releases/releases).

## 0.15.1

The first public release; 0.15.0 was not published.

- **HTTP/2:** reading pauses at the request that would pile answers up, not at the end of the 4 KiB of input it was in. A client that sent more than 128 requests at once without reading could have some of them refused; now every one is answered.

## 0.15.0

Distribution: a Docker image, signed releases, and a license for the binaries.

- **Docker image** `ofkindness/xenia` for amd64 and arm64, from the same static binary as the archives: no shell, runs as uid 65532 on 8080/8443, config in `/etc/xenia`, a healthcheck through the admin socket, the access log on stdout. See [docs/docker.md](docker.md).
- **`--access-log -`** writes the access log to standard output.
- **Signed releases:** the image and `SHA256SUMS` are signed with Sigstore (cosign, keyless), and each release has an SPDX SBOM of xenia and every library in it.
- **License:** the Xenia License: free to use, commercially or not; unmodified binaries and images may be passed on. The source is not published.
- **Where to find it:** releases, issues and the security policy at [xenia-platform/releases](https://github.com/xenia-platform/releases), the docs at [xenia-platform.github.io](https://xenia-platform.github.io).

## 0.14.0

Operability: running as a service.

- **`--user USER[:GROUP]`:** started as root, xenia binds its ports, then becomes that user for good, before anything else runs.
- **systemd:** readiness, reloads and stopping are reported on `NOTIFY_SOCKET` (`Type=notify-reload`), and `contrib/systemd/xenia.service` is a unit to start from.
- **Behind load balancers:** the client `server.trustedProxies` name is the `X-Forwarded-For` sent upstream and the access log's `client`, with the proxy in `peer`.
- **Responses:** `x-content-type-options: nosniff` on everything xenia makes itself; file ETags include the inode.
- **HTTP/2:** a second GOAWAY never names a later stream than the first.
- **Docs** split into configuration, running, deployment and design guides.

## 0.13.0

Hardening: every high and medium finding of a full review.

- **HTTP/2:** resets xenia sends count against the reset budget (CVE-2025-8671, "MadeYouReset"); reading pauses while output piles up; streams stalled on a shut window are reset; frames after a reset are ignored; even stream ids and control characters in values are refused.
- **Signals** never end the process during startup or a drain; reloads during name lookups wait their turn.
- **TLS:** a request without a host needs the default certificate.
- **Proxy:** a client's own `X-Forwarded-*`, `Forwarded` and `X-Real-IP` never go upstream; cache keys include the scheme; verified client certificates bypass the cache; strict chunk sizes.
- **Admin:** per-listener connection budgets, a stricter token file, escaped log lines, no symbolic links for the access log.

## 0.12.0

Project configs.

- The config lives in the project as `xenia.conf.js`, found where xenia starts, and imports the project's own `./` and `../` modules.
- `xenia types` writes `xenia.d.ts` for editors to check configs with; `xenia check` validates one.

## 0.11.0

Serving stale (RFC 5861).

- `stale-while-revalidate`: stale entries are served at once while one background refresh renews them.
- `stale-if-error`: a kept entry answers when the upstream fails.
- Route-level `staleWhileRevalidate` and `staleIfError` where responses say nothing.

## 0.10.0

Upstream hostnames.

- Servers can be `host:port`; every address a name resolves to is a server of its own, refreshed every `resolveInterval`.
- Names that don't resolve never fail a config; `xenia ctl resolve` and drains by name.

## 0.9.0

Response caching for proxy routes.

- Named in-memory caches (RFC 9111, as a shared cache), with `Vary`, revalidation, request coalescing across workers, gzip copies and purging (`xenia ctl cache purge`).

## 0.8.0

Rate limits.

- Named limits (`rate`, `burst`) per client, shared by routes that name them, exact across workers; `429` with `retry-after`; clients behind `server.trustedProxies` taken from `X-Forwarded-For`.

## 0.7.0

Compression.

- gzip on the fly for static, file and (opt-in) proxied responses, with weak tags and `vary: accept-encoding`.

## 0.6.0

HTTP/2 for clients.

- On TLS listeners by ALPN and on plain ones with prior knowledge, for every kind of route, with flow control and limits against floods.

## 0.5.0

Observability.

- Prometheus metrics (`--metrics`), request IDs (`x-request-id`), and the admin interface over TCP with a token.

## 0.4.0

HTTP caching for files.

- Strong ETags, `Last-Modified`, conditional requests (`304`, `412`) and single ranges (`206`, `416`).

## 0.3.0

Static files.

- File routes: `root`, `index`, `listing`, `precompressed`, `followSymlinks`, `contentTypes`, with safe paths and a small-file cache.

## 0.2.0

The first release.

- Static Linux binaries (musl, hardened) and a macOS build.
- Routes from immutable snapshots compiled from a JavaScript config and swapped atomically on `SIGHUP`.
- Reverse proxy with pooling, balancing, health checks, rewriting and WebSockets; TLS termination with virtual hosts, client certificates and revocation lists; TLS to upstreams; an admin interface and JSON access logs.
