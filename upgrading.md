# Upgrading

What each release changes that a config or an operator has to know or act on, newest first. Read every section after the release you run, up to the one you install; a release that changes nothing of the kind has none. 0.15.1 was the first public release. Each release's notes, and the [changelog](changelog.md), have the rest.

After any upgrade:

- run `xenia check` on your config with the new binary before you restart or reload, since a stricter release may refuse what an older one took;
- run `xenia types` again in your project, so editors check the config against the new release's types.

## 0.16.0

- **HTTP/1.1 is stricter, against request smuggling.** Requests 0.15 took are now refused: a second `Content-Length`, even with the same value (`400`); `Transfer-Encoding` with empty list elements (`501`) or a tab (`400`); whitespace after a chunk size, and chunk extensions outside their grammar; trailers that are not `name: value`, or that name `Content-Length`, `Transfer-Encoding` or `Connection`; and absolute-form targets with userinfo (`400`). Clients that follow the standard send none of these: if one of yours starts getting `400` or `501`, its framing is why. The chunked rules hold for upstream responses too.
- **A client that leaves while it waits for the upstream ends the exchange.** When a client closes its connection, or only its side of it, after its request and before the response begins, xenia now closes the upstream connection too. Upstreams see such requests cancelled, and each has an access log line with `"status": null` and `"aborted": true`. A client that half-closes on purpose after sending its request no longer gets an answer; once the answer has begun, it still does.
- **Redirects behind port mapping.** With `tls.redirect`, redirects name the port of the first `--listen-tls`, as before. Behind a mapping such as `-p 443:8443`, set the new `tls.redirectPort: 443` so they name the port clients reach.
- **`GET /snapshot`.** A limit's `rate` is `null` when it has only `inFlight`, and the JSON has new keys: `file` on file routes, `inFlight` and its counts on limits, `maxConnectionsPerClient` in the server settings. A script that reads it should expect them.
- **Docs.** "Rate limits" is now [Limits](configuration.md#limits); links to `#rate-limits` still land there.
- **From a release archive to the Debian package.** The package puts the binary in `/usr/bin/xenia` and its unit in `/usr/lib/systemd/system`. The archive's install steps put them in `/usr/local/bin/xenia` and `/etc/systemd/system/xenia.service`, and a unit there wins over the package's, so it would keep starting the old binary. Before installing the package, stop the service and remove both, keeping any listener changes as a `systemctl edit xenia` override. `/etc/xenia` stays as it is: dpkg asks before it replaces a config file you already have, and keeps yours unless told otherwise.
