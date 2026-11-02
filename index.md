# Xenia

A single-binary web server and reverse proxy, configured in JavaScript. Version 0.16.0.

- **Static routes and files:** bodies from the config, directories from the disk, with validators, ranges, precompressed copies and gzip.
- **Reverse proxy:** pooling, balancing, health checks, upstream hostnames, WebSockets, TLS to upstreams, and a shared response cache that serves stale while it refreshes.
- **TLS and HTTP/2:** virtual hosts, client certificates, revocation lists, HSTS.
- **Limits per client** (rates, requests in flight, connections), **metrics, request IDs, JSON access logs** and an admin interface.
- **The config is a JavaScript module** compiled into an immutable snapshot. A reload swaps the snapshot atomically: requests in flight finish on the one they began with, and the request path takes no locks.

```js
// xenia.conf.js
import { defineConfig } from 'xenia';

export default defineConfig({
  routes: [
    { path: '/', body: 'hello\n' },
    { prefix: '/static/', root: 'public' },
    { prefix: '/api/', proxy: 'api' },
  ],
  upstreams: { api: { servers: ['127.0.0.1:3000'] } },
});
```

New to xenia? [Getting started](getting-started.md) goes from an empty directory to a site over HTTPS in about ten minutes, and the [recipes](recipes.md) are whole configs for common setups.

## Installing

The [Docker image](docker.md), for amd64 and arm64:

```sh
docker run -d -p 80:8080 -v "$PWD:/etc/xenia:ro" ofkindness/xenia:0.16.0
```

On Debian or Ubuntu, the package `xenia-server` from [the releases](https://github.com/xenia-platform/releases/releases), a systemd service with an example project in `/etc/xenia` (see [Deployment](deployment.md#packages)):

```sh
sudo apt install ./xenia-server_0.16.0_amd64.deb    # or _arm64
```

On FreeBSD 14 or later (amd64), the package `xenia-server`, the rc.d service `xenia`:

```sh
pkg add ./xenia-server-0.16.0-freebsd-amd64.pkg && sysrc xenia_enable=YES && service xenia start
```

With Homebrew, on macOS or Linux:

```sh
brew install xenia-platform/tap/xenia-server
brew services start xenia-server                       # on 127.0.0.1:8080
```

Or an archive from [the releases](https://github.com/xenia-platform/releases/releases), with the binary, the license, an example project and a systemd unit:

| Archive | Runs on |
| --- | --- |
| `xenia-0.16.0-linux-x86_64.tar.gz`, `xenia-0.16.0-linux-arm64.tar.gz` | any Linux on that architecture: a static binary, no C library or package needed |
| `xenia-0.16.0-macos-arm64.tar.gz` | macOS 13 or later on Apple silicon |
| `xenia-0.16.0-freebsd-amd64.tar.gz` | FreeBSD 14 or later on amd64 |

```sh
sha256sum -c --ignore-missing SHA256SUMS    # shasum -a 256 -c on macOS
tar -xzf xenia-0.16.0-linux-x86_64.tar.gz
./xenia-0.16.0-linux-x86_64/xenia --version
```

The macOS binary is signed ad hoc, so a copy downloaded with a browser is quarantined: `xattr -d com.apple.quarantine xenia` lets it run.

### Verifying a release

The image and each release's `SHA256SUMS` are signed with [Sigstore](https://www.sigstore.dev) by the workflow that publishes them, and each release has an SPDX SBOM of xenia and the libraries in it. With [cosign](https://docs.sigstore.dev/cosign/system_config/installation/) 3 or later, which finds the image's signature where cosign 2 doesn't look:

```sh
cosign verify ofkindness/xenia:0.16.0 \
  --certificate-identity https://github.com/xenia-platform/xenia/.github/workflows/publish.yml@refs/heads/main \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com

cosign verify-blob SHA256SUMS --bundle SHA256SUMS.sigstore.json \
  --certificate-identity https://github.com/xenia-platform/xenia/.github/workflows/publish.yml@refs/heads/main \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
sha256sum -c --ignore-missing SHA256SUMS
```

## Documentation

| | |
| --- | --- |
| [Getting started](getting-started.md) | from an empty directory to a site over HTTPS with Docker Compose: files, an API, TLS, a cache, reloads |
| [Recipes](recipes.md) | whole, tested configs: a single-page app, a precompressed static site, WebSockets, behind a load balancer, a rollout with feature flags |
| [Configuration](configuration.md) | the config file and every option: routes, files, compression, upstreams and proxying, names, health checks, WebSockets, virtual hosts, feature flags, limits, caching, TLS, client certificates, HTTP/2 |
| [Running](running.md) | the command line, running as a user and under systemd, shutdown, signals, the admin interface, access logs, metrics, request IDs |
| [Docker](docker.md) | the image: ports, mounting a config, HTTPS, reloads, stopping, health, compose and Kubernetes |
| [Deployment](deployment.md) | putting xenia on a host: users, ports below 1024, file permissions, the systemd unit, log rotation, behind a load balancer |
| [Upgrading](upgrading.md) | what each release changes that a config or an operator has to know or act on |
| [Design](design.md) | the model behind xenia: snapshots, the data and control planes, memory and concurrency |
| [Changelog](changelog.md) | what changed in each release |

Questions and bugs go to [the issues](https://github.com/xenia-platform/releases/issues); vulnerabilities, privately, as [the security policy](https://github.com/xenia-platform/releases/blob/main/SECURITY.md) says.

## License

Xenia is free to use, commercially or not, under the [Xenia License](https://github.com/xenia-platform/releases/blob/main/LICENSE): run it anywhere, and pass on unmodified copies of the binaries and images. Its source is not published.

## Third-Party Code

Xenia contains these libraries, each an unmodified upstream release under its own license. Their license texts are in [THIRD_PARTY_LICENSES](https://github.com/xenia-platform/releases/blob/main/THIRD_PARTY_LICENSES), in each archive and under `/licenses` in the image.

| Library | Version | License | For |
| --- | --- | --- | --- |
| [libuv](https://libuv.org) | 1.53.0 | MIT | event loops and I/O |
| [mbedtls](https://www.trustedfirmware.org/projects/mbed-tls/) | 4.1.1 | Apache-2.0 OR GPL-2.0-or-later | TLS and cryptography (used under Apache-2.0) |
| [mimalloc](https://github.com/microsoft/mimalloc) | 3.5.3 | MIT | the allocator, in the static Linux binaries only |
| [quickjs-ng](https://quickjs-ng.github.io/quickjs/) | 0.17.0 | MIT | the JavaScript engine that compiles configs |
| [zlib](https://zlib.net) | 1.3.2 | Zlib | gzip compression |
