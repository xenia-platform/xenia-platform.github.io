# Changelog

What changed for people who run xenia, newest first. Each release's notes, with the details, are its tag message and its page on [the releases repo](https://github.com/xenia-platform/releases/releases).

## 0.16.0

Trust, packages for Debian, FreeBSD and Homebrew, and the gaps users hit first: what 0.15.1 got wrong in the open, conformance, single-file routes, limits on connections and requests in flight, and docs to start from.

- **Docs:** [Getting started](getting-started.md), from an empty directory to a site over HTTPS with Docker Compose, run for real before each release; [Recipes](recipes.md), whole configs for a single-page app, a precompressed static site, WebSockets, a load balancer in front and a rollout with feature flags, each served and checked by the tests; and [Upgrading](upgrading.md), what each release changes for configs and operators, from the new "Upgrading" part of the release's notes here.
- **RSA keys in PKCS#8 form** (`-----BEGIN PRIVATE KEY-----`, what `openssl req -newkey rsa:2048` writes) now load, in `tls.certificates` and as an upstream's `tls.key`. Every release with TLS refused them as "the key does not match the certificate", because Mbed TLS cannot compare such a key with its certificate; xenia now compares the two public keys itself, and still refuses a key that is not the certificate's. The same key in PKCS#1 form (`BEGIN RSA PRIVATE KEY`) and EC keys always worked.
- **Single-file routes:** `{ path: '/robots.txt', file: 'public/robots.txt' }` serves one file, with everything a file route does: validators, ranges, precompressed copies, gzip and the small-file cache. With a `prefix`, it serves the file for every path under it, the fallback a single-page app needs: `{ prefix: '/', file: 'dist/index.html' }`, while longer prefixes keep their own routes. See [Single files](configuration.md#single-files).
- **Requests in flight per client:** a limit can bound how many requests a client has open at once on its proxy and file routes, `inFlight`, with or without a `rate`; one more gets `429`. Behind `trustedProxies` the client is the one `X-Forwarded-For` names; a WebSocket counts until it closes. "Rate limits" in the docs is now [Limits](configuration.md#limits).
- **A client that leaves while it waits for the upstream** has given up: when it closes its connection, or only its side of it, after its request is sent and before the response begins, xenia ends the exchange and closes the upstream connection, as nginx does, instead of keeping the upstream at work for nobody. Its access log line says `aborted`. Before, a client that closed only its side still got the response; it still does once the response has begun.
- **Connections per client:** `server.maxConnectionsPerClient` closes a client's connections past that many as soon as they are accepted. It counts the connection's own address, and leaves trusted proxies out. Off unless set.
- **FreeBSD**, 14 or later, amd64: an archive and a package, `xenia-server-VERSION-freebsd-amd64.pkg`, in each release, covered by `SHA256SUMS`. The package installs an rc.d script (`sysrc xenia_enable=YES`, `service xenia start`) that runs xenia on port 80 under `daemon(8)`, serving as the user `xenia`, and the example project in `/usr/local/etc/xenia`. `--user` sets no new privileges there too, as on Linux. See [Deployment](deployment.md#packages).
- **Debian and Ubuntu packages.** Each release has `xenia-server_VERSION_amd64.deb` and `_arm64.deb`, covered by the signed `SHA256SUMS`. A package installs the static binary as `/usr/bin/xenia`, the systemd unit, the example project in `/etc/xenia` (conffiles, root:xenia), the user `xenia` and a logrotate entry, and enables and starts the service. `apt purge` removes the config and logs, and keeps the user. See [Deployment](deployment.md#packages).
- **Homebrew**, on macOS and Linux: `brew install xenia-platform/tap/xenia-server`, then `brew services start xenia-server` to run it on 127.0.0.1:8080.
- **Stricter HTTP/1.1, against request smuggling.** Comparing xenia's parser with an independent one (llhttp, in a new fuzz target) found input xenia took that stricter parsers refuse, which a proxy in front could read differently:
  - a second `Content-Length`, even with the same value, is `400`;
  - `Transfer-Encoding` with empty list elements (`, chunked`) is `501`: only a lone `chunked` is chunked;
  - a tab in `Transfer-Encoding` (`chunked\t`) is `400`: whitespace to xenia, another coding to some;
  - a chunk size takes no whitespace after it, and chunk extensions must follow their grammar (`;name`, `;name=token` or `;name="quoted"`);
  - a trailer is `name: value`, and never `Content-Length`, `Transfer-Encoding` or `Connection`;
  - an absolute-form target with userinfo (`http://user@host/`) is `400`.

  The chunked rules hold for upstream responses too.
- **HTTP/2 conformance:** h2spec runs against every release and every change to the HTTP/2 code. All 146 cases pass over TLS; with prior knowledge, all but one, on purpose (see [HTTP/2](configuration.md#http2)).
- **Cache:** when the client whose miss started a fill leaves while others wait for it, the fill goes on without it, and they get its answer; before, it was dropped, and one of them went upstream again. Counted as `xenia_cache_fills_continued_total`. `/snapshot` also counts the requests that waited for another's fill (`waited`).
- **`tls.redirectPort`:** the port HTTPS redirects send clients to, when it isn't the one xenia listens on: behind `-p 443:8443`, `redirectPort: 443`. Without it, as before, the first `--listen-tls` port. It changes with a reload; `xenia check` and `GET /snapshot` show it.
- **Docs:** the site's title is Xenia; verifying needs cosign 3 or later, which the docs now say; the README and the release notes say how to run a macOS binary downloaded with a browser.
- **Releases repo:** it carries `THIRD_PARTY_LICENSES`, which its README and security policy point to.
- **Docs:** every whole config is written with `defineConfig`, so editors check it as it is copied; a bare `export default { … }` still works.
- **Publishing:** the docs site is checked once it is live: every page, link and anchor, and the links into the releases repo.
- **Proxy, HTTP/2 uploads:** a worker could crash when a write to the client failed to start (out of memory, a TLS error) just as a request body went upstream: the window update for the body closed the connection, and the proxy went on with the exchange that closing had ended. Found by the HTTP/2 fuzz target.

### Upgrading

- **HTTP/1.1 is stricter, against request smuggling.** Requests 0.15 took are now refused: a second `Content-Length`, even with the same value (`400`); `Transfer-Encoding` with empty list elements (`501`) or a tab (`400`); whitespace after a chunk size, and chunk extensions outside their grammar; trailers that are not `name: value`, or that name `Content-Length`, `Transfer-Encoding` or `Connection`; and absolute-form targets with userinfo (`400`). Clients that follow the standard send none of these: if one of yours starts getting `400` or `501`, its framing is why. The chunked rules hold for upstream responses too.
- **A client that leaves while it waits for the upstream ends the exchange.** When a client closes its connection, or only its side of it, after its request and before the response begins, xenia now closes the upstream connection too. Upstreams see such requests cancelled, and each has an access log line with `"status": null` and `"aborted": true`. A client that half-closes on purpose after sending its request no longer gets an answer; once the answer has begun, it still does.
- **Redirects behind port mapping.** With `tls.redirect`, redirects name the port of the first `--listen-tls`, as before. Behind a mapping such as `-p 443:8443`, set the new `tls.redirectPort: 443` so they name the port clients reach.
- **`GET /snapshot`.** A limit's `rate` is `null` when it has only `inFlight`, and the JSON has new keys: `file` on file routes, `inFlight` and its counts on limits, `maxConnectionsPerClient` in the server settings. A script that reads it should expect them.
- **Docs.** "Rate limits" is now [Limits](configuration.md#limits); links to `#rate-limits` still land there.
- **From a release archive to the Debian package.** The package puts the binary in `/usr/bin/xenia` and its unit in `/usr/lib/systemd/system`. The archive's install steps put them in `/usr/local/bin/xenia` and `/etc/systemd/system/xenia.service`, and a unit there wins over the package's, so it would keep starting the old binary. Before installing the package, stop the service and remove both, keeping any listener changes as a `systemctl edit xenia` override. `/etc/xenia` stays as it is: dpkg asks before it replaces a config file you already have, and keeps yours unless told otherwise.

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
