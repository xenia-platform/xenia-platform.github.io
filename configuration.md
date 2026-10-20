# Configuration

A configuration is a JavaScript module that lives in the project it serves, as `xenia.conf.js`, and goes into version control with the rest of it. Its default export describes the whole routing state; it is compiled into typed commands, validated in C, and built into an immutable snapshot. Editors check it as it is written (see [Types](#types)). See [`examples/xenia.conf.js`](https://github.com/xenia-platform/releases/blob/main/examples/xenia.conf.js).

```js
import { defineConfig } from 'xenia';

const pages = ['about', 'contact'];

export default defineConfig({
  routes: [
    { path: '/', body: 'hello from xenia\n' },
    { path: '/old', status: 301, headers: { location: '/' } },
    ...pages.map((page) => ({
      path: `/${page}`,
      contentType: 'text/html; charset=utf-8',
      body: `<h1>${page}</h1>\n`,
    })),
  ],
});
```

| Field | Default | Rules |
| --- | --- | --- |
| `path` | one of `path` and `prefix` | exact match; starts with `/`; visible ASCII, no `?` or `#` |
| `prefix` | | matches every path that starts with it; starts and ends with `/` |
| `host` | none: the default site | serve only this host: a DNS name, or `*.` and one (see [Virtual hosts](#virtual-hosts)) |
| `method` | `'GET'` | an HTTP token; `HEAD` is answered from `GET` |
| `status` | `200` | 200–599, except 204, 205 and 304 |
| `contentType` | `text/plain; charset=utf-8` for strings, `application/octet-stream` for bytes | `''` sends no content type |
| `headers` | none | up to 32 extra response headers; names the server sets itself are refused. `X-Content-Type-Options` replaces xenia's `nosniff` |
| `body` | `''` | a string (sent as UTF-8), `Uint8Array` or `ArrayBuffer`; at most 1 MiB, 64 MiB per config |
| `proxy` | | forward to this upstream group instead (see [Upstreams and proxying](#upstreams-and-proxying)) |
| `rewrite` | none | proxy routes: replaces the part of the path the route matched (see [Path rewriting](#upstreams-and-proxying)) |
| `root` | | serve the files in this directory instead (see [Files](#files)); `index`, `listing`, `precompressed`, `followSymlinks` and `contentTypes` go with it |
| `compress` | `true`, `false` on proxy routes | gzip responses worth it for clients that take it (see [Compression](#compression)) |
| `allowHttp` | `false` | served over plain HTTP even with `tls.redirect` (see [TLS](#tls)) |
| `clientCert` | `false` | only to clients with a verified certificate, else `403` (see [Client certificates](#client-certificates)) |
| `flag` | none | served only while a feature flag is on (`'name'`) or off (`'!name'`) (see [Feature flags](#feature-flags)) |
| `limit` | none | count each client's requests against this rate limit (see [Rate limits](#rate-limits)) |

An exact `path` beats a `prefix`, and the longest prefix wins. Whichever path matches decides the outcome: a method it has no route for is `405`, not a shorter prefix. Matching is case-sensitive and takes the path as sent; `/a/../b` is not `/b`. Static and file routes take no request body (`413`).

**What xenia's own responses say.** Every response xenia makes itself (static routes, files and listings, errors, redirects, `429` and admin replies) carries `x-content-type-options: nosniff`, so browsers take its content type as given; a route's own `X-Content-Type-Options` replaces it. Proxied responses and cache hits are the upstream's as they came.

Unknown keys are errors, and every error names its place: `xenia.conf.js:12:5: TypeError: …` for JavaScript errors, `xenia.conf.js: routes[2].status: must be an integer` for the shape.

```sh
xenia check                           # compile and validate ./xenia.conf.js, list the routes
xenia                                 # serve it
kill -HUP $(pgrep -x xenia)           # reload after editing
xenia check site.js                   # another file: check it,
xenia --config site.js                # ...and serve it
```

Without `--config`, the server takes `xenia.conf.js` from the directory it starts in, and with none there a single built-in route, `GET /` answering `hello from xenia`; the log says which. `--check FILE` is `xenia check FILE`.

**What goes in the repository.** The config and the files it imports. Certificates, keys and the admin token are named by path and never written into it, so the config can be committed as it is while those files stay wherever secrets are kept; paths are relative to the config.

**Reloading.** On `SIGHUP` the server compiles the file again and replaces the whole routing state, keeping only the live changes made through the [admin interface](running.md#admin-interface). A config that fails to compile or validate is logged (`reload failed: …`) and the current snapshot keeps serving. Compiling never blocks request handling: it runs in a child process, `xenia --compile FILE`, which the server kills after 3 seconds, so a runaway or crashing config cannot take the server down.

**Imports.** A config can import `'xenia'`, whose `defineConfig` returns the config it is given, and the project's own files: `./` and `../` paths to `.js` or `.mjs` files that stay inside the config's directory. Paths are resolved by their text, and symbolic links in them are followed. Packages, `node:` modules and absolute paths cannot be imported. A reload reads every module again; the config and its modules together are limited to 1 MiB and 256 files.

```js
import { defineConfig } from 'xenia';
import { pages } from './routes/pages.js';

export default defineConfig({ routes: [{ path: '/', body: 'home\n' }, ...pages] });
```

**Determinism.** The same files always compile to the same snapshot, and the same live changes on top of it give the same one again. Configs run in a fresh JavaScript runtime each time, without `Date`, `performance`, `WeakRef` or `FinalizationRegistry`, and `Math.random` throws. Evaluation is limited to 64 MiB of memory and 2 seconds. [File routes](#files) are the one exception to what a snapshot holds: the route is in it, and the files are read when they are asked for.

## Types

`xenia types` writes `xenia.d.ts`, the types of every option with its default, unit and rules, and a `jsconfig.json` that has editors check `xenia.conf.js`, and what it imports, against them:

```sh
cd my-site && xenia types             # xenia.d.ts, and jsconfig.json if there is no project file
```

VS Code, and any editor with the TypeScript language server, then completes keys and shows each one's documentation, and marks a misspelled key, a value of the wrong type, and a route that mixes the keys of two kinds (`proxy` with `body`, `root` with `path`): before `xenia check` would. Names that refer to other parts of the config, such as a route's `proxy`, are `xenia check`'s to check. `defineConfig` is what lets them, by giving the export its type; it returns the config as it is. The same check runs anywhere TypeScript does:

```sh
npx -p typescript tsc -p jsconfig.json    # in CI, say
```

- **A project with its own** `tsconfig.json` or `jsconfig.json` gets no `jsconfig.json`: add `xenia.d.ts` and the config to its `include`. The one xenia writes uses only the ES2023 library, as configs have no DOM, Node or `Date`, and names `xenia.conf.js`; a config of another name goes in its `include`.
- **Upgrades.** The types come from the binary, so run `xenia types` again after upgrading; `xenia check` warns when the `xenia.d.ts` beside a config is another version's. Committing it gives editors the types on a fresh checkout, and `xenia check` says when it falls behind.
- Configs are JavaScript; TypeScript configs are not supported yet. A module's routes can say what they are with JSDoc, `/** @type {import('xenia').StaticRoute[]} */`, as [`examples/pages.js`](https://github.com/xenia-platform/releases/blob/main/examples/pages.js) does.

## Server settings

An optional `server` section sets timeouts and limits. It is part of the snapshot like the routes, so a reload changes it without a restart; connections that are already open follow the new values.

```js
export default {
  server: { headerTimeout: 10, idleTimeout: 30, writeTimeout: 30,
            maxConnections: 10000, shutdownTimeout: 10 },
  routes: [ ... ],
};
```

| Setting | Default | Meaning |
| --- | --- | --- |
| `headerTimeout` | 10 s | from accepting a connection, or from the first byte after a keep-alive gap, to a complete request head. A partial head gets `408 Request Timeout`; a connection that never sent anything is closed silently. Trickled bytes do not extend it. |
| `idleTimeout` | 30 s | the keep-alive gap between requests, and the wait for a client to close after the server's final response |
| `writeTimeout` | 30 s | a queued response must move at least 1 KiB per window, or the connection is aborted. Progress is measured into the kernel, so socket buffers of a few hundred KiB sit between it and the client. |
| `maxConnections` | 10000 | open connections per process; at the limit the server stops accepting and new connections wait in the kernel backlog. Capped to fit under the open-file limit, which is raised at startup. |
| `shutdownTimeout` | 10 s | how long a drain waits for in-flight responses |
| `maxRequestBody` | 16 MiB | bytes in a proxied request body: a larger `Content-Length` gets `413` at once, a chunked body as soon as it goes over |
| `trustRequestId` | `false` | keep a client's well-formed `X-Request-Id` instead of making one (see [Request IDs](running.md#request-ids)) |
| `http2` | `true` | serve [HTTP/2](#http2): `h2` by ALPN on TLS listeners, and with prior knowledge on plain ones |
| `trustedProxies` | none | up to 64 addresses or prefixes (`'10.0.0.0/8'`, `'::1'`) whose `X-Forwarded-For` names the client: for [rate limits](#rate-limits), the `X-Forwarded-For` sent upstream and the [access log](running.md#access-logs) |

Timeouts are seconds (fractions allowed, 0.05–3600) and are checked four times a second.

## Files

A route with a `root` serves the files in that directory, read from the disk when they are asked for, so a deploy that changes them needs no reload. The path after the route's prefix is the file's path below the root:

```js
{ prefix: '/static/', root: 'public' },             // /static/css/a.css is public/css/a.css
{ prefix: '/', root: '/srv/site', precompressed: true,
  headers: { 'cache-control': 'max-age=3600' } },
{ prefix: '/downloads/', root: 'dl', index: [], listing: true },
```

| Field | Default | Meaning |
| --- | --- | --- |
| `root` | | the directory, relative to the config file like certificates. It is made absolute when the config compiles, with symbolic links left as they are, so a deploy that switches `current -> releases/42` takes effect at once. It must be a directory then; if it goes away later, its files are `404`. |
| `index` | `['index.html']` | a name or up to 8: the file to serve for a directory, the first that exists; `[]` for none |
| `listing` | `false` | a directory without an index file gets a page listing it |
| `precompressed` | `false` | serve `name.br` or `name.gz` beside `name` when the client takes `br` or `gzip` (see below) |
| `followSymlinks` | `false` | follow symbolic links below the root, wherever they lead |
| `contentTypes` | | up to 64 extensions and their content types, over the built-in ones: `{ md: 'text/markdown' }` |
| `headers` | none | as on other routes; `content-encoding` and `vary` are the server's to set here |

A file route takes the prefix form, answers `GET` and `HEAD` (any other method is `405`), and has no `status`, `contentType`, `body`, `proxy` or `rewrite`.

**Paths.** Percent-escapes are decoded, then a path is refused with `404` if it could leave the root or reach what is hidden: a `.` or `..` segment, an empty one (`//`), an escaped `/`, a NUL, a control character or a backslash, or a segment that starts with `.` (dotfiles, `.git/`), except a first `.well-known`. A malformed escape is `400`. A directory asked for without its `/` is `301` to it, the query kept; with it, the first index file there, then the listing if there is one, else `404`. A file asked for with a `/` after it is `404`. The query string has no part in finding the file.

**What is opened.** Below the root, no symbolic link is followed, in the middle of a path or at its end, unless the route says `followSymlinks` (the root itself may be one). Only regular files and directories are opened at all: a FIFO, a socket or a device is `404`, without being touched. A file the server may not read is `403`; running out of descriptors or memory, `503`.

**Content types** come from the extension, case aside: `html`, `css`, `js`, `mjs`, `json`, `svg`, `png`, `jpg`, `webp`, `avif`, `ico`, `woff2`, `wasm`, `pdf`, `mp4`, `txt`, `xml`, `csv` and some forty more, text ones with `; charset=utf-8`. Anything else is `application/octet-stream`.

**Precompressed files.** With `precompressed: true`, a request for `app.js` from a client whose `Accept-Encoding` takes `br` gets `app.js.br` if it is there, with `content-encoding: br`; one that takes `gzip`, `app.js.gz`; `br` first if both. The content type is still that of `app.js`. Every file such a route serves carries `vary: accept-encoding`. A precompressed copy wins over [compressing on the fly](#compression), and it is the only way to serve `br`: make the files when you build the site (`brotli -k`, `gzip -k`), at levels too slow to spend on each request.

**Listings** show a directory's files and directories, with their sizes, directories first and then by name. Hidden files are left out, and so are symbolic links unless the route follows them. Names are escaped; at most 10,000 entries are shown.

**Validators and conditional requests.** Every file goes out with an `etag`, a `last-modified` and `accept-ranges: bytes`. The tag is the file's size, its modification time in nanoseconds and its inode, `"5e-17979cfe3d85cd15-3c9a1"`: strong, since any change to the file changes it. The inode tells apart files of one size and time, which deploys that pin times (`tar`, `rsync -t`, Nix) put in place; so servers behind one name give their copies of a file different tags, which costs a `200` where a `304` could have been, never the reverse. A precompressed copy has its own, as another representation of the file must, and a [gzipped](#compression) file a weak one; listings have none. `last-modified` is never later than `date`. Conditions are evaluated in the order RFC 9110 gives:

| Field | When it holds | Otherwise |
| --- | --- | --- |
| `If-Match` | the file's tag is listed, by strong comparison, or `*` | `412` |
| `If-Unmodified-Since` (without `If-Match`) | the file was not modified after the date | `412` |
| `If-None-Match` | the tag is listed, by weak comparison, or `*` | `304` for `GET` and `HEAD`, `412` otherwise |
| `If-Modified-Since` (without `If-None-Match`) | the file was modified after the date; a date in the future is ignored | `304` |

A `304` has no body: it carries the validators, `vary` and the route's headers, such as `cache-control`, so a cache can freshen what it holds. Dates may come in any of the three forms HTTP allows; a malformed one is ignored.

**Ranges.** A `GET` with `Range: bytes=…` naming one range gets `206 Partial Content` with `content-range`, read from the disk from where the range starts: `bytes=500-999`, `bytes=500-` and `bytes=-500` (the last 500 bytes); an end past the file is cut to it. A range that starts past the end is `416`, with `content-range: bytes */size`. A request for several ranges, a malformed `Range`, or another unit gets the whole file, as HTTP allows: multipart responses are rare, and an easy way to make a server send a file many times over. `If-Range` keeps the range only for the same file: its tag (strong comparison), or a date equal to `last-modified` when that is at least a second old, so a file changed within its second cannot pass for the old one. `HEAD` ignores `Range`. A range of a precompressed copy is a range of that copy; a request with `Range` is never gzipped on the fly.

**How it is served.** The disk is touched only on libuv's threadpool, never on a worker's event loop, so a slow disk stalls no connection that is not waiting for it. A file goes out in 64 KiB reads as fast as the client takes it, through the same backpressure as proxied bodies, and the write timeout covers a disk that does not answer too. The pool has two threads per worker, at least 4 and at most 64, unless `UV_THREADPOOL_SIZE` says otherwise. A file that gets shorter while it is being sent resets the connection, since its `content-length` promised more; one that grows is sent as long as it was. A download in progress copies what it needs from the route, so a reload goes on around it. A connection may hold a file besides its socket, so with file routes `maxConnections` is capped to half the descriptors, as with proxying.

**Small files are kept for a second.** Each worker keeps the files of up to 64 KiB it served last, whole and as they went, gzipped or not (16 MiB in all), and answers the same request within a second from memory, conditions and ranges included: so a changed or removed file can be served for up to a second after. Listings and errors are not kept.

The access log writes a line per file response as for any other, with `aborted` when the connection went before the file did; the `files` counter counts the files, ranges and listings served (`200` and `206`).

Not yet: several ranges in one response, and routes for single files.

## Compression

Xenia gzips responses as it serves them, with [zlib](index.md#third-party-code), for clients whose `Accept-Encoding` takes `gzip` (`*` takes it; `gzip;q=0` refuses it, whatever else is there). Static and file routes do it unless they say `compress: false`; proxy routes only with `compress: true`, since an upstream often compresses itself, and xenia compresses a proxied body on the worker's event loop.

```js
{ prefix: '/', root: 'public' },                              // compressed
{ prefix: '/raw/', root: 'public', compress: false },          // as it is
{ prefix: '/api/', proxy: 'app', compress: true },             // the upstream's answers, compressed
```

**What is compressed.** A response whose content type is worth it: `text/*`, JSON, JavaScript, XML, SVG and any other `+json` or `+xml` type, `application/wasm`, TrueType and OpenType fonts, and icons; with a body of at least 1 KiB when its length is known. Never one that is encoded already (a route's or an upstream's own `content-encoding`, or a [precompressed](#files) file, which wins), one whose `cache-control` says `no-transform`, a `206`, `204` or `304`, or the answer to a request with `Range`: ranges are of the file as it is, so downloads resume and media seeks as before. An HTTP/1.0 client gets gzip only where its length is known, since a body of unknown length would cost it its connection.

Static bodies are compressed once, so they get zlib's best; everything else is compressed for each request, so it gets its fastest, which on text costs a tenth to a third more bytes than the default level, for five to seven times the speed.

| Where | How | Level |
| --- | --- | ---: |
| static route | once per snapshot, kept only if smaller; sent with its length | 9 |
| file up to 64 KiB, listing | whole on the threadpool, sent with its length, and kept in the [small-file cache](#files) as it went | 1 |
| larger file | 64 KiB at a time on the threadpool as the client takes it; chunked on HTTP/1.1, plain DATA on HTTP/2 | 1 |
| proxied | as it arrives, flushed after every read from the upstream, so server-sent events and long polls arrive as they are sent; chunked or plain DATA | 1 |
| [cached](#caching) proxied | once per entry on the threadpool, at its first gzip hit, kept beside it; sent with its length | 6 |

**What a compressed response says.** `content-encoding: gzip`; `content-length` only where it is known; no `accept-ranges`. Its `etag` is the identity tag made weak with a suffix, `"5e-17979cfe3d85cd15-3c9a1"` becoming `W/"5e-17979cfe3d85cd15-3c9a1-gzip"`: its bytes depend on zlib's version and level, so it is never a strong validator. `If-None-Match` with it gets a `304`; `If-Match` and `If-Range`, which compare strongly, never match it. On a proxy route, `If-None-Match` goes upstream with such tags as they were made from, `W/"abc"` for `W/"abc-gzip"`, so revalidation still gets the upstream's `304`.

**`vary: accept-encoding`** comes with every answer that could have been gzipped, the plain one and `304`s included, so caches keep the two apart; a route's or an upstream's own `Vary` stays, with it beside, unless it names `accept-encoding` already or is `*`.

Static routes' gzip variants share the 64 MiB a config's bodies may take: they get the room the bodies leave, and the rest go as they are. A live change keeps the variants it has instead of compressing again. `/snapshot` shows each route's `compress`, and a static route's `gzipBytes`; `xenia check` marks routes where it is not the default.

**Counters:** `compressed` (responses sent gzipped), `compress_in_bytes` and `compress_out_bytes` (their bodies before and after), in `/stats`, `SIGUSR1` and `/metrics`.

Not yet: `br` or `zstd` on the fly (precompressed `.br` files are served), and a choice of level or types.

## Upstreams and proxying

`upstreams` names groups of HTTP/1.1 servers, and a route with `proxy` forwards to one. Servers are addresses, or names the control plane resolves (see [Names](#names)); workers only ever see addresses.

```js
export default {
  upstreams: {
    api: {
      servers: ['127.0.0.1:9001', '[::1]:9002'],
      responseTimeout: 30,
    },
  },
  routes: [
    { prefix: '/api/', proxy: 'api' },              // any method
    { path: '/upload', method: 'POST', proxy: 'api' },
    { path: '/', body: 'hello\n' },
  ],
};
```

| Setting | Default | Meaning |
| --- | --- | --- |
| `servers` | required | 1–64 servers, each at most once: `IPv4:port`, `[IPv6]:port`, or `host:port` (see [Names](#names)) |
| `connectTimeout` | 2 s | to establish a connection |
| `responseTimeout` | 30 s | from the request being written to the response head; then also the longest gap while reading the body, and how long an upstream may refuse a request body |
| `keepaliveTimeout` | 60 s | how long an idle connection is kept for reuse |
| `keepalive` | 32 | idle connections kept per server, per worker; 0 keeps none |
| `downTime` | 10 s | how long a server sits out after a failed connect |
| `balance` | `'round-robin'` | or `'least-connections'` (see [Balancing and health checks](#balancing-and-health-checks)) |
| `healthCheck` | none | check each server actively (see [Balancing and health checks](#balancing-and-health-checks)) |
| `tunnelIdleTimeout` | 300 s | how long a WebSocket tunnel may go without a byte either way (see [WebSockets](#websockets)) |
| `tls` | none: plain HTTP | connect over TLS (see [TLS to upstreams](#tls-to-upstreams)) |
| `resolveInterval` | 30 s | how often names are looked up again, 1 s to a day |
| `family` | `'any'` | the addresses a name's servers may have: `'any'`, `'ipv4'` or `'ipv6'` |

A proxy route has no `status`, `contentType`, `headers` or `body`; without `method` it takes every method. With proxy routes, clients get half of the open-file budget toward `maxConnections`, since each one may hold a connection upstream too.

**Requests.** The target goes upstream as sent (absolute-form becomes origin-form), or rewritten (see below), with the client's header fields except hop-by-hop ones (`Connection` and what it names, except `Host`, `Keep-Alive`, `Proxy-Connection`, `Proxy-Authorization`, `TE`, `Trailer`, `Transfer-Encoding`, `Upgrade`, all but a [WebSocket](#websockets) upgrade), `Expect` and the framing. `X-Forwarded-For` is replaced by the client's address, never appended to (behind `server.trustedProxies`, the client they name, as [rate limits](#rate-limits) find it), and `X-Forwarded-Proto` (`http` or `https`) is added, with what xenia saw of the client's certificate (see [Client certificates](#client-certificates)). What only a proxy may say is never taken from the client: its own `X-Forwarded-*` fields (`-Host`, `-Port`, `-Prefix` and any other), `Forwarded`, `X-Real-IP`, `X-Client-Verify` and `X-Client-Cert-*` are removed, so an upstream that builds URLs or redirects from them, trusting its proxy, cannot be told another host; `Host` passes through, or is added for HTTP/1.0 clients that sent none. A path that an upstream could read as a different one is refused with `400`: `.` or `..` segments, empty segments, backslashes, and escaped `/`, `\`, NUL or unreserved characters. `Expect: 100-continue` is answered by xenia.

**Bodies** stream in both directions and are framed anew on each side: `Content-Length` passes through, chunked is decoded and encoded again without extensions or trailers, and a response body that ends when the upstream closes reaches HTTP/1.1 clients chunked. HTTP/1.0 clients get any unsized response body delimited by closing the connection. Neither side outruns the other: a slow client pauses reading from the upstream and a slow upstream pauses reading from the client. A request body must move at least 1 KiB per `headerTimeout` while it is uploading. If the response arrives before the request body is through, it is forwarded at once and the rest of the body is read and dropped; the connection then closes. A route with `compress: true` gzips the responses worth it on the way (see [Compression](#compression)).

**Failures.** Requests go to a group's servers as its `balance` says, skipping servers drained through the [admin interface](running.md#admin-interface) and those [health checks](#balancing-and-health-checks) found unhealthy. A failed or timed-out connect marks the server down on that worker for `downTime` and tries the next one; when every server is down, the one back soonest is tried. When only unhealthy servers are left, requests go to them anyway. A group whose servers are all drained answers `503`. Running out of file descriptors, memory or local ports never marks a server down. Before the response head is forwarded, a failure is answered with `502 Bad Gateway`, or `504 Gateway Timeout` after `responseTimeout`; the client connection stays open if the request was read whole. After it, the client connection is reset, so a cut-off body never looks complete. Interim responses (`100`, `103`) are dropped; a `101` that is not a [WebSocket](#websockets) one, and malformed or ambiguous framing, are `502`.

**Connection reuse.** An upstream connection is kept for the next request only after a complete exchange with a server that allows it: not HTTP/1.0, not `Connection: close`, nothing after the response. If a kept connection turns out to be closed before any response byte arrives, the request is sent once more on a new one, but only if the server never got it, or for `GET`, `HEAD` or `OPTIONS` without a body.

**Path rewriting.** A proxy route's `rewrite` replaces the part of the path the route matched, its prefix or its whole path; the rest of the path and the query go as they came.

```js
{ prefix: '/api/', proxy: 'api', rewrite: '/' },         // /api/users?id=7 goes as /users?id=7
{ prefix: '/v1/', proxy: 'api', rewrite: '/legacy/' },   // /v1/a goes as /legacy/a
{ path: '/status', proxy: 'api', rewrite: '/healthz' },  // /status goes as /healthz
```

A rewrite follows the rules for `path`, and on a prefix route it ends with `/` as well. The client's path is checked as above before it is rewritten, and the access log shows it as sent.

Not yet: HTTP/2 upstreams, upgrades other than WebSocket, rewriting by pattern, and balancing by client address.

## Names

A server may be a name, `host:port`, as containers and Kubernetes give them. The control plane resolves it and puts its addresses in the snapshots; workers never look a name up, and see only addresses.

```js
export default {
  upstreams: {
    app: {
      servers: ['app.default.svc.cluster.local:8080', '10.0.0.9:8080'],  // names and addresses mix
      resolveInterval: 10,
    },
  },
  routes: [{ prefix: '/', proxy: 'app' }],
};
```

- **Every address is a server.** A name stands for each address it resolves to (A and AAAA records, unless `family` says one), each with its own pooled connections, down marks and health checks, balanced like listed ones. A group has at most 64 servers: addresses past that are left out, and the log says how many. An address that two entries share is one server.
- **Lookups** go to the system resolver (`getaddrinfo`, on libuv's threadpool), so `/etc/hosts` and `nsswitch.conf` apply. Each name is looked up again every `resolveInterval` of the groups that give it (the shortest); `getaddrinfo` reports no TTLs, so the interval is the refresh. A change publishes a new generation, built on the active one, so live changes stay; with 8 earlier snapshots still held by slow clients, it waits for them.
- **Failures never fail the config.** A name that does not resolve has no servers yet (a group with none answers `503`) and is tried every 5 s. A name that stops resolving keeps the addresses it had. Each is logged once, and so are changes: `upstream name app:8080 resolves to 10.0.0.5, 10.0.0.6`.
- **Reloads** look up the names a config adds before it is published, waiting up to 5 s; names known already keep their addresses, so a reload never leaves a group without them.
- **`$XENIA_HOSTS`**, when set, names a file in hosts format (`10.0.0.5 app.internal`, `#` comments) read before the system resolver at every lookup: for tests, and for overriding names on one machine without touching `/etc/hosts`.
- **TLS** to a name sends the name as SNI and checks the certificate against it, unless the group gives a `serverName` (see [TLS to upstreams](#tls-to-upstreams)).
- **Drains** take a name, `xenia ctl drain app:8080`: every address it has, then and later, sits out. An address drains one instance.

`xenia ctl resolve` (`POST /resolve`) looks every name up now. `/snapshot` lists each group's servers as configured and each server's address and name, and each name with its addresses, when it last resolved, its failures and its last error. `xenia check` does not resolve: it shows names as given.

Not yet: DNS TTLs, SRV records, and names in a health check's `host` or in listeners.

## Balancing and health checks

```js
export default {
  upstreams: {
    api: {
      servers: ['10.0.0.5:8080', '10.0.0.6:8080'],
      balance: 'least-connections',
      healthCheck: { path: '/healthz', interval: 5, timeout: 2, healthy: 2, unhealthy: 3 },
    },
  },
  routes: [{ prefix: '/api/', proxy: 'api' }],
};
```

**Balancing.** `round-robin` takes the servers in turn. `least-connections` sends each request to the server with the fewest requests in flight, a WebSocket tunnel counting as one; ties go in turn. Each worker counts its own, so at light load the counts mostly tie and the servers take turns. `/upstreams` shows the counts summed over the workers (`active`).

**Health checks.** With `healthCheck`, every server of the group is checked every `interval`.

| Setting | Default | Meaning |
| --- | --- | --- |
| `path` | required | the path and query to request; the rules of `path` apply |
| `host` | the TLS server name, else the server as configured (its address, or its name and port) | the `Host` field: a DNS name or an address, with any port |
| `interval` | 5 s | how often each server is checked |
| `timeout` | 2 s | for the response head; less than `interval` |
| `healthy` | 2 | passes in a row that make an unhealthy server healthy again, 1–10 |
| `unhealthy` | 3 | failures in a row that make a server unhealthy, 1–10 |

- **A check** is a new connection, over TLS with the group's `tls` settings when it has them, sending `GET path` with `Connection: close` and `User-Agent: xenia/VERSION health`. A status of 200–399 passes. An error, a timeout or any other status fails. Errors of xenia's own, out of descriptors, memory or local ports, count neither way.
- **Servers start healthy**, and requests skip unhealthy ones. When every server a group has left is unhealthy, requests go to them anyway and are counted as `fail_open`: the checks may be what is wrong, and a struggling server's answer beats none. Drained servers stay out either way.
- **One thread** runs every check, at most 32 at once. Each server's checks fall at a point in the interval set by what is checked, so a reload does not send them all at once.
- **What the checks found outlasts reloads and live changes.** A server is checked once per address, path, `host` and TLS settings, however many groups list it, and those groups share the result; their timings must then agree, or the config is refused. Changing the timings keeps what was found. A new path or host starts over, healthy.
- **Logs.** Each change is logged with its reason: `upstream 10.0.0.5:8080 (/healthz) is unhealthy after 3 failed checks: status 503`, then `is healthy again after 2 passed checks`; and once when a group has no healthy server left: `upstream group api: every server is unhealthy, so requests go to them anyway`.
- **The admin interface.** `/upstreams` shows each server's `health`, `last_check` and `checked_at`, and `/stats` counts the checks (`health_checks`) and the failed ones (`health_checks_failed`).
- **Failed connects** still mark a server down for `downTime` on the worker that saw them, with or without checks.

## WebSockets

A WebSocket upgrade goes through any proxy route: an HTTP/1.1 `GET` without a body whose `Connection` names `upgrade` and whose one `Upgrade` field lists `websocket`. It goes upstream with `Upgrade: websocket` alone, so no other protocol, `h2c` among them, reaches an upstream that way; a request with more than one `Upgrade` field is `400`. If the upstream answers `101` with `Upgrade: websocket`, the `101` goes to the client and from then on bytes pass both ways as they are, in a tunnel xenia does not parse. Any other `101` is `502`, and any other status an ordinary response.

- **Ending.** When either side closes in order (a FIN, or over TLS a `close_notify`), what is queued goes out to each side and both connections close; tunnels are never left half-open. A reset, an error, or a FIN over TLS without `close_notify`, which could be an attacker's, resets both. `tunnelIdleTimeout` closes a tunnel nothing has moved through either way (WebSocket pings count), and `writeTimeout` a client that stops reading.
- **Backpressure** works as for bodies: a slow reader on either side pauses reading from the other.
- **Memory.** A tunnel holds about 25 KiB of buffers, plus about 50 KiB for each side over TLS.
- **The access log** writes one line per tunnel, when it ends: `status` 101, `bytes` sent to the client, `request_bytes` received from it, `duration` the tunnel's life, and `aborted` after a reset, an error or the idle timeout. The `tunnels` counter counts them.
- **Drains and stops.** A tunnel to a drained or unhealthy server stays open until one side ends it. On `SIGTERM`, open tunnels keep the server up until they end or `shutdownTimeout` passes.
- The upstream must not answer with HTTP on the connection after its `101`.

## Virtual hosts

A route with `host` serves only requests for that host. Routes without one form the default site, which serves every host no route names.

```js
export default {
  routes: [
    { host: 'example.com', path: '/', body: 'example.com\n' },
    { host: '*.example.com', path: '/', body: 'a subdomain\n' },
    { path: '/', body: 'everything else\n' },
  ],
};
```

- `host` is a DNS name, or `*.` and one, compared without case. A wildcard covers exactly one more label: `*.example.com` serves `www.example.com`, but neither `example.com` nor `a.b.example.com`.
- A request's host is the authority of an absolute-form target, else its `Host` field, without its port or a trailing dot. Its site is the route set for that exact host, else the wildcard for its parent, else the default site.
- Paths match within that site only. A path the site lacks is `404`, never another site's route. To share routes between sites, build them once and spread them into each: the config is JavaScript.
- A malformed `Host` is `400`. HTTP/1.0 requests without one get the default site.
- Duplicate routes are per site: `GET /` may appear once per host.
- On TLS connections the host must also be one the handshake's certificate names, else `421 Misdirected Request` (see below).

## Feature flags

`flags` declares feature flags and their values. A route with `flag` is served only while its flag is on, or, with `!` in front, only while it is off. Flags can be changed live through the [admin interface](running.md#admin-interface), so a route can be switched on or off, or a site moved to another upstream group, without editing the config.

```js
export default {
  flags: { green: false, exports: true },
  upstreams: {
    blue: { servers: ['10.0.0.5:8080'] },
    green: { servers: ['10.0.0.6:8080'] },
  },
  routes: [
    { prefix: '/', proxy: 'blue', flag: '!green' },   // while green is off
    { prefix: '/', proxy: 'green', flag: 'green' },   // ...and this one while it is on
    { path: '/export', proxy: 'blue', flag: 'exports' },
  ],
};
```

- A flag name is 1–32 lowercase letters, digits, `_` or `-`, and starts with a letter. A config has at most 64 flags.
- A route whose flag is against it behaves as if it were not in the config: its paths fall to a shorter prefix, a host whose routes are all off gets the default site, and `xenia check` marks it `(off)`.
- Two routes on one path whose methods overlap conflict unless they sit on opposite sides of one flag, as `blue` and `green` do above. So no setting of the flags can make both serve, and changing a flag can never cause a conflict.
- Each snapshot keeps the routes that are off, so a flag change needs no recompile. It publishes a new generation, as a reload does.

## Rate limits

`limits` names rate limits, and a route with `limit` counts each client's requests against one. A client over it gets `429 Too Many Requests` with a `retry-after`, before anything else happens for the request: nothing is proxied, no file is opened.

```js
export default {
  server: { trustedProxies: ['10.0.0.0/8'] },        // the load balancers in front
  limits: {
    api:   { rate: 10, burst: 20 },                  // 10 a second, 20 at once
    login: { rate: 1 / 60, burst: 5 },               // 5 at once, then one a minute
  },
  routes: [
    { prefix: '/api/', proxy: 'app', limit: 'api' },
    { path: '/login', method: 'POST', proxy: 'app', limit: 'login' },
  ],
};
```

| Field | Default | Meaning |
| --- | --- | --- |
| `rate` | | requests a second, fractions allowed: from one a day to a million a second |
| `burst` | the rate rounded up, at least 1 | how many may come at once, after a quiet spell; 1 to 1,000,000 |
| `clients` | 65,536 | how many clients the limit keeps track of at once, 1,024 to 4,194,304; about 24 bytes each |

- **The algorithm** is the generic cell rate algorithm, a token bucket kept as one timestamp per client: requests may come `burst` at once, then one every `1/rate` seconds. A refused request costs nothing, so a client that keeps trying waits no longer than one that waits. `retry-after` is the whole seconds until a request would pass.
- **Routes that name one limit share it:** a client's requests to `/api/` and to `/login` above count separately, but every method and path under `/api/` counts against the same budget. `HEAD` counts as `GET`. A limit no route names does nothing; up to 64 limits.
- **Exact across workers.** A client's connections land on any worker, so the limits are shared: each limit has one table for all of them, split into 256 stripes by a seeded hash of the client, each with a lock held for a lookup and its update. The tables live outside the snapshots, so a reload or a live change keeps what each limit knew; changing a limit's `clients` starts it afresh.
- **A full table** makes room by forgetting a client at full credit first, which changes nothing, else the one closest to it, which gets a full budget early; the `evicted` count says how often.

**The client** is the connection's address, without its port; IPv6 by its /64, which one host has to itself. When the connection comes from one of `server.trustedProxies`, the client is who they say: `X-Forwarded-For` (all of its fields, in order) is read from the right, skipping trusted addresses, and the first one that is not trusted is the client. A client cannot choose its address by sending its own `X-Forwarded-For`, since the proxy appends the address it saw. With every address trusted, the leftmost counts; a malformed entry ends the walk, and the address to its right counts.

**Over the limit:** `429` with `retry-after`, `content-type: text/plain` and `too many requests`, over HTTP/1.1 and HTTP/2 alike. On HTTP/1.1 the connection closes after it when the request had a body, which is not read.

`/snapshot` lists the limits with what they counted (`allowed`, `refused`, `evicted`), `trustedProxies`, and each route's `limit`; `xenia check` lists them too. The counter `status_429` and `xenia_ratelimit_requests_total{limit,result}` are in `/stats` and `/metrics`.

The same client is the `X-Forwarded-For` that goes upstream and the access log's `client`.

Not yet: limits on connections or on requests in flight, keys other than the client's address, and waiting instead of refusing.

## Caching

`caches` names response caches in memory, and a proxy route with `cache` keeps its upstream's responses in one, to answer later requests for them without going upstream. A cache is shared by every worker and outlives reloads.

```js
export default {
  caches: { pages: { size: 256 * 1024 * 1024 } },            // bytes
  upstreams: { app: { servers: ['10.0.0.5:8080'] } },
  routes: [
    { prefix: '/', proxy: 'app', cache: 'pages', ttl: 60 },   // ttl: when the upstream says nothing
    { prefix: '/api/', proxy: 'app' },                         // not cached
  ],
};
```

| Field | Default | Meaning |
| --- | --- | --- |
| `size` | | the cache's budget in bytes, 16 MiB to 64 GiB: heads, bodies and gzip copies |
| `maxEntry` | 1 MiB, or `size`/128 if less | the largest body kept, at most `size`/128 |
| route `cache` | | the cache a proxy route keeps its responses in |
| route `ttl` | | seconds a response stays fresh when the upstream gives it no lifetime, 1 s to a year; only with `cache` |
| route `staleWhileRevalidate` | | seconds past its lifetime a response may be answered with while it is refreshed, when the upstream does not say; 1 s to a year, only with `cache` |
| route `staleIfError` | | ...and while the upstream fails |

Xenia caches as a shared cache must ([RFC 9111](https://www.rfc-editor.org/rfc/rfc9111)): **the upstream decides**, and `ttl` only fills in where it says nothing.

- **What is looked up:** `GET` and `HEAD` (a `HEAD` reads what a `GET` kept), by scheme (plain HTTP and HTTPS never share an entry, since what goes upstream differs), host (lowercase, without the port) and target (path and query, as sent). Requests with a body, `Authorization`, a verified client certificate, `Range`, `If-Match`, `If-Unmodified-Since` or `If-Range`, WebSocket upgrades and `Cache-Control: no-store` go round the cache. `Cache-Control: no-cache` or `Pragma: no-cache` revalidates.
- **What is kept:** a whole response without `no-store`, `private`, `Set-Cookie` or `Vary: *`, and not over `maxEntry`; one with a `content-encoding` only if it varies on `Accept-Encoding`. `200`, `203`, `204`, `300`, `301`, `308`, `404`, `405`, `410`, `414` and `501` with an explicit lifetime; `ttl` covers `200`, `301`, `308`, `404` and `410`. A response that cannot be kept sets a pass marker: for 10 seconds requests for it go straight upstream.
- **Freshness,** the first that applies: `s-maxage`, `max-age`, `Expires` less `Date`, the route's `ttl`. The upstream's `Age` counts. `no-cache` or a lifetime of 0 keeps it only to be revalidated each time, and only with a validator (`ETag`, `Last-Modified`).
- **Variants:** a response's `Vary` fields are matched on, up to 8 variants a key, as the request goes upstream: a field it does not take there (one its `Connection` names, say) names no variant.

**A hit** goes straight back: the upstream's head as it was kept (its hop-by-hop fields, `set-cookie`, `xenia-snapshot` and `x-request-id` gone), with `age`, `x-cache: hit` and our own fields; or a `304` when the client's `If-None-Match` or `If-Modified-Since` holds for it. Nothing upstream is touched or counted.

**A miss** goes upstream without the client's `If-None-Match` and `If-Modified-Since`, so the whole response comes back to be kept; the client's `304` is decided here. The body is kept as it streams to the client, and stored once it is whole: a response cut off, or grown past `maxEntry`, is not.

**Revalidation.** A stale entry with a validator is fetched with `If-None-Match` and `If-Modified-Since` built from it: a `304` renews it (the fields it gives replace those kept, the body stays, and freshness and stale windows come from the fields that make; its age starts over, from the `304`'s `Date` and `Age`) and the client is answered from it (`x-cache: revalidated`); a `200` replaces it. Without a validator a stale entry is a plain miss.

**Serving stale** ([RFC 5861](https://www.rfc-editor.org/rfc/rfc5861)). A response's `stale-while-revalidate=N` and `stale-if-error=N`, or where it has neither the route's `staleWhileRevalidate` and `staleIfError`, let it be used for N seconds after it goes stale. `no-cache`, `must-revalidate`, `proxy-revalidate` and `s-maxage` forbid both, whatever the route says.

```js
{ prefix: '/', proxy: 'app', cache: 'pages', staleWhileRevalidate: 30, staleIfError: 86400 },
```

- **While revalidating:** every request that finds it within the window is answered from it at once, with `x-cache: stale`, `HEAD` included; none waits. The first one hands the revalidation to a **refresh**: the same request, with the entry's validators, forwarded by the worker with no client behind it, whose `304` renews the entry and whose keepable `200` replaces it. A refresh that gets nothing it can keep leaves the entry as it was, for the next request to refresh. A client asking for `no-cache` revalidates as before. A worker runs at most 64 refreshes at once (past that, a request revalidates itself), refreshes are not in the access log, and a shutdown cancels those in flight rather than wait.
- **When the upstream fails:** a request that had an entry within its window to go by, and gets `502`, `503` or `504` from xenia (no connection, a timeout, every server drained) or `500`, `502`, `503` or `504` from the upstream, is answered from it instead, with `x-cache: stale-error`. The upstream's body is not read. Other statuses go through, and so does everything once the window has passed.

**Coalescing.** The first miss for a key fetches it; requests for the same key meanwhile, on any worker, wait for that (up to the upstream group's `responseTimeout`) and are then answered from what was kept, as hits. If it could not be kept, or failed, each goes upstream itself. A waiting HTTP/1.1 connection reads no further requests; on HTTP/2 only that stream waits.

**gzip.** On a route with `compress`, what is kept is the upstream's body as it came. The first hit from a client that takes gzip makes a gzip copy on the threadpool, at level 6, and keeps it beside the entry; later ones get it with its length, as static bodies are.

**Memory.** Each cache is split into 32 stripes by a seeded hash of the key, each with a lock, a table, a least-recently-used list and a share of the budget: over it, the least recently used go. An entry is one allocation that never changes once stored; a hit copies what it sends, so an entry evicted or purged meanwhile is no matter. A reload keeps a cache's entries while its `size` and `maxEntry` stay; changing either starts it afresh.

**`x-cache`** on every response of a route with a cache says what happened: `hit`, `miss`, `revalidated`, `stale`, `stale-error`, `pass` (a pass marker) or `bypass`; so does the access log's `"cache"`.

**Purging.** `xenia ctl cache purge pages prefix=/blog/` (`POST /cache/purge?cache=pages&prefix=/blog/`) drops entries under a prefix; `path=/x` those for a path, with any query; neither, all of them; `host=example.com` limits it to one host. It answers with how many went.

`/snapshot` lists the caches with their entries, bytes and counts (`hits`, `misses`, `revalidated`, `bypassed`, `stale`, `staleErrors`, `refreshes`, `refreshFailures`, `stored`, `evicted`, `purged`), and each route's `cache`, `ttl`, `staleWhileRevalidate` and `staleIfError`; `xenia check` lists them. The counters `cache_hits` (stale answers among them) and `cache_misses`, and `xenia_cache_requests_total{cache,result}`, `xenia_cache_entries`, `xenia_cache_bytes`, `xenia_cache_stored_total`, `xenia_cache_evicted_total`, `xenia_cache_refreshes_total` and `xenia_cache_refresh_failures_total`, are in `/stats` and `/metrics`. A `stale-error` answer counts as the `miss` it began as, as `revalidated` does.

Not yet: keeping caches on disk, ranges from kept bodies, caching static and file routes (they have their own), client `Cache-Control` beyond `no-store` and `no-cache` (`max-stale`, `min-fresh`, `stale-if-error`), and purging by tag.

## TLS

`--listen-tls HOST:PORT` serves HTTPS with the certificates in the config's `tls` section; it can be given several times, alongside `--listen`.

```sh
xenia --listen 0.0.0.0:80 --listen-tls 0.0.0.0:443     # ./xenia.conf.js
```

```js
// ACME HTTP-01 challenges, answered over plain HTTP.
const acme = { prefix: '/.well-known/acme-challenge/', proxy: 'acme', allowHttp: true };

export default {
  tls: {
    certificates: [
      { cert: 'certs/example.com.pem', key: 'certs/example.com.key' },   // the default
      { cert: 'certs/example.org.pem', key: 'certs/example.org.key' },
    ],
    redirect: true,
    hsts: { maxAge: 31536000 },
  },
  upstreams: { acme: { servers: ['127.0.0.1:8402'] } },
  routes: [
    { ...acme, host: 'example.org' },               // each site needs its own
    { host: 'example.org', path: '/', body: 'example.org\n' },
    acme,
    { path: '/', body: 'example.com\n' },
  ],
};
```

| Setting | Default | Meaning |
| --- | --- | --- |
| `certificates` | required | 1–64 `{ cert, key, clients }` entries: PEM files, relative to the config file, of a certificate chain, leaf first (at most 64 KiB), and its private key (16 KiB), and optionally a client certificate policy (see [Client certificates](#client-certificates)). The first is the default. |
| `minVersion` | `'1.2'` | `'1.2'` or `'1.3'` |
| `sessionTickets` | `true` | session resumption by ticket |
| `redirect` | `false` | plain HTTP requests get `308` to the same host and target over HTTPS |
| `hsts` | `false` | `true` (a year), or `{ maxAge, includeSubDomains, preload }` with `maxAge` in seconds: the `Strict-Transport-Security` header on every TLS response |

**Certificates** are read whenever the config is compiled, at startup and on every `SIGHUP`, and travel with the compiled config: the snapshot is the file plus the files it names, with live changes on top. A key that does not match its certificate, a certificate that has expired or is not yet valid, and two certificates naming the same host are errors, which keep the current snapshot serving; one that expires within 14 days is logged. Each serves the DNS names in its subject alternative names, or its common name if it has none. Private keys are wiped from memory once parsed. A `--listen-tls` without `tls.certificates` stops startup, and a reload that removes them is refused.

**Renewal.** Replace the files, then `kill -HUP`. New handshakes get the new certificates; open connections keep the ones they started with, which are freed when the last of those connections closes.

**Choosing a certificate.** The client's SNI name picks the certificate that names it, else one whose wildcard covers it, else the default; clients without SNI get the default. ALPN offers `h2`, then `http/1.1` (see [HTTP/2](#http2)).

**Host and certificate.** A request's host must be a name the chosen certificate covers, or the request gets `421 Misdirected Request` and the connection stays open. So one domain's certificate never reaches another domain's routes. Certificates name DNS names, so a `Host` that is an IP address gets `421` too. A request without a host (HTTP/1.0 without `Host`, or an empty one) goes to the default site, so it is served only on a connection that chose the default certificate, and gets `421` on any other: a certificate that asks for no client certificate cannot reach the default site of one that requires them.

**Resumption.** Session tickets are sealed with one AES-256-GCM key per process, which Mbed TLS rotates daily. A client resumes on whichever worker it lands, and across reloads, but not across restarts. TLS 1.3 resumption still runs an (EC)DHE exchange.

**Redirect and HSTS.** With `redirect`, a request on a plain listener gets `308 Permanent Redirect` to `https://` its host and target. The port is the first `--listen-tls` listener's, left out when it is 443. Routes with `allowHttp: true` are served over plain HTTP all the same, for ACME HTTP-01 challenges and health checks. Like any route, one belongs to its site: a host with a site of its own needs its own copy, as in the example above. A request without a `Host` has nowhere to go and gets `400`. With `hsts`, every response on a TLS connection carries `Strict-Transport-Security`: static, error and proxied ones, unless the upstream sent its own. Plain HTTP responses never do.

**Proxying.** For TLS clients the request carries `X-Forwarded-Proto: https`. Upstreams speak plain HTTP unless their group has `tls` (see [TLS to upstreams](#tls-to-upstreams)).

**Memory.** A TLS connection holds up to about 50 KiB more than a plain one: Mbed TLS keeps a 16 KiB record buffer each way, and xenia 16 KiB for ciphertext not yet decrypted. The same goes for each TLS connection to an upstream, pooled ones included.

Not yet: OCSP stapling, certificates other than files, and listeners defined in the config.

## Client certificates

A certificate entry can ask clients for certificates, for the names it serves; a route can require a verified one.

```js
export default {
  tls: {
    certificates: [
      { cert: 'certs/example.com.pem', key: 'certs/example.com.key' },
      { cert: 'certs/admin.pem', key: 'certs/admin.key',
        clients: { ca: 'certs/staff-ca.pem', crl: 'certs/staff.crl' } },   // required
      { cert: 'certs/api.pem', key: 'certs/api.key',
        clients: { ca: 'certs/partners-ca.pem', required: false } },
    ],
  },
  upstreams: { api: { servers: ['127.0.0.1:9001'] } },
  routes: [
    { host: 'admin.example.com', prefix: '/', proxy: 'api' },
    { host: 'api.example.com', prefix: '/partners/', proxy: 'api', clientCert: true },
    { host: 'api.example.com', prefix: '/', proxy: 'api' },
    { path: '/', body: 'example.com\n' },
  ],
};
```

| Setting | Default | Meaning |
| --- | --- | --- |
| `clients.ca` | required | the CAs whose certificates count: a file or an array of up to 16, PEM or DER |
| `clients.crl` | none | revocation lists from those CAs, the same way (see [Revocation lists](#revocation-lists)) |
| `clients.required` | `true` | `true`: no handshake without a verified certificate. `false`: anyone gets in, and the certificate is recorded if it verifies |

**Which policy.** The certificate the handshake chooses decides, by SNI as for any certificate; clients without SNI get the default certificate's. With the Host and certificate rule above, a request reaches a domain's routes only through that domain's handshake, so its policy cannot be stepped around. A route with `clientCert: true` answers `403` without a verified certificate. Over plain HTTP, which would bypass the policy, a host whose certificate requires client certificates gets `403`, except on `allowHttp` routes; with `tls.redirect` such requests are redirected anyway. `clientCert` needs some certificate with `clients`, and does not go with `allowHttp`.

**Identity to upstreams.** Proxied requests carry what xenia saw:

```
x-client-verify: SUCCESS
x-client-cert-subject: CN=alice
x-client-cert-issuer: CN=staff CA
x-client-cert-serial: 14
x-client-cert-fingerprint: 3f5c...   (SHA-256 of the certificate's DER, hex)
```

`x-client-verify` is `SUCCESS`, `FAILED` (optional, and the certificate does not verify) or `NONE` (no certificate asked for or given, or plain HTTP); the `x-client-cert-*` lines come only with `SUCCESS`. Whatever `x-client-verify` and `x-client-cert-*` fields a client sends are removed, so an upstream can trust these.

**No resumption.** Where client certificates count, sessions are not resumed: every connection makes a full handshake and checks the certificate against the current CAs and CRLs. Mbed TLS keeps no client certificate in a resumed TLS 1.3 session, and a session ticket is not bound to the certificate entry that checked it. Other domains resume as usual.

**Limits.** TLS 1.3 sends clients no list of acceptable CAs (TLS 1.2 does), so browsers offer every certificate they have; post-handshake authentication is not supported, so a certificate is asked for at the handshake or not at all. A client turned away over TLS 1.3 sees a decryption error rather than the alert that says why: Mbed TLS sends that alert under its handshake keys, which the client has already left behind.

## TLS to upstreams

An upstream group with `tls` connects to its servers over TLS. A server given as a [name](#names) is checked against that name; addresses need the group to name the server it expects.

```js
export default {
  upstreams: {
    api: {
      servers: ['10.0.0.5:443', '10.0.0.6:443'],
      tls: {
        serverName: 'api.internal',
        ca: 'certs/internal-ca.pem',
        cert: 'certs/xenia.pem', key: 'certs/xenia.key',   // for upstreams that ask
      },
    },
  },
  routes: [{ prefix: '/api/', proxy: 'api' }],
};
```

| Setting | Default | Meaning |
| --- | --- | --- |
| `serverName` | each name; required with addresses | sent as SNI, and what the server's certificate must name; with names, it overrides theirs for the whole group |
| `ca` | required | the CAs to trust: a file or an array of up to 16, PEM or DER |
| `crl` | none | revocation lists from those CAs (see [Revocation lists](#revocation-lists)) |
| `cert`, `key` | none | xenia's own certificate chain and key, for servers that ask for one |
| `minVersion` | `'1.2'` | `'1.2'` or `'1.3'` |

There is no way to skip verification: give the CA that signed the servers' certificates. Files are read when the config compiles, as for listener certificates; groups with the same settings share them.

**Failures.** The handshake runs within `connectTimeout`. A handshake that fails, whether a certificate does not verify, the server sends an alert, or the connection drops or times out, counts as a failed connect: the server sits out `downTime`, logged once with the reason (`upstream 10.0.0.5:443 is down for 10 s: TLS handshake: The certificate is not correctly signed by the trusted CA`), and the next server is tried. A server counts as up again only after a handshake succeeds.

**Connections.** Pooled connections are reused only for the settings they were made with, so a reload that changes nothing keeps them, and two groups that share servers with different certificates never mix. Each worker keeps the last session per server to resume, so new connections usually skip the full handshake. Connections close with `close_notify`; a response body that ends with the connection is whole only if `close_notify` ended it, else the client's connection is reset, as for any cut-off body.

## Revocation lists

`crl` names certificate revocation lists, PEM or DER, for client certificates and upstream servers alike. They are read when the config compiles: renew a CRL by replacing the file, then `kill -HUP`. Revocation is checked at the handshake, so connections already open keep going.

A CRL is refused, keeping the current snapshot, when it is past its `nextUpdate` or before its `thisUpdate`, when none of the `ca` certificates issued it (Mbed TLS would ignore it), or when its CA's key usage does not allow signing CRLs (Mbed TLS would then refuse every certificate that CA issued). One due within two days is logged. If a CRL's `nextUpdate` passes while it is in use, every certificate its CA issued is refused until a newer one is loaded.

## HTTP/2

Clients can speak HTTP/2 (RFC 9113) on every route. It is on by default:

- **On `--listen-tls`,** ALPN offers `h2` before `http/1.1`, so browsers and other clients that support it take it.
- **On plain `--listen`,** a connection that starts with the HTTP/2 preface speaks it ("prior knowledge"). This is the usual way behind a load balancer, or with `curl --http2-prior-knowledge`. The deprecated `Upgrade: h2c` is ignored.

`server: { http2: false }` turns both off: ALPN offers `http/1.1` alone, and the preface gets `505`. A reload changes it for new connections.

```sh
curl --http2 https://example.com/             # ALPN
curl --http2-prior-knowledge http://10.0.0.2/ # behind a load balancer
```

**What changes for a request.** The answer itself does not change: routes, files, proxying, request IDs, the access log (`"proto":"HTTP/2.0"`) and metrics work as they do for HTTP/1.1.

- **Concurrency:** a connection carries up to 128 requests at once, each on its own stream, and a slow one holds back no other.
- **Responses:** they go out as the client's flow-control windows allow. The streams take turns, a frame at a time.
- **Upstreams:** proxied requests still go to upstreams over HTTP/1.1. A request body without a `content-length` goes chunked.
- **Held requests:** a request whose body is still coming waits until the client has sent all of it, unless it is for a proxy route, whose bodies stream. Its length and its frames are checked first, and a body on a route that takes none is `413` then.

**Limits.** Each connection sets these:

| | |
| --- | --- |
| concurrent streams | 128; one more is refused with `REFUSED_STREAM`, which clients retry |
| header list | 16 KiB as RFC 9113 counts it, or 64 fields; more is `431` |
| header block | 32 KiB, in at most 64 frames; more ends the connection |
| frames | 16 KiB |
| windows | 256 KiB per stream, 1 MiB per connection, given back as the request body is forwarded |

**Abuse guards.** A connection is ended with `GOAWAY ENHANCE_YOUR_CALM` when the client:

- resets streams, or has xenia reset them, faster than 200 per 10 seconds together (rapid reset, and its kind where the client sends what xenia must answer with `RST_STREAM`: a `WINDOW_UPDATE` of 0, a malformed or refused request, `DATA` past its `content-length`);
- sends frames that cost xenia more than the client faster than 1000 per 10 seconds: PING, SETTINGS, PRIORITY, empty DATA, frames for closed streams.

A malformed request ends its stream with `PROTOCOL_ERROR`; the connection goes on. Examples are an uppercase field name, a field value with a control character (as HTTP/1.1 refuses them), a connection-specific field such as `connection` or `transfer-encoding`, missing or repeated pseudo-headers, or a `content-length` the DATA does not match. Frames the client sent before a reset of xenia's reached it, trailers after an early answer say, are ignored.

**Backpressure.** xenia stops reading a connection while more than 64 KiB is queued for the client, its answers to the client's own frames included, and reads on below 16 KiB, as with HTTP/1.1: a client that sends faster than it reads waits. Data the client's windows hold back does not count, so its `WINDOW_UPDATE` always gets in.

**Descriptors.** A stream that would take a descriptor (a proxied request or a file) when its worker's share is spoken for is refused with `REFUSED_STREAM`. That share is half the open-file budget, split between the workers.

**Timeouts.** They apply as for HTTP/1.1:

- `headerTimeout` until the first request;
- `idleTimeout` with no stream open;
- `writeTimeout` while response bytes wait, whether in the socket or for the client to open its window; a stream whose window stays shut twice that long, once xenia has sent all it could, is reset with `CANCEL` (and its upstream let go) even while other bytes keep the connection moving, counted as a write timeout;
- the body timeout for uploads that stall.

A drain sends `GOAWAY`; streams already open finish, and the connection closes after them.

**Over TLS,** HTTP/2 needs TLS 1.3, or TLS 1.2 with an ECDHE suite and an AEAD cipher. Any other TLS 1.2 suite gets `GOAWAY INADEQUATE_SECURITY`.

**Counters:** `h2_connections`, `h2_streams`, `h2_refused`, `h2_resets` (by the client) and `h2_goaways` (with an error), in `/stats`, `SIGUSR1` and `/metrics`.

Not supported:

- **WebSockets over HTTP/2** (RFC 8441): xenia does not advertise them, so browsers open an HTTP/1.1 connection for them.
- **Server push.**
- **Priorities,** which RFC 9113 deprecates: streams take turns.
- **HTTP/2 to upstreams.**
- **`CONNECT`,** which gets `405`.
