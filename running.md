# Running

`--listen HOST:PORT` (default `127.0.0.1:8080`; IPv6 as `[::1]:8080`) can be given several times to serve on several addresses at once, and `--listen-tls HOST:PORT` likewise serves HTTPS (see [TLS](configuration.md#tls)); up to 16 listeners in all. `--workers N` (default: one per CPU, at most 64) sets the number of data-plane threads. The control thread accepts every connection and hands it to the worker with the fewest open connections; requests never cross threads. `--admin PATH` serves the [admin interface](#admin-interface) (and `--admin-listen HOST:PORT` over TCP), `--metrics HOST:PORT` serves [metrics](#metrics) for Prometheus, and `--access-log FILE` writes [access logs](#access-logs). `--version` prints xenia's version with those of libuv, QuickJS-ng and Mbed TLS, and says whether the binary is static.

**Running as a user.** `--user USER[:GROUP]` (names or numbers; without `GROUP`, the user's own) binds every `--listen`, `--listen-tls`, `--admin-listen` and `--metrics` address first, which ports below 1024 need root for, then becomes that user for good: its groups, its group, then the user, and on Linux `no_new_privs`, before any thread starts. Everything after is done as that user: reading the admin token file (which it must own), creating the admin socket, opening the access log, and compiling every config, the first and each reload's. So the config, the modules it imports, TLS keys and served files must be readable by the user, and the directories of the admin socket and the access log writable by it. Naming the user xenia already runs as changes nothing, and only root can name another; a static Linux binary finds names in `/etc/passwd` only. Started as root without `--user`, xenia warns. `GET /status` reports the `uid` it serves as. [Deployment](deployment.md) puts it together on a host.

**Under systemd.** With `NOTIFY_SOCKET` set, as systemd sets it for `Type=notify` and `Type=notify-reload` services, xenia reports its state: `READY=1` once it accepts connections, `RELOADING=1` (with `MONOTONIC_USEC`) when a reload starts and `READY=1` when it ends, applied or not (`STATUS` says which generation serves, and why a reload failed), and `STOPPING=1` when a drain starts. Reloads asked for while one runs report as one. So `systemctl reload` waits for the outcome. [`contrib/systemd/xenia.service`](https://github.com/xenia-platform/releases/blob/main/contrib/systemd/xenia.service) is a unit to start from: it runs xenia as its own user, allowed to bind ports below 1024 and little else. `cmake --install` installs it under `share/xenia`, and release archives carry it.

**Graceful shutdown.** On `SIGTERM` or `SIGINT` the server stops accepting, closes idle keep-alive connections, and lets responses already in flight finish, proxied ones included; each connection then closes cleanly. Idle upstream connections close at once, and health checks stop. [WebSocket](configuration.md#websockets) tunnels go on until one side ends them. It exits once no connections remain, or after `shutdownTimeout`, closing whatever is left. A second signal closes them at once.

## Signals

The running server answers these signals:

| Signal | Effect |
| --- | --- |
| `SIGHUP` | recompile the config and publish it as a new snapshot generation |
| `SIGTERM`, `SIGINT` | drain gracefully (see [Running](#running)); a second one stops at once |
| `SIGUSR1` | each worker logs its connection, request and timeout counters, what it proxied, its TLS handshakes, resumptions, failures, `421`s, redirects and `403`s, its TLS handshakes with upstreams, the files it served, what it compressed, the requests its rate limits refused, and the access log lines it wrote and dropped |
| `SIGUSR2` | reopen the access log (see [Access logs](#access-logs)) |

## Admin interface

`--admin PATH` serves an admin interface: HTTP/1.1 with JSON replies, on a Unix socket at `PATH` that only the user xenia runs as can use (mode 0600). `xenia ctl` is its client. `curl --unix-socket PATH http://x/status` works too (use `-g` for IPv6 brackets).

```sh
xenia --listen 0.0.0.0:80 --admin /run/xenia/admin.sock
export XENIA_ADMIN=/run/xenia/admin.sock
xenia ctl status
xenia ctl reload                  # waits for the outcome
xenia ctl flags green=on          # several at once go into one generation
xenia ctl drain 10.0.0.5:8080     # out of rotation, in every group
xenia ctl undrain 10.0.0.5:8080
xenia ctl metrics                 # what Prometheus scrapes
xenia ctl cache purge pages prefix=/blog/
```

| Request | `xenia ctl` | |
| --- | --- | --- |
| `GET /status` | `status` | version, uptime, the generation and when it was published, the last reload's outcome, listeners, open connections, the access log, live changes |
| `GET /snapshot` | `snapshot` | the active snapshot: server settings, flags, routes (those a flag holds back included), upstream groups, TLS |
| `GET /stats` | `stats` | each worker's counters, their totals, and the health checks run and failed |
| `GET /metrics` | `metrics` | the [metrics](#metrics), in Prometheus' text format |
| `GET /upstreams` | `upstreams` | each upstream server: drained or not, on how many workers it is marked down and for how much longer, idle pooled connections, requests in flight, and what [health checks](configuration.md#balancing-and-health-checks) found |
| `POST /reload` | `reload` | compiles and publishes the config, as `SIGHUP` does, then answers with the new generation, or `422` and the error |
| `POST /flags?NAME=on&...` | `flags NAME=on ...` | sets flags `on`, `off`, or back to the config's value with `reset` |
| `POST /drain?server=ADDR&...` | `drain ADDR ...` | takes upstream servers out of rotation: addresses, or names for every address they have |
| `POST /undrain?server=ADDR&...` | `undrain ADDR ...` | puts them back |
| `POST /logs/reopen` | `reopen` | reopens the access log, as `SIGUSR2` does |
| `POST /cache/purge?cache=NAME&...` | `cache purge NAME ...` | drops a [cache](configuration.md#caching)'s entries: `prefix=`, `path=` or all, and `host=` |
| `POST /resolve` | `resolve` | looks every upstream [name](configuration.md#names) up now, then answers with what each has |

**Over TCP.** `--admin-listen HOST:PORT --admin-token-file FILE` serves the same interface over TCP, for reaching it from another host. It is plain HTTP, so keep it on a private network or behind something that adds TLS.

```sh
openssl rand -hex 32 > /etc/xenia/admin.token && chmod 600 /etc/xenia/admin.token
xenia --admin-listen 10.0.0.2:9000 --admin-token-file /etc/xenia/admin.token
# elsewhere, with a copy of the token:
xenia ctl --admin-listen 10.0.0.2:9000 --token-file admin.token status
curl -H "Authorization: Bearer $(cat admin.token)" http://10.0.0.2:9000/status
```

- **Every request needs the token,** as `Authorization: Bearer TOKEN` in one field. It is compared in constant time. Anything else gets `401` with `WWW-Authenticate: Bearer`.
- **The token file** is read once, at startup. It holds one line of 16 to 1024 visible characters; a trailing newline is dropped. It must be a regular file of the user xenia runs as, that group and others can neither read nor write (`chmod 600`). If any of this fails, xenia does not start.
- **Both options are needed:** either alone is a usage error. They work beside `--admin` or without it, and `GET /status` shows each listener.
- `xenia ctl` takes `--admin-listen` and `--token-file`, or `$XENIA_ADMIN_LISTEN` and `$XENIA_ADMIN_TOKEN_FILE`.

**Live changes.** Flag changes and drains each publish a new generation, built from the active one; a request that changes nothing publishes nothing.

- **Reloads keep them.** A reload builds the config with them on top. It drops, with a log line, a flag the config no longer defines and a drain for an address no group lists.
- **Restarts do not.** A restart starts from the config alone.
- **Slow clients make them wait.** Each generation holds every route body, and a client too slow to read pins the generation that answered it and every later one. So live changes are refused with `503` while 8 earlier snapshots are still in use.

**Draining.** A drained server gets no new requests from any group that lists it.

- Requests in flight finish, and their connections then close rather than go back to the pool. Idle connections to it close at once.
- When the other servers of a group are all down, the one back soonest among them is tried, never a drained one.
- A group whose servers are all drained answers `503 Service Unavailable`.

**Requests.**

- Each connection takes one request, without a body. At most 16 connections are served at once on each listener (the socket, TCP and metrics each have their own, so idle clients of one cannot keep `xenia ctl` out of another), each for at most 10 seconds.
- `POST /reload` waits for a compile that starts after the request arrived, so it reads the file as it is by then.
- `GET /upstreams` and `GET /metrics` ask every worker what it knows. After two seconds they answer with what they have, and say which workers did not answer.

**The socket file.** At startup, a socket file left by a server that is gone is replaced. A socket a running server answers on, or any other file at the path, stops startup. The file is removed when the drain starts, unless another server has replaced it by then.

`xenia ctl` exits 0 for a 2xx reply, 1 for any other (the error goes to stderr) or no reply, and 2 for a usage error.

Not yet: editing routes and upstreams live, live changes that outlast a restart, TLS on the admin listener.

## Access logs

`--access-log FILE` appends one JSON line per request to `FILE`. The file is created with mode 0640, since lines hold query strings, and never opened through a symbolic link. `--access-log -` writes the lines to standard output instead, as containers collect it, after the `listening on` lines xenia prints there at start; reopening does nothing for it.

```json
{"time":"2026-10-12T06:02:47.123Z","request_id":"5f0c9e1ab27d4c6e8f3a1b2c3d4e5f60","worker":2,"gen":7,"client":"203.0.113.7:51234","tls":true,"host":"example.com","method":"GET","target":"/a?b=1","proto":"HTTP/1.1","status":200,"bytes":5120,"request_bytes":0,"duration":0.012345,"upstream":"10.0.0.5:443","retried":false,"aborted":false,"referer":null,"user_agent":"curl/8.7.1"}
```

| Field | |
| --- | --- |
| `time` | when the request head was read, UTC, to the millisecond |
| `request_id` | the request's [ID](#request-ids) |
| `worker`, `gen` | the worker that served it, and the snapshot generation |
| `client` | the client's address and port; when [trusted proxies](configuration.md#rate-limits) name the client, its address alone |
| `peer` | only then: the connection's address and port, the proxy's |
| `tls` | over HTTPS |
| `host` | the request's host, lowercase and without the port; `null` without one |
| `method`, `target`, `proto` | as sent (`HTTP/2.0` for HTTP/2, whose target is its `:path`); `null` for a request too malformed to parse |
| `status` | what the client got; `null` if no response began |
| `bytes` | response bytes handed to the connection, head included |
| `request_bytes` | request body bytes read, for proxied requests |
| `duration` | seconds from the request head to the last byte handed over |
| `upstream` | the upstream server tried last |
| `retried` | sent again after a pooled connection turned out closed |
| `aborted` | the response was cut off, or the client left before it was complete |
| `referer`, `user_agent` | those request headers |

**Content.**

- Strings are escaped and malformed UTF-8 becomes U+FFFD, so every line is valid JSON.
- `target` is cut at 4 KiB, and `referer` and `user_agent` at 1 KiB. A cut field ends with `...`.
- Connections that send no request leave no line, such as a failed handshake or an idle close. They show in the counters.

**Writing.**

- Each worker puts its lines in a 1 MiB buffer of its own and never waits for the disk.
- A writer thread writes the lines out at least every 100 ms, so lines from different workers can be out of order by about that much.
- When the disk falls behind and a buffer fills up, lines are dropped and counted (`/status`, `/stats`, `SIGUSR1`).
- A write error drops what could not be written, logged at most once a minute.
- At shutdown, the last lines get 2 seconds.

**Rotation.** Rename the file, then `kill -USR2` (or `xenia ctl reopen`). Xenia writes what it holds to the old file, then carries on in a new one. With logrotate:

```
/var/log/xenia/access.log {
    daily
    rotate 14
    compress
    delaycompress
    postrotate
        systemctl kill -s USR2 xenia.service
    endscript
}
```

Without systemd, `XENIA_ADMIN=/run/xenia/admin.sock xenia ctl reopen` (as root, or the user xenia runs as) does the same for exactly that server, where a `kill` by process name would reach every xenia on the host.

Not yet: per-route switches, other formats.

## Metrics

`--metrics HOST:PORT` serves `GET /metrics` in Prometheus' text format (0.0.4), for Prometheus to scrape. That listener has no token and serves nothing else: other paths are `404`, other methods `405`. The admin interface serves the same at `GET /metrics`.

```yaml
scrape_configs:
  - job_name: xenia
    static_configs:
      - targets: ['10.0.0.2:9100']   # xenia --metrics 10.0.0.2:9100
```

| Series | |
| --- | --- |
| `xenia_requests_total{host,route,method,code}` | responses by route and status class (`1xx` to `5xx`) |
| `xenia_request_duration_seconds{host,route,method}` | a histogram, from the request head to the last byte handed to the connection, in buckets from 0.5 ms to 10 s. A `101`'s tunnel is not timed. |
| `xenia_upstream_responses_total{server,code}` | response heads from each upstream server |
| `xenia_upstream_response_seconds{server}` | a histogram, from the request sent to the response head |
| `xenia_upstream_connect_failures_total{server}` | connects and TLS handshakes that failed |
| `xenia_upstream_errors_total{server,error}` | requests answered `502` (`bad_response`) or `504` (`timeout`) while at that server |
| `xenia_upstream_in_flight`, `xenia_upstream_idle_connections`, `xenia_upstream_marked_down` | per `server`, now: requests at it, pooled connections, and workers that set it aside for `downTime` |
| `xenia_upstream_up{upstream,server}`, `xenia_upstream_drained` | what health checks found (1 without them), and drains, per group |
| `xenia_COUNTER_total` | each counter of `/stats`, summed over the workers: `xenia_connections_total`, `xenia_proxied_total`, `xenia_status_502_total`, `xenia_tls_handshakes_total`, `xenia_files_total`, `xenia_compressed_total`, `xenia_status_429_total`, … |
| `xenia_open_connections`, `xenia_snapshot_generation`, `xenia_snapshots_retired` | client connections now, the active generation, and earlier ones still in use |
| `xenia_last_reload_success`, `xenia_last_reload_timestamp_seconds` | the last reload's outcome, once there has been one |
| `xenia_health_checks_total`, `xenia_health_checks_failed_total` | health checks run, and those that failed |
| `xenia_file_cache_bytes`, `xenia_access_log_write_errors_total` | the small-file cache in all workers, and access log lines lost to write errors |
| `xenia_ratelimit_requests_total{limit,result}`, `xenia_ratelimit_evicted_total{limit}` | requests each rate limit let through (`allowed`) and refused, and clients its full table forgot early |
| `xenia_resolve_lookups_total{result}`, `xenia_upstream_name_addresses{server,family}` | upstream [name](configuration.md#names) lookups that found addresses (`ok`) or not, and how many addresses each name has |
| `xenia_cache_requests_total{cache,result}` | requests on routes with a cache: `hit`, `miss`, `revalidated`, `stale`, `stale-error`, `bypass`; `revalidated` and `stale-error` are among the `miss`es they began as, so the results add up to more than the requests |
| `xenia_cache_refreshes_total{cache}`, `xenia_cache_refresh_failures_total{cache}` | refreshes started for `stale-while-revalidate`, and those that kept nothing |
| `xenia_cache_entries`, `xenia_cache_bytes`, `xenia_cache_stored_total`, `xenia_cache_evicted_total` | per `cache`: responses kept now and their bytes, responses stored or renewed, and those dropped to make room |
| `xenia_start_time_seconds`, `xenia_build_info{version}` | the process |
| `xenia_scrape_missing_workers` | workers that did not report within two seconds; their upstream and cache figures are left out of that scrape |

**Labels** come from the config, never from requests, so the config bounds the number of series.

- `host` is the route's host, empty for the default site. `route` is its path, or its prefix followed by `*`. `method` is its method, or `*` for any. A `HEAD` answered by a `GET` route counts as that route's.
- Requests that matched no route count under route `_none`: `404` for unknown paths, `405`, malformed requests and `408`. A file route's own `404` counts as that route's.
- A route keeps its series across reloads and live changes, since the same host, route and method are the same series. A removed route leaves the scrape, and if it comes back it picks up where it left off. After 2048 distinct routes in the life of the process, new ones share `_other`.
- `server` is the server as the config writes it: an address, or a name, whose addresses are summed under it. A server listed in several groups is counted once. `xenia_upstream_up` and `xenia_upstream_drained` are per group; for a name, up while any of its addresses is, and drained while all of them are.

**Cost.** Each worker counts into memory of its own, without locks: about 340 KiB per worker. A scrape asks every worker for its upstream figures, as `/upstreams` does.

## Request IDs

Every request gets an ID: 32 hex digits, 128 random bits, from a generator each worker seeds from the system at start.

- **Every response** carries it as `x-request-id`, errors and redirects included.
- **A proxied request** carries it upstream as the only `X-Request-Id`: a client's is dropped. An upstream's `x-request-id` in its response is replaced with it, so the client sees the one in the logs.
- **The access log** has it as `request_id`.

With `server: { trustRequestId: true }`, a client's `X-Request-Id` is kept instead, if it is well-formed: one field, of 1 to 128 characters from `A-Z a-z 0-9 . _ : -`. Otherwise xenia makes one as usual. Turn it on only behind a proxy that sets the field itself, since clients could choose it otherwise.
