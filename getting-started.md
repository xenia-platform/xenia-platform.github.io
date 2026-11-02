# Getting started

From an empty directory to a site served over HTTPS, with Docker Compose: static files and an API, then HTTPS, a cache and reloads. It takes about ten minutes, and needs Docker with Compose, `curl` and `openssl`. The finished project is [`examples/getting-started`](https://github.com/xenia-platform/releases/blob/main/examples/getting-started/).

## 1. A site

Make a directory for the project, with a page in `www/`:

```sh
mkdir -p site/www && cd site
echo '<h1>Hello from xenia</h1>' > www/index.html
```

and `xenia.conf.js`, the config, beside it. It is JavaScript, run once when xenia starts or reloads, and it exports what to serve:

```js
import { defineConfig } from 'xenia';

export default defineConfig({
  routes: [
    { prefix: '/', root: 'www' }, // every path from the files in www/
  ],
});
```

`defineConfig` changes nothing; it lets an editor check the config as you type, once `xenia types` has written `xenia.d.ts` beside it (see [Types](configuration.md#types)).

## 2. Run it

`compose.yaml` runs the [image](docker.md) with the project mounted where it looks for its config, read-only:

```yaml
services:
  xenia:
    image: ofkindness/xenia:0.16.0
    ports: ["80:8080"]
    volumes: ["./:/etc/xenia:ro"]
    command: ["--listen", "0.0.0.0:8080", "--admin", "/run/xenia/admin.sock", "--access-log", "-"]
    read_only: true
    tmpfs: ["/run/xenia:uid=65532,gid=65532"]
    stop_grace_period: 30s
    restart: unless-stopped
```

```sh
docker compose up -d
curl http://localhost/                         # <h1>Hello from xenia</h1>
docker compose exec xenia xenia check          # the config, as xenia reads it
```

The image runs as user 65532, so the project must be readable by others (it is, as made above). Inside, xenia listens on 8080; Compose maps port 80 to it. The admin socket on the `tmpfs` is how `xenia ctl` and the image's health check reach it.

Files are read from the disk when they are asked for: change `www/index.html`, and the next request has it, with no reload. A request for a file that is not there is `404`.

## 3. An API

Add an app for xenia to proxy: [whoami](https://github.com/traefik/whoami), a small server that answers with the request it got. In `compose.yaml`, under `services:`:

```yaml
  api:
    image: traefik/whoami:v1.11.0@sha256:200689790a0a0ea48ca45992e0450bc26ccab5307375b41c84dfc4f2475937ab
    restart: unless-stopped
```

and in `xenia.conf.js`, an upstream group and a route for `/api/` before the files:

```js
  upstreams: {
    api: { servers: ['api:80'] },
  },
  routes: [
    { prefix: '/api/', proxy: 'api' },
    { prefix: '/', root: 'www' },
  ],
```

```sh
docker compose up -d                           # starts api
docker compose kill -s HUP xenia               # reloads the config
curl http://localhost/api/hello
```

whoami's answer shows what xenia sent it: `GET /api/hello`, `X-Forwarded-For` with your address, `X-Forwarded-Proto: http`. `api:80` is the Compose service's name: xenia looks it up, and follows it when it changes.

A reload compiles the changed config and switches to it without dropping a connection; one that fails, with a typo say, keeps the old config serving and logs why (`docker compose logs xenia`). See [Upstreams and proxying](configuration.md#upstreams-and-proxying).

## 4. HTTPS

A certificate for `localhost`, made for this machine alone: browsers will warn about it, which is expected. For a real name, use one from your CA or ACME client in its place (see [TLS](configuration.md#tls)).

```sh
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 -nodes -days 30 \
  -subj /CN=localhost -addext subjectAltName=DNS:localhost -keyout site.key -out site.pem
chmod 644 site.key                             # for the image's user; keep real keys tighter
```

In `xenia.conf.js`:

```js
  tls: {
    certificates: [{ cert: 'site.pem', key: 'site.key' }],
    redirect: true,     // plain HTTP gets 308 to https://
    redirectPort: 443,  // the port clients reach; xenia listens on 8443
  },
```

In `compose.yaml`, an HTTPS listener and its port:

```yaml
    ports: ["80:8080", "443:8443"]
    command: ["--listen", "0.0.0.0:8080", "--listen-tls", "0.0.0.0:8443",
              "--admin", "/run/xenia/admin.sock", "--access-log", "-"]
```

```sh
docker compose up -d                           # a new command: the container is recreated
curl -k https://localhost/                     # -k: the certificate is self-signed
curl -I http://localhost/api/hello             # 308, location: https://localhost/api/hello
```

xenia serves HTTP/2 over TLS to clients that ask for it, and HTTP/1.1 to the rest. A certificate is read when the config compiles: to renew one, replace the files and reload.

## 5. A cache

The API's answers can be kept for a while, so repeats don't reach the app. In `xenia.conf.js`, a cache, and the API route using it:

```js
  caches: {
    pages: { size: 64 * 1024 * 1024 },
  },
  routes: [
    { prefix: '/api/', proxy: 'api', cache: 'pages', ttl: 10 },
    { prefix: '/', root: 'www' },
  ],
```

```sh
docker compose exec xenia xenia ctl reload     # reloads, and waits to say how it went
curl -sk -D- -o /dev/null https://localhost/api/hello | grep x-cache   # x-cache: miss
curl -sk -D- -o /dev/null https://localhost/api/hello | grep x-cache   # x-cache: hit
```

`ttl: 10` keeps an answer for 10 seconds when the app says nothing of its own about caching; `Cache-Control` from the app wins. See [Caching](configuration.md#caching).

## The whole project

`compose.yaml`:

<!-- file: examples/getting-started/compose.yaml -->
```yaml
# The project docs/getting-started.md builds: xenia in front, serving www/
# and an API, over HTTP and HTTPS. Run it here with: docker compose up -d
services:
  xenia:
    image: ofkindness/xenia:0.16.0
    ports: ["80:8080", "443:8443"]
    volumes: ["./:/etc/xenia:ro"]
    command: ["--listen", "0.0.0.0:8080", "--listen-tls", "0.0.0.0:8443",
              "--admin", "/run/xenia/admin.sock", "--access-log", "-"]
    read_only: true
    tmpfs: ["/run/xenia:uid=65532,gid=65532"]
    stop_grace_period: 30s
    restart: unless-stopped
    depends_on: [api]
  api:
    # A small app that answers with the request it got.
    image: traefik/whoami:v1.11.0@sha256:200689790a0a0ea48ca45992e0450bc26ccab5307375b41c84dfc4f2475937ab
    restart: unless-stopped
```

`xenia.conf.js`:

<!-- file: examples/getting-started/xenia.conf.js -->
```js
// The project docs/getting-started.md builds: the site in www/, an API
// behind /api/ with a cache, over HTTPS.

import { defineConfig } from 'xenia';

export default defineConfig({
  tls: {
    certificates: [{ cert: 'site.pem', key: 'site.key' }],
    redirect: true, // plain HTTP gets 308 to https://
    redirectPort: 443, // the port clients reach; xenia listens on 8443
  },
  upstreams: {
    // The compose service's name, which xenia looks up and follows.
    api: { servers: ['api:80'] },
  },
  caches: {
    pages: { size: 64 * 1024 * 1024 },
  },
  routes: [
    { prefix: '/api/', proxy: 'api', cache: 'pages', ttl: 10 },
    { prefix: '/', root: 'www' },
  ],
});
```

`docker compose logs -f xenia` shows the access log, a JSON line per request. `docker compose down` stops it all: xenia finishes the requests in progress first, for up to its shutdown timeout.

## Where next

- [Recipes](recipes.md): whole configs for a single-page app, a static site, WebSockets, a load balancer in front, and a rollout with feature flags.
- [Configuration](configuration.md): every field.
- [Docker](docker.md): the image, its health check, Kubernetes, and verifying it.
- [Deployment](deployment.md): packages, systemd, and running without containers.
- [Running](running.md): the command line, the admin interface, logs and metrics.
- [Upgrading](upgrading.md): what each release changes.
