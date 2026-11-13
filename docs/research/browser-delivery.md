# How would a hand-rolled app learn about new notifications in the browser?

Research for [issue #7](https://github.com/adamp01/cohort-005-project/issues/7). This note covers how a new notification could reach a Cadence tab that is already open, if we build the notification system ourselves. It compares four options: polling, React Router revalidation, server-sent events (SSE) and WebSockets. It does not make the decision.

Versions checked: `react-router`, `@react-router/serve`, `@react-router/express` and `@react-router/node` 8.3.0; `better-sqlite3` 12.11.1; `compression` 1.8.2; Node v26 docs. Local `node_modules` paths are given at those versions. Web sources were read on 13 Nov 2026.

## Summary

- **Polling a small resource route** (plain `fetch` or `useFetcher().load()` on a timer) needs nothing beyond what Cadence already runs. Each poll is one short request that runs one loader. A new notification shows up at the next tick, so the delay is up to one interval. Browsers slow timers down in hidden tabs.
- **`useRevalidator().revalidate()` on a timer or on focus** is supported by React Router and recommended in its own docs. But in framework mode it re-runs the loader of *every* route on the page, including the heavy `layout.app.tsx` loader, unless each route opts out with `shouldRevalidate`.
- **SSE through a resource route** already works under `react-router-serve`. The Express adapter flushes headers for `text/event-stream` and flushes after every chunk, and we confirmed that events arrive promptly even through the `compression()` middleware. The real work is (a) fanning a write out to the streams that are open, (b) heartbeats so proxies don't close idle connections, and (c) the six-connection limit browsers apply on HTTP/1.1.
- **WebSockets** can't be served by `react-router-serve`. It never exposes its HTTP server, so nothing can listen for the `upgrade` event. We would need a custom server and a WebSocket server library (Node has a built-in WebSocket *client* but no server). WebSockets have the same fan-out problem as SSE.
- **More than one instance.** Polling and revalidation just read SQLite, so they work from any process that can open the file. Push options (SSE, WebSockets) need a write in one process to reach a connection held by another. SQLite has no notification feature that crosses connections, so each process would have to watch the database or use an external broker. [ADR 0006](../adr/0006-one-sqlite-file-through-drizzle.md) already says Cadence can't run on more than one machine against the same data.

## What Cadence runs today

- `npm start` runs `react-router-serve ./build/server/index.js` (`package.json`).
- `react-router-serve` creates an Express 5 app, adds `compression()` and `morgan`, sends every route to `@react-router/express`'s `createRequestHandler`, and calls `app.listen()`. It keeps the returned server to itself and only uses it to close on SIGTERM/SIGINT. Nothing lets app code reach the HTTP server or add middleware (`node_modules/@react-router/serve/dist/cli.js`, the `run()` function).
- The app layout loader (`app/routes/layout.app.tsx`) runs several queries on every load: all users, the current user, recently progressed courses with a progress calculation for each, and team-admin status.
- The database is one `better-sqlite3` connection per process, opened lazily, in WAL mode (`app/db/index.ts`). ADR 0006 records the decision: "the app cannot run on more than one machine against the same data" ([ADR 0006](../adr/0006-one-sqlite-file-through-drizzle.md)).
- Sessions are cookie-based (`createCookieSessionStorage`, `app/lib/session.ts`). That matters for how each transport authenticates (see below).

## Option 1: Poll a notifications resource route

**How it works.** Add a resource route, for example `api/notifications/unread`: a module with a `loader` and no default export ([React Router docs: Resource Routes](https://reactrouter.com/how-to/resource-routes); local copy `node_modules/react-router/docs/how-to/resource-routes.md`). The bell calls it on a timer, either with `fetch` or with `useFetcher().load()`.

**What React Router does with a fetcher load.** A fetcher load goes through `singleFetchLoaderFetcherStrategy`, which requests only the fetcher's route (`fetchAndDecode(args, [routeId])`, which sets `_routes=<id>` on the `.data` URL). No other loader runs (`node_modules/react-router/dist/development/lib/dom/ssr/single-fetch.js`, `singleFetchLoaderFetcherStrategy` and `fetchAndDecodeViaTurboStream`). The docs say `fetcher.load` data revalidates only "after action submissions and explicit revalidation requests via `useRevalidator`" (`node_modules/react-router/docs/how-to/optimize-revalidation.md`). React Router has no built-in interval; we would write the timer.

**Cost.** Each open tab sends one request per interval. Every request parses the session cookie and runs one synchronous SQLite query. The cost scales with open tabs ÷ interval: for example, 500 tabs polling every 30 s is about 17 requests per second, mostly returning "nothing new". `better-sqlite3` says it is "fast enough to use in the main thread without blocking for a noticeable amount of time" for most apps ([better-sqlite3 docs: threads](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/threads.md)).

**Delay.** Up to one interval. Browsers enforce minimum timer delays in inactive tabs: Firefox desktop waits at least 1 s; Chrome checks throttled timers once per second, and after 5 minutes hidden ("intensive throttling") once per minute; Firefox for Android waits at least 15 minutes ([MDN: setTimeout, "Timeouts in inactive tabs"](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout#timeouts_in_inactive_tabs)). The Page Visibility API's `visibilitychange` event lets the app pause polling while hidden and poll immediately when the tab comes back ([MDN: Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)).

**Hosting.** Short requests only, so proxy idle timeouts don't apply. Every process reads the same SQLite file, so it works with several processes on one host. WAL needs all processes "on the same host computer; WAL does not work over a network filesystem" ([SQLite: WAL](https://www.sqlite.org/wal.html)).

**Complexity.** Lowest. One resource route, one hook, and an optional visibility handler. No change to the server.

## Option 2: React Router revalidation (`useRevalidator`) on an interval or on focus

**How it works.** `useRevalidator()` returns `revalidate()` and a `state`. Its JSDoc says it is for revalidating "for reasons outside of normal data mutations like Window focus or polling on an interval", and its example uses a focus hook the app supplies itself (`node_modules/react-router/dist/development/lib/hooks.d.ts`, `useRevalidator`). Nothing calls it automatically on focus or on a timer.

**What it re-runs.** `revalidate()` starts a navigation to the current location (`router.js`, `revalidate()`, which calls `startNavigation(..., { startUninterruptedRevalidation: true })`). In `getMatchesToLoad`, a same-URL navigation sets `defaultShouldRevalidate = true` for every matched route with a loader, and each route's `shouldRevalidate` is consulted (`node_modules/react-router/dist/development/lib/router/router.js`, `getMatchesToLoad`, around lines 1960–1971). Single fetch then sends **one** `.data` request for all those routes. It adds `_routes=` only when some route opted out (`single-fetch.js`, `singleFetchLoaderNavigationStrategy`). Loaded fetchers are revalidated too. The docs say that in framework mode, loaders "are automatically revalidated after all navigations and form submissions", and `shouldRevalidate` is how a route opts out ([React Router docs: Route Module, shouldRevalidate](https://reactrouter.com/start/framework/route-module#shouldrevalidate); local `node_modules/react-router/docs/start/framework/route-module.md`).

**Cost.** For Cadence, each tick re-runs the `layout.app.tsx` loader (the multi-query sidebar data) plus the current page's loader. That page might be a lesson, a dashboard or an analytics view. So each poll costs as much as reloading the page's data, not just a notification-count query. The cost could be cut by adding `shouldRevalidate` to routes. But the same function also controls revalidation after navigations and actions, and the type docs warn that returning anything other than `defaultShouldRevalidate` risks the UI getting "out of sync with your data on the server" (`node_modules/react-router/dist/development/lib/router/utils.d.ts`, line ~310).

**Side effects.** `revalidate()` calls `interruptActiveLoads()` and sets `revalidation: "loading"` (`router.js`, `revalidate()`). A timer can therefore cut into an in-flight load. Anything that reads `revalidator.state` or navigation state may also flicker on every tick.

**Delay and hosting.** Same as Option 1: interval-bound, slowed down in hidden tabs, short requests only, multi-process-safe on one host.

**Complexity.** Low in code (a few lines). The cost is the extra load on every tick and the interaction with every route's `shouldRevalidate`. Revalidating *on focus* only (no timer) is cheaper. It catches up when the user returns to the tab, but won't update a tab the user is looking at.

## Option 3: Server-sent events from a resource route

**How it works.** A resource-route `loader` returns `new Response(readableStream, { headers: { "Content-Type": "text/event-stream" } })`. The browser opens it with `new EventSource(url)` and gets `message` events ([WHATWG HTML §9.2 Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)).

**Does it work under `react-router-serve`?** Yes. We checked this in source and ran a test:

- `sendRemixResponse` in `@react-router/express` calls `res.flushHeaders()` when the `Content-Type` matches `text/event-stream` (`node_modules/@react-router/express/dist/index.js`). The changelog entry is "Flush headers for `text/event-stream` responses" (`node_modules/@react-router/express/CHANGELOG.md`, [remix#7619](https://github.com/remix-run/remix/pull/7619)).
- The body is piped by `writeReadableStreamToWritable`, which calls `writable.flush()` after every chunk if one exists (`node_modules/@react-router/node/dist/index.js`).
- `react-router-serve` adds `compression()` (`cli.js`). `compression` treats any `text/*` type as compressible (`node_modules/compressible/index.js`, `COMPRESSIBLE_TYPE_REGEXP`). So SSE responses get gzipped unless the response sets `Cache-Control: no-transform` (`node_modules/compression/index.js`, `shouldTransform`). `compression` also provides `res.flush()`, which flushes the zlib stream (same file). So each event still reaches the client right away.
- **Test we ran:** we put Express 5 + `compression()` + `writeReadableStreamToWritable` in front of a stream that sent one event per second, and used `curl --compressed`. The response came back `Content-Encoding: gzip`, and the events arrived one second apart rather than all at the end.
- When the client disconnects, the adapter aborts the `Request`'s `AbortSignal` (`res.on("close", () => controller?.abort())` in `createRemixRequest`). The loader can listen for that and unsubscribe.

**Auth.** `EventSource` makes a "potential-CORS request" with the CORS setting "Anonymous", unless `withCredentials` is set ([WHATWG HTML §9.2.2](https://html.spec.whatwg.org/multipage/server-sent-events.html#dom-eventsource)). That is the same-origin credentials mode, so the session cookie is sent on a same-origin stream and the loader can call `getCurrentUserId(request)` as normal.

**Reconnection and catch-up.** The browser reconnects automatically after a disconnect. The server can set the reconnection delay with a `retry:` field and tag events with `id:`. On reconnect the browser sends the last ID in the `Last-Event-ID` request header, so the server can resend anything missed. Responding with HTTP 204 tells the client to stop reconnecting ([WHATWG HTML §9.2.1, §9.2.4, §9.2.6](https://html.spec.whatwg.org/multipage/server-sent-events.html)).

**Long-lived connections and proxies.**

- The spec says: "Legacy proxy servers are known to, in certain cases, drop HTTP connections after a short timeout. To protect against such proxy servers, authors can include a comment line (one starting with a ':' character) every 15 seconds or so" ([WHATWG HTML §9.2.7 Authoring notes](https://html.spec.whatwg.org/multipage/server-sent-events.html#authoring-notes)). The same notes warn that chunking done "by a different layer unaware of the timing requirements" can hurt reliability.
- Some common idle limits that would close a quiet stream without heartbeats:
  - nginx `proxy_read_timeout` defaults to 60 s between two reads. `proxy_buffering` is on by default and can be turned off per response with `X-Accel-Buffering: no` ([nginx proxy module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)).
  - AWS ALB's idle timeout defaults to 60 s ([AWS ALB attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html)).
  - Heroku requires the first byte within 30 s, then a rolling 55 s window where "each byte sent… resets" it ([Heroku HTTP routing](https://devcenter.heroku.com/articles/http-routing)).
- Node itself won't close the stream: `server.timeout` defaults to 0 (no timeout), and `server.requestTimeout` (300 s) only covers *receiving* the request ([Node: http.Server](https://nodejs.org/api/http.html#serverrequesttimeout)). `react-router-serve` doesn't change either.
- Each open tab holds one socket and one stream on the server for as long as it stays open.

**Browser connection limit.** Over HTTP/1.1, browsers allow 6 connections per browser and domain, shared by all tabs. Chrome and Firefox have marked this "Won't fix". Over HTTP/2 the limit is negotiated (default 100 streams) ([MDN: EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)). The spec suggests sharing one `EventSource` between pages (for example, via a shared worker) or letting users switch it off per page ([WHATWG HTML §9.2.7](https://html.spec.whatwg.org/multipage/server-sent-events.html#authoring-notes)). `react-router-serve` uses Express's `app.listen()` (plain HTTP/1.1, `cli.js`). Whether the browser gets HTTP/2 depends on whatever proxy terminates TLS in front of it.

**Complexity.** Medium. The resource route itself is small. The work is fan-out (below), heartbeats, `Last-Event-ID` catch-up, cleanup on abort, and the connection limit.

## Option 4: WebSockets

**How it works.** The browser opens `new WebSocket("wss://…")`. The connection starts as an HTTP request and switches protocol with `Upgrade` ([WHATWG WebSockets Standard](https://websockets.spec.whatwg.org/); RFC 6455). It is two-way. The browser `WebSocket` interface "doesn't support backpressure" ([MDN: WebSockets API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)), which barely matters at notification volumes.

**Under `react-router-serve`.** Not possible as shipped. A Node server handles upgrades through the `'upgrade'` event on its `http.Server` ([Node: http, Event 'upgrade'](https://nodejs.org/api/http.html#event-upgrade)). `react-router-serve` keeps its server private and provides no hooks (`cli.js`). Its handler is a fetch-style `Request → Response` function, and a resource route has no API for taking over the socket. Node has a global `WebSocket` *client* (stable since v22.4.0) ([Node: globals, WebSocket](https://nodejs.org/api/globals.html#class-websocket)), but no built-in WebSocket server. So this option means:

1. Replacing `react-router-serve` with a custom Express server, the documented path. React Router ships a "Node with Docker (Custom Server)" template with a "custom express server for more control" ([React Router docs: Deploying](https://reactrouter.com/start/framework/deploying); local `node_modules/react-router/docs/start/framework/deploying.md`). The server would rebuild what `cli.js` does (static assets, compression, logging, the React Router handler).
2. Adding a WebSocket server library on that server, which is a new third-party dependency.
3. Authenticating the upgrade request ourselves by reading the session cookie outside React Router's loaders.
4. Handling WebSocket connections in development, where `react-router dev` runs Vite rather than `react-router-serve`. We did not look into this.

**Proxies.** `Upgrade` is a hop-by-hop header, so reverse proxies need explicit configuration. nginx closes the connection "if the proxied server does not transmit any data within 60 seconds" unless `proxy_read_timeout` is raised or the server sends ping frames ([nginx: WebSocket proxying](https://nginx.org/en/docs/http/websocket.html)). ALB and Heroku idle windows apply as they do for SSE (sources above). Browsers leave pages with open WebSockets unthrottled to avoid timeouts ([MDN: Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)).

**Reconnection.** The `WebSocket` API doesn't reconnect automatically, and there's nothing like `Last-Event-ID`. Reconnect and catch-up logic is ours to write (compare the SSE features in [WHATWG HTML §9.2](https://html.spec.whatwg.org/multipage/server-sent-events.html)).

**Complexity.** Highest. Server replacement, a new dependency, our own auth and reconnection, and the same fan-out problem as SSE. Its main advantage, two-way messaging, isn't needed for a bell and badge.

## Cross-cutting: how does a write in one request reach a stream held by another?

This only matters for SSE and WebSockets. With polling and revalidation, the next request simply reads the new row.

**Within one process.** The request that inserts a notification and the request holding the user's stream live in the same Node process. An in-memory publish/subscribe mechanism, such as a Node `EventEmitter` ([Node: events](https://nodejs.org/api/events.html)) keyed by user ID, can pass the event straight to the stream's controller. The stream unsubscribes when `request.signal` aborts. This is enough as long as Cadence stays a single process, as ADR 0006 assumes.

**Across processes (several instances against one file on one host).** In-memory pub/sub doesn't cross processes. SQLite has no notification that crosses connections:

- `sqlite3_update_hook` registers a callback "with the database connection identified by the first argument", so it only reports changes made through *that* connection ([SQLite: sqlite3_update_hook](https://www.sqlite.org/c3ref/update_hook.html)). The `better-sqlite3` API doesn't expose it at all ([better-sqlite3 API](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/api.md)).
- `PRAGMA data_version` is the cheap way to detect a change: values from two calls on the same connection "will be different if changes were committed to the database by any other connection in the interim", including connections in other processes ([SQLite: PRAGMA data_version](https://www.sqlite.org/pragma.html#pragma_data_version)). It only says *something* changed. The process would still have to query for new notifications.
- So each process would poll SQLite on a short server-side timer (one timer per process, not per tab) and push to its own streams, or the app would add an external message broker. Either way, the server ends up polling or we run more infrastructure.

**Across machines.** Ruled out by the database itself: WAL needs all processes on one host and doesn't work over a network filesystem ([SQLite: WAL](https://www.sqlite.org/wal.html)). ADR 0006 records the same limit. Moving to multiple machines means changing the database, and at that point the fan-out design would be revisited anyway.

**Write contention.** Notification inserts add writes. SQLite allows "only one writer at any instant in time", but "writers queue up" and a write usually takes milliseconds ([SQLite: Appropriate Uses](https://www.sqlite.org/whentouse.html)). WAL lets readers and writers proceed concurrently ([SQLite: WAL](https://www.sqlite.org/wal.html)). `better-sqlite3` notes that constant overlapping reads can starve WAL checkpoints and suggests `wal_checkpoint(RESTART)` if the WAL file grows ([better-sqlite3: performance](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/performance.md)). Only server-side polling of the database every few hundred milliseconds would make that a realistic concern.

## Comparison

| | Polling a resource route | `useRevalidator` on timer / focus | SSE resource route | WebSockets |
|---|---|---|---|---|
| Delay | Up to one interval; slowed in hidden tabs | Up to one interval (or next focus) | About immediate | About immediate |
| Server work per tick or event | 1 small loader per tab per interval | Every active route's loader per tab per interval | 1 write per event per open stream, plus heartbeats | 1 frame per event per socket, plus pings |
| Works with `react-router-serve` as-is | Yes | Yes | Yes (confirmed by test, through `compression()`) | No; needs a custom server and a library |
| New dependencies | None | None | None | WebSocket server library |
| Long-lived connections | No | No | Yes (1 per tab, or 1 per browser if shared) | Yes (1 per tab) |
| Proxy / idle timeouts | Not affected | Not affected | Needs heartbeats (≤15 s per spec); may need proxy buffering off | Needs Upgrade-aware proxy and pings |
| Browser connection limit | Normal requests | Normal requests | 6 per browser+domain on HTTP/1.1 | No equivalent cap found in the sources read (MDN documents the 6-connection cap for SSE only) |
| Reconnect / catch-up | Built in (next poll) | Built in (next revalidation) | Automatic reconnect + `Last-Event-ID` | Write it ourselves |
| Auth with cookie session | Normal loader | Normal loader | Normal loader (same-origin cookies sent) | Our own check on the upgrade request |
| Several processes, one host | Works | Works | Needs cross-process fan-out (DB polling or broker) | Same as SSE |
| Several machines | Blocked by SQLite / ADR 0006 | Same | Same | Same |
| Rough complexity | Low | Low code; side effects across routes | Medium | High |

## Implications for the decision

- **Freshness requirement.** The pull options (1, 2) and the push options (3, 4) differ mainly in how quickly the badge updates on a visible tab. If a delay of tens of seconds is acceptable, polling needs no server or hosting changes. If near-instant matters, SSE is the push option that fits the current server.
- **If we pick polling, choose the endpoint carefully.** Polling a dedicated resource route (Option 1) costs one small query per tick. Revalidating the whole page (Option 2) re-runs the layout and page loaders on every tick, and reducing that means touching `shouldRevalidate` on many routes. Revalidating only on focus is cheap, but doesn't update a visible tab.
- **SSE's cost is operational, not framework support.** Heartbeats, proxy buffering and timeouts, `Last-Event-ID` catch-up, and the HTTP/1.1 six-connection limit across tabs all need answers. Several depend on the hosting target, which isn't chosen yet.
- **WebSockets add work SSE doesn't for this feature.** Server replacement, a new dependency (Medium security risk under our PR rules), and our own reconnection and auth. The extra capability, two-way messaging, isn't something a bell needs.
- **Push options and scaling.** In-process fan-out is simple, but only while Cadence stays one process. Going to several processes means server-side database polling or a broker. Going to several machines is blocked by the database choice in ADR 0006, whatever transport we pick.
- **Hybrids are possible.** For example, SSE or focus-revalidation for the badge combined with a loader that recomputes the true unread count. Each piece inherits the constraints above.
- **Build vs buy.** A hosted notification or realtime service would take over the long-lived connections, fan-out and multi-instance concerns, which are exactly the parts that make Options 3 and 4 costly here. Polling makes those concerns disappear without buying anything.

## Sources

Local, at installed versions:

- `node_modules/@react-router/serve/dist/cli.js`: server setup, `compression()`, private `app.listen()`
- `node_modules/@react-router/express/dist/index.js`: `flushHeaders()` for `text/event-stream`, abort on close
- `node_modules/@react-router/express/CHANGELOG.md`: "Flush headers for `text/event-stream` responses"
- `node_modules/@react-router/node/dist/index.js`: `writeReadableStreamToWritable` flushes after each chunk
- `node_modules/compression/index.js`, `node_modules/compressible/index.js`: compressibility of `text/*`, `no-transform`, `res.flush()`
- `node_modules/react-router/dist/development/lib/hooks.d.ts`: `useRevalidator` docs
- `node_modules/react-router/dist/development/lib/router/router.js`: `revalidate()`, `getMatchesToLoad`
- `node_modules/react-router/dist/development/lib/dom/ssr/single-fetch.js`: single-fetch navigation and fetcher strategies, `_routes`
- `node_modules/react-router/docs/`: `how-to/resource-routes.md`, `how-to/optimize-revalidation.md`, `start/framework/route-module.md`, `start/framework/deploying.md`
- Cadence: `package.json`, `app/routes.ts`, `app/routes/layout.app.tsx`, `app/db/index.ts`, `app/lib/session.ts`, `docs/adr/0006-one-sqlite-file-through-drizzle.md`

Web:

- WHATWG HTML Living Standard, §9.2 Server-sent events: https://html.spec.whatwg.org/multipage/server-sent-events.html
- WHATWG WebSockets Standard: https://websockets.spec.whatwg.org/
- MDN EventSource: https://developer.mozilla.org/en-US/docs/Web/API/EventSource
- MDN WebSockets API: https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
- MDN setTimeout (inactive tabs): https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout#timeouts_in_inactive_tabs
- MDN Page Visibility API: https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API
- Node.js http: https://nodejs.org/api/http.html (`'upgrade'`, `server.timeout`, `server.requestTimeout`)
- Node.js globals, WebSocket: https://nodejs.org/api/globals.html#class-websocket
- Node.js events: https://nodejs.org/api/events.html
- SQLite WAL: https://www.sqlite.org/wal.html
- SQLite PRAGMA data_version: https://www.sqlite.org/pragma.html#pragma_data_version
- SQLite sqlite3_update_hook: https://www.sqlite.org/c3ref/update_hook.html
- SQLite Appropriate Uses: https://www.sqlite.org/whentouse.html
- better-sqlite3 docs: https://github.com/WiseLibs/better-sqlite3/tree/master/docs (`api.md`, `threads.md`, `performance.md`)
- React Router docs: https://reactrouter.com/how-to/resource-routes, https://reactrouter.com/start/framework/route-module#shouldrevalidate, https://reactrouter.com/start/framework/deploying
- nginx proxy module: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- nginx WebSocket proxying: https://nginx.org/en/docs/http/websocket.html
- AWS ALB attributes: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html
- Heroku HTTP routing: https://devcenter.heroku.com/articles/http-routing

Not investigated: serverless and edge platforms (Vercel, Netlify, Cloudflare), since a single local SQLite file already limits hosting to a persistent server with a disk; WebSocket handling under `react-router dev`; and hosted notification products (out of scope for this ticket).
