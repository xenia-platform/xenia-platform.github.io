# Deployment

Putting xenia on a Linux host as a service. Xenia runs in the foreground and logs to stderr, so a service manager starts it, restarts it and keeps its log; it never forks itself into the background. The examples use systemd, and say what changes without it.

## Layout

| Path | What | Owner, mode |
| --- | --- | --- |
| `/usr/local/bin/xenia` | the binary, from a [release archive](index.md#installing) or `cmake --install` | root, 755 |
| `/etc/xenia/xenia.conf.js` | the config, the modules it imports, and certificates and keys | root:xenia, 640 (the directory 750) |
| `/etc/xenia/admin.token` | the admin token, if the admin interface listens on TCP | xenia, 600 |
| `/var/log/xenia/access.log` | the [access log](running.md#access-logs) | xenia, in a directory xenia can write |
| `/run/xenia/admin.sock` | the [admin interface](running.md#admin-interface)'s socket | xenia, 600, made at start |

Xenia runs as a user of its own, with no shell or home:

```sh
useradd --system --no-create-home --shell /usr/sbin/nologin xenia
install -d -o root -g xenia -m 750 /etc/xenia
```

## Who reads what

Every config compile, the first and each reload's, runs as the user xenia serves as, in a child process. So that user must be able to read the config, every module it imports, every certificate, key, CA and CRL it names, and the files its routes serve. A key only root can read starts fine under root and makes every later reload fail: give its group read access instead. xenia never writes to any of them.

The token file is read once, at start, as that user, and must be owned by it with no access for group or others. The admin socket's directory and the access log's must be writable by the user: the socket is made there at start and removed at shutdown, and a [rotated](#log-rotation) log is created there anew.

## Ports below 1024

Binding them needs privileges; nothing after binding does. Three ways, the first preferred:

- **systemd** starts xenia as its user with only `CAP_NET_BIND_SERVICE`, as the [unit](#systemd) below does.
- **`--user USER[:GROUP]`**, for launchers that cannot: started as root, xenia binds every listener, then becomes the user for good (its groups, its group, the user, and `no_new_privs`) before anything else. See [Running as a user](running.md#running).
- **File capabilities**, `setcap cap_net_bind_service=+ep /usr/local/bin/xenia`, which let any user bind them with that binary; copying or upgrading the binary drops them.

Started as root without `--user`, xenia warns, and serves as root. `GET /status` reports the `uid` it serves as.

## systemd

[`contrib/systemd/xenia.service`](https://github.com/xenia-platform/releases/blob/main/contrib/systemd/xenia.service), installed by `cmake --install` under `share/xenia` and carried in release archives, is a unit to start from:

```sh
install -m 644 xenia.service /etc/systemd/system/xenia.service
systemctl daemon-reload
systemctl enable --now xenia
systemctl reload xenia        # recompile the config, and wait for the outcome
journalctl -u xenia           # xenia's log
```

- **`Type=notify-reload`** (systemd 253 or later): xenia reports [readiness](running.md#running) on `NOTIFY_SOCKET`, so `systemctl start` returns once it accepts connections, and `systemctl reload` once the reload is applied or has failed, with the reason in `systemctl status`. With an older systemd: `Type=notify` and `ExecReload=/bin/kill -HUP $MAINPID`.
- **`User=xenia`** with `AmbientCapabilities=CAP_NET_BIND_SERVICE`: the unit needs no `--user`.
- **`RuntimeDirectory=xenia` and `LogsDirectory=xenia`** make `/run/xenia` and `/var/log/xenia` for the user; `ProtectSystem=strict` leaves the rest of the system read-only to it.
- **`TimeoutStopSec`** should exceed the config's `server.shutdownTimeout` (10 s by default), so a stop drains in-flight responses before systemd ends it.

Edit `ExecStart` for the listeners: `--listen-tls 0.0.0.0:443 --listen-tls [::]:443` for HTTPS, `--metrics 127.0.0.1:9100` for Prometheus. `xenia ctl`, with `XENIA_ADMIN=/run/xenia/admin.sock`, works as root or as the xenia user.

## Log rotation

Rename the file, then have xenia reopen it: `systemctl kill -s USR2 xenia.service`, or `xenia ctl reopen`. A logrotate entry:

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

## Behind a load balancer

The client of a request that came through a proxy or load balancer is the address that proxy says it was for. List the proxies in `server.trustedProxies` (addresses or prefixes, up to 64), and xenia takes the client from their `X-Forwarded-For`: for [rate limits](configuration.md#rate-limits), the `X-Forwarded-For` it sends upstream, and the access log's `client` (with `peer`, the proxy). Requests from anywhere else are taken at their own address, whatever they send.

```js
export default defineConfig({
  server: { trustedProxies: ['10.0.0.0/8'] },
  // ...
});
```

## Without systemd

Any supervisor that runs a foreground process will do (runit, s6, supervisord, a container runtime). Start xenia as root with `--user xenia`, or as the user with file capabilities, and give it the directories above. Without `NOTIFY_SOCKET` nothing is reported; `SIGHUP` reloads, `SIGTERM` drains and exits, and `GET /status` tells how the last reload went.
