# morojs

MoroJS on its native HTTP engine, clustered by the framework itself.

## Stack

- **Language:** JavaScript
- **Runtime:** Node.js 26
- **Framework:** [MoroJS 1.8](https://github.com/Moro-JS/moro) on `@morojs/engine` 1.1.14, its own native HTTP engine
- **Build:** Multi-stage on `node:26-trixie-slim`; the engine ships prebuilt binaries, so no toolchain

## Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/pipeline` | GET | Returns `ok` (plain text) |
| `/baseline11` | GET/POST | Sums query parameter values, plus the body for POST |
| `/baseline2` | GET | Sums query parameter values (behind the gateway proxies) |
| `/delay/:ms` | GET | Waits `ms` milliseconds on an awaited timer, then returns `ms` |
| `/json/:count` | GET | Serializes a slice of the dataset, brotli or gzip when the client asks for it |
| `/echo` | POST | Returns the request body back verbatim (TLS listener) |
| `/ws` | WebSocket | Echoes every text and binary frame unchanged |
| `/async-db` | GET | Reads from PostgreSQL through `pg`, prepared statement, pool sized under `DATABASE_MAX_CONN` |
| `/fortunes` | GET | Reads 200 rows from PostgreSQL, appends the runtime row, sorts and renders `views/fortunes.ejs` |
| `/static/:filename` | GET | Serves a file from disk through the framework's `staticFiles` middleware (TLS listener) |
| `/public/baseline`, `/public/json/:count` | GET | The baseline and JSON routes as the production-stack edge forwards them, no auth |
| `/api/items/:id` | GET/POST | Cache-aside read with `X-Cache: HIT` or `MISS`, or a JSON-body update that invalidates the entry and answers 204 |
| `/api/me` | GET | Reads the user named by `X-User-Id` through the same cache-aside |

## Notes

- Standard mode: the app is `createApp()` with its defaults on the native engine (`engine: 'moro'`,
  which is also the default). Request tracking and request logging are turned off through their
  documented options; nothing else is configured.
- `/pipeline` is a literal handler, `.handler('ok')`: the framework's static route, answered inside
  the engine without entering JS, with the same `text/plain` reply `res.send('ok')` would give.
- Clustering is Moro's own (`performance.clustering`): one worker per core, which on the native
  engine means worker threads that each bind the port through `SO_REUSEPORT`. The count is passed
  in explicitly from the cgroup quota, because Moro's `auto` counts every host core even under
  `--cpuset-cpus`.
- `json-comp` is the framework's `compression()` middleware attached to the `/json` route alone,
  so no other endpoint pays for the encoder. It negotiates off `Accept-Encoding` per request and
  leaves the body alone when the header is absent. gzip at level 3 is preferred over brotli
  through the middleware's `encodings` and `gzip` options: the profile scores bytes squared
  against rate, and on these bodies level 3 costs a little over a third of the CPU of the default
  level 6 for 9% more bytes, and about half the CPU of brotli at quality 3 for 10% more. The
  middleware keeps one gzip encoder per worker and resets it between bodies (`reuseEncoders`, the
  default), so no response pays for building one. brotli at quality 3, the setting the other JS
  entries use, remains available to a client that accepts nothing else.
- `UV_THREADPOOL_SIZE=64` is set in the Dockerfile. The libuv pool serves the compression and
  file reads of every worker thread in the process; Moro sets 64 itself but only at `listen()`,
  after its startup has initialised the pool at Node's default of 4, and Node reads the variable
  only at process start.
- `json-tls`, `static-tls` and `8gbit` listen on `8081` behind the engine's own TLS listener
  (TLS 1.3, ALPN `http/1.1`), when `/certs/server.crt` and `/certs/server.key` are mounted. Moro
  locks one configuration per process, so that listener is a second process running the same file
  with `MORO_ROLE=tls`, clustered the same way; it is not started when the certificates are absent.
- `tls_check` is opted into: a third process of the same file (`MORO_ROLE=tlscheck`) listens on
  `9000` reading `/certs-tls`, with `server.ssl.watch: true`, so a pair replaced on disk is
  validated and swapped onto the running listener without a restart and without touching
  connections already established. Only started when `/certs-tls` is mounted, which validate.sh
  alone does.
- `baseline-h2`, `static-h2`, `baseline-h2c` and `json-h2c` run on Moro's HTTP/2 server, which is
  Node's http2 module behind the framework's own request and response objects (`engine: 'node'`
  with `http2: true`); the native engine speaks HTTP/1.1 only. `:8443` terminates TLS with ALPN
  `h2` (HTTP/1.1 still accepted), `:8082` is cleartext prior-knowledge HTTP/2. Each is its own
  process for the same one-configuration-per-process reason, clustered as node:cluster processes,
  which is Moro's transport for its Node servers.
- `/echo` sends back `req.rawBody`, the framework's undecoded copy of the request body, so the
  bytes are the bytes that arrived, `Content-Length` or chunked alike; a Buffer handed to
  `res.send()` goes out `application/octet-stream` through the framework's prepared reply.
- `/ws` is the engine's own RFC 6455 support through `app.websocket()` with `{ raw: true }`, so a
  text frame reaches the handler as the string it carried and `socket.send()` returns it as one.
  The engine corks the connection for the duration of each read (`@morojs/engine` 1.1.14), so
  the echoes of a pipelined batch of frames leave in one write.
- Static files go through `staticFiles({ root: '/data/static', precompressed: true, cache: true })`:
  the framework's handler with its in-memory cache. A file and its `.br` and `.gz` sidecars are
  read once and served from memory with their `ETag` and `Last-Modified`, the sidecar negotiated
  per request from `Accept-Encoding`. The disk is stat'ed again at most once a second, and as soon
  as the directory reports a change, so a replaced file (same length or not) is served promptly.
  Without the cache the handler resolves, stats and reads the file and its sidecars on every
  request, each a libuv threadpool round trip shared by every worker thread of the process. It is
  mounted on the TLS listener behind a path check so the JSON route on that port does not pay for
  the lookup; the check calls `next()` synchronously, which keeps that route on the server's
  synchronous dispatch.
- `fortunes` goes through EJS directly: Moro ships no view layer, so the template is rendered by
  the engine's own `renderFile`, a separate file with `<%= %>` escaping every cell.
- The gateway and production-stack edges are stock nginx and Caddy, configured as the fulmine
  entry configures them: `/static/*` off disk at the edge, everything else forwarded over
  loopback h1 with a keepalive pool, and `/api/*` past the shared JWT verifier first.
- The production-stack cache-aside goes through the framework's cache adapters:
  `RedisCacheAdapter` on the stack's `REDIS_URL`, one connection per worker so an update on any
  worker invalidates what every other one reads, and `MemoryCacheAdapter` otherwise. Items live
  for at most a second, users for thirty.
