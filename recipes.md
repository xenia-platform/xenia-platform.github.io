# Recipes

Whole configs for common setups, each in its own folder under [`examples/recipes`](https://github.com/xenia-platform/releases/blob/main/examples/recipes/) with the files it serves. Each one is served and checked by the tests, so what a recipe says it does, it does. Copy the folder, change the addresses and paths, and run it:

```sh
cd examples/recipes/spa
xenia check                          # compile, validate, list the routes
xenia --listen 0.0.0.0:8080          # serve it; xenia.conf.js is the default
```

Upstream addresses such as `127.0.0.1:3000` stand for your own backends. Paths are relative to the config file. See [Configuration](configuration.md) for every field.

## A single-page app with an API

An app built into `dist/`: hashed assets, an API, and the client-side routes the app draws itself. A link to `/settings/profile`, or a reload there, must get `index.html` too, or the browser shows a `404`.

<!-- file: examples/recipes/spa/xenia.conf.js -->
```js
// A single-page app with an API: the built assets, the API behind /api/,
// and index.html for every other path, so the app's own routes load from a
// link or a reload.
//
//   xenia check && xenia --listen 0.0.0.0:8080

import { defineConfig } from 'xenia';

export default defineConfig({
  upstreams: {
    api: { servers: ['127.0.0.1:3000'] },
  },
  routes: [
    // Built with hashed names (app.3f9c1e.js), so they can be kept for good.
    {
      prefix: '/assets/',
      root: 'dist/assets',
      headers: { 'cache-control': 'public, max-age=31536000, immutable' },
    },
    { prefix: '/api/', proxy: 'api' },
    // Everything else is the app, which reads the path and draws the page.
    // no-cache: browsers check for a new index.html, which names new assets.
    { prefix: '/', file: 'dist/index.html', headers: { 'cache-control': 'no-cache' } },
  ],
});
```

- **`/assets/`** serves the built files. Their names change whenever their content does, so `immutable` lets browsers keep them for a year.
- **`/api/`** goes to the backend.
- **`{ prefix: '/', file: 'dist/index.html' }`** is the fallback: every other path gets the app's page, as `text/html` from the file's name. Longer prefixes win, so `/assets/` and `/api/` keep their own routes; a missing asset is a real `404`, not the page. See [Single files](configuration.md#single-files).
- **`no-cache`** on the page makes browsers ask again each time, with its tag, so a deploy that names new assets is seen at once, at the cost of a `304`.

The whole recipe: [`examples/recipes/spa`](https://github.com/xenia-platform/releases/blob/main/examples/recipes/spa/).

## A static site, compressed ahead

A site whose files are compressed when it is built, at the slowest and best levels, instead of on every request:

```sh
cd site && find . -type f \( -name '*.html' -o -name '*.css' -o -name '*.js' -o -name '*.svg' \) \
  -exec brotli -kf -q 11 {} \; -exec gzip -kf -9 {} \;
```

<!-- file: examples/recipes/static-site/xenia.conf.js -->
```js
// A static site, compressed when it is built: each file beside its .br and
// .gz copies (brotli -k, gzip -k), sent to the clients that take them, at
// levels too slow to spend on every request.
//
//   xenia check && xenia --listen 0.0.0.0:8080

import { defineConfig } from 'xenia';

export default defineConfig({
  routes: [
    { path: '/robots.txt', file: 'site/robots.txt', headers: { 'cache-control': 'max-age=86400' } },
    {
      prefix: '/',
      root: 'site',
      precompressed: true,
      headers: { 'cache-control': 'max-age=300' },
    },
  ],
});
```

- **`precompressed: true`** sends `style.css.br` to clients that take `br`, `style.css.gz` to those that take `gzip`, and `style.css` to the rest, with `vary: accept-encoding`. It is the only way xenia sends `br`. See [Precompressed files](configuration.md#files).
- **`/robots.txt`** is a [single file](configuration.md#single-files) with a longer cache life than the pages.
- A file without copies beside it is gzipped on the fly when it is worth it, as on any file route.

The whole recipe: [`examples/recipes/static-site`](https://github.com/xenia-platform/releases/blob/main/examples/recipes/static-site/).

## A WebSocket backend

An app whose pages are static and whose live part is a WebSocket:

<!-- file: examples/recipes/websocket/xenia.conf.js -->
```js
// An app that talks over WebSockets: its socket endpoint proxied, the rest
// of the site static. Any proxy route passes an upgrade through; what is
// worth setting is how long a quiet socket stays open, and how many one
// client may hold.
//
//   xenia check && xenia --listen 0.0.0.0:8080

import { defineConfig } from 'xenia';

export default defineConfig({
  upstreams: {
    // A socket with nothing on it for ten minutes is closed.
    app: { servers: ['127.0.0.1:3000'], tunnelIdleTimeout: 600 },
  },
  limits: {
    // An open socket is a request in flight: 4 at once per client.
    sockets: { inFlight: 4 },
  },
  routes: [
    { prefix: '/ws/', proxy: 'app', limit: 'sockets' },
    { prefix: '/', root: 'public' },
  ],
});
```

- **Any proxy route** passes a WebSocket upgrade through; nothing turns it on. See [WebSockets](configuration.md#websockets).
- **`tunnelIdleTimeout`** closes a socket that has carried nothing for that long, in seconds. The default is 300.
- **`inFlight`** counts an open socket as a request in flight until it closes, so a client can hold at most 4; the fifth gets `429`. See [Limits](configuration.md#limits).

The whole recipe: [`examples/recipes/websocket`](https://github.com/xenia-platform/releases/blob/main/examples/recipes/websocket/).

## Behind a load balancer

xenia behind a load balancer or a CDN that ends TLS, and connects from its own addresses:

<!-- file: examples/recipes/behind-load-balancer/xenia.conf.js -->
```js
// Behind a load balancer or a CDN that ends TLS and connects from
// 10.0.0.0/8. Its X-Forwarded-For names each client: for the limits, the
// access log, and the X-Forwarded-For that goes upstream.
//
//   xenia check && xenia --listen 0.0.0.0:8080

import { defineConfig } from 'xenia';

export default defineConfig({
  server: {
    trustedProxies: ['10.0.0.0/8'],
    // No maxConnectionsPerClient: every connection comes from the
    // balancer, which it would not count anyway. The limit below does.
  },
  upstreams: {
    app: { servers: ['127.0.0.1:3000'] },
  },
  limits: {
    // Per client, not per balancer: 20 a second, bursts of 40, and at most
    // 10 requests in flight at once.
    perClient: { rate: 20, burst: 40, inFlight: 10 },
  },
  routes: [
    { path: '/healthz', body: 'ok\n' }, // for the balancer's health checks
    { prefix: '/', proxy: 'app', limit: 'perClient' },
  ],
});
```

- **`trustedProxies`** names the balancer's addresses. Requests from them are taken as from the client their `X-Forwarded-For` names; requests from anywhere else, at their own address, whatever they send. See [Behind a load balancer](deployment.md#behind-a-load-balancer).
- **The limit** counts real clients: without `trustedProxies`, the balancer would be one client, and all of them would share its budget.
- **`maxConnectionsPerClient`** is left out: it only knows a connection's own address, and it never counts trusted proxies.
- **`/healthz`** answers the balancer's health checks without reaching the app.

The whole recipe: [`examples/recipes/behind-load-balancer`](https://github.com/xenia-platform/releases/blob/main/examples/recipes/behind-load-balancer/).

## A gradual rollout with feature flags

Moving from one backend to its replacement a route at a time, and back if it misbehaves:

<!-- file: examples/recipes/rollout/xenia.conf.js -->
```js
// Moving from one backend (blue) to its replacement (green) a route at a
// time, with feature flags. Each switch is live, needs no reload, and goes
// back the same way. Flags switch whole routes: there are no percentages.
//
//   xenia --listen 0.0.0.0:8080 --admin /run/xenia/admin.sock
//   xenia ctl flags green-export=on    exports go to green first
//   xenia ctl flags green=on           then the rest of /api/
//   xenia ctl flags green=off          ...and back, if green misbehaves

import { defineConfig } from 'xenia';

export default defineConfig({
  upstreams: {
    blue: { servers: ['127.0.0.1:3001'] },
    green: { servers: ['127.0.0.1:3002'] },
  },
  flags: { 'green-export': false, green: false },
  routes: [
    // While the flag is off, this route is not there, and /api/ has export.
    { prefix: '/api/export/', proxy: 'green', flag: 'green-export' },
    { prefix: '/api/', proxy: 'green', flag: 'green' },
    { prefix: '/api/', proxy: 'blue', flag: '!green' },
  ],
});
```

- **Routes with a flag** are there only while it is on (`'green'`) or off (`'!green'`). Turning `green-export` on sends `/api/export/` to green, while the rest of `/api/` stays on blue; `green` then moves the rest.
- **`xenia ctl flags NAME=on`** switches one live, on every worker at once, without a reload; `=off` switches it back, and `=reset` returns it to the config's value. It needs `--admin`. See [Feature flags](configuration.md#feature-flags).
- **A live switch survives reloads, but not a restart.** Once green has taken over, write `green: true` in the config, or remove blue.
- **Flags switch whole routes:** there is no sending a share of the requests to each. To shift by weight, change the servers in the upstream groups instead, or drain them (`xenia ctl drain`).

The whole recipe: [`examples/recipes/rollout`](https://github.com/xenia-platform/releases/blob/main/examples/recipes/rollout/).
