# Live updates for Notifications: SSE vs polling vs WebSockets

Researched 2026-11-10 against react-router / @react-router/* 8.3.0 and compression 1.8.2 as installed in `node_modules`.

## Question

How should an open Cadence tab learn about a new Notification without a reload? We need a toast plus an updated recent list, sent to one recipient when an event happens. The options compared are Server-Sent Events (SSE), polling (React Router revalidation or fetcher polling), and WebSockets. They are judged on connection handling, deploy constraints and complexity for this stack.

## Short answer

**Use SSE from a React Router resource route, with an in-process `EventEmitter` doing the fan-out. Back it with a refetch of the recent-notifications list on focus and on reconnect.** It runs unchanged under the stock `react-router-serve` and under `react-router dev`. The React Router Express adapter already flushes headers for `text/event-stream` and flushes after every chunk, so the `compression` middleware does not hold events back. It also aborts `request.signal` when the client disconnects, which gives us a clean place to unsubscribe. WebSockets would mean replacing `react-router-serve` with a custom server, and that buys nothing for a one-way stream. Plain polling is the simplest fallback. `useRevalidator` polling is a bad fit here, because it re-runs every active loader, including the heavy `layout.app.tsx` loader.

## Stack constraints (verified in repo)

| Fact | Evidence |
| --- | --- |
| Production server is `react-router-serve`: Express 5 plus `compression()`, `express.static`, `morgan`, then `app.all("/{*splat}", createRequestHandler(...))`, started with `app.listen`. It registers no `upgrade` handler. | `node_modules/@react-router/serve/dist/cli.js` lines 113–127; `package.json` `"start"` |
| The `compression` middleware is on for every non-RSC build. | `node_modules/@react-router/serve/dist/cli.js` line 113 (`if (!isRSCBuild) app.use(compression())`) |
| Dev server is `react-router dev` (Vite). Requests go through `@remix-run/node-fetch-server` `createRequest` / `sendResponse`. | `node_modules/@react-router/dev/dist/vite.js` lines 27, 37–41, 1659–1660 |
| `ssr: true`, framework mode. | `react-router.config.ts` |
| Database is SQLite through better-sqlite3, a single file opened by a single Node process. There is no Redis or pub/sub. | stack facts in the task brief; `app/db` |
| Auth is a cookie session (`cadence_session`, `httpOnly`, `sameSite: "lax"`), read with `getCurrentUserId(request)`. | `app/lib/session.ts` |
| The `<Toaster>` from `sonner` is rendered in the app layout. The layout loader runs several DB queries (users, recent courses with progress, team admin). | `app/routes/layout.app.tsx` lines 174–216, 247 |
| The current user can change mid-session through the DevUI panel (`api.switch-user.ts`). | `app/routes/api.switch-user.ts`, `app/components/dev-ui.tsx` |

## Option A: Server-Sent Events (EventSource)

### How it would work here

1. A resource route, for example `app/routes/api.notifications.stream.ts`, exports only a `loader`. React Router treats a module with a loader and no default export as a resource route, and it may return any `Response` ([RR resource routes](https://reactrouter.com/how-to/resource-routes)). The loader:
   - calls `getCurrentUserId(request)` and returns 401 if there is no user;
   - returns `new Response(readableStream, { headers: { "Content-Type": "text/event-stream", "Cache-Control": "no-cache" } })`. The `text/event-stream` MIME type is required ([MDN, Using SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)), and any other status or content type makes the browser fail the connection permanently ([WHATWG HTML §9.2](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-eventsource-interface));
   - subscribes to a module-level `EventEmitter` on a per-user channel such as `notification:<userId>`, and writes `id: <notificationId>\nevent: notification\ndata: <json>\n\n` for each event. The `event`, `data`, `id` and `retry` fields are defined in [MDN, Using SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events);
   - sends a comment line (`: ping\n\n`) about every 15 s, which the spec recommends for proxies that drop idle connections ([WHATWG HTML §9.2.7 authoring notes](https://html.spec.whatwg.org/multipage/server-sent-events.html#authoring-notes));
   - on `request.signal` `abort`, removes the listener, clears the ping interval and closes the stream.
2. Whatever code creates a Notification row (an action) calls `emitter.emit("notification:<recipientId>", row)` after the insert.
3. On the client, a small component in `layout.app.tsx` opens `new EventSource("/api/notifications/stream")` in a `useEffect` keyed on `currentUser.id`, calls `toast(...)` on each `notification` event, and closes the source on cleanup. The request is same-origin, so the session cookie goes with it. `withCredentials` only governs *cross-origin* (CORS) credentials and defaults to `false` ([MDN EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)). Keying the effect on the user id matters because a DevUI user switch changes the session, while an open stream stays bound to the user it authenticated as.

### What the installed code does with an SSE response (confirmed in source)

- **Headers are flushed immediately.** `sendRemixResponse` calls `res.flushHeaders()` when the `Content-Type` matches `text/event-stream`. Source: `node_modules/@react-router/express/dist/index.js` line 63.
- **Each chunk is flushed through compression.** `writeReadableStreamToWritable` calls `writable.flush()` after every `write` when a `flush` method exists. Source: `node_modules/@react-router/node/dist/index.js` lines 107–122. The `compression` middleware adds `res.flush()`, which flushes its zlib stream. Source: `node_modules/compression/index.js` lines 92–97.
  - Without that flush, events would be held back. `compressible("text/event-stream")` returns `true` (checked locally), so `compression` *will* gzip an SSE response under the default filter. Source: `node_modules/compression/index.js` lines 299–307.
  - To skip compression entirely, send `Cache-Control: no-cache, no-transform`. `compression` bypasses responses whose `Cache-Control` contains `no-transform`. Source: `node_modules/compression/index.js` lines 318–321 and 182–185.
- **Client disconnect aborts `request.signal`.**
  - In production: `createRemixRequest` wires `res.on("close", () => controller?.abort())` into the `Request`'s `signal`. Source: `node_modules/@react-router/express/dist/index.js` lines 41–48.
  - In dev, the same happens through `@remix-run/node-fetch-server`. Source: `node_modules/@remix-run/node-fetch-server/dist/lib/request-listener.js` line 175.
  - This abort is the unsubscribe hook.
- **Compression cleans up too.** It frees its zlib stream on response `close`. Source: `node_modules/compression/index.js` lines 164–172.

### Reconnection and missed events

- **Automatic reconnect.** `EventSource` reconnects by itself after network errors. The `retry:` field sets the reconnection delay, and the initial delay is implementation-defined, "probably in the region of a few seconds" ([WHATWG HTML §9.2.2, §9.2.6](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-eventsource-interface)).
- **Resuming from the last event.** On reconnect the browser sends a `Last-Event-ID` header with the last `id:` it saw ([WHATWG HTML §9.2.4](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-last-event-id-header)). Since notifications are DB rows with ids, the loader can replay `WHERE id > Last-Event-ID AND recipient = me` before it subscribes.
- **When the browser stops for good.** A non-200 status or a wrong content type fails the connection with no reconnect, and a 204 tells the client to stop reconnecting ([WHATWG HTML §9.2](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events-intro)). A 401 for a logged-out user therefore ends the stream cleanly. The client must re-create the `EventSource` after login or a user switch.

### Connection limits and multiple tabs

- **HTTP/1.1.** SSE connections are capped at **6 per browser and domain, across all tabs**, and Chrome and Firefox have marked this "Won't fix" ([MDN EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)). `react-router-serve` uses `app.listen`, which serves HTTP/1.1. If a user opens 6 Cadence tabs on HTTP/1.1, the streams take every connection slot for that origin, and further page loads or fetches stall.
- **HTTP/2.** The limit is the negotiated number of concurrent streams, which defaults to 100 ([MDN EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)). A TLS reverse proxy that speaks HTTP/2 to the browser (most PaaS edges do) removes the problem in practice.
- **Mitigations.**
  - Close the `EventSource` on `visibilitychange` to `hidden`, reopen it on `visible`, and refetch the list when the tab comes back. The `visibilitychange` event fires when the user switches tabs or minimises the browser ([MDN visibilitychange](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event)).
  - Alternatively, share one connection across tabs (BroadcastChannel / SharedWorker). That is more complex and not needed at first.

### Pros / cons

- **Pros**
  - Push arrives within a few milliseconds of the event.
  - The browser API is built in and reconnects automatically, and `Last-Event-ID` covers resuming.
  - It works with the existing cookie auth and the existing resource-route pattern.
  - It needs **no server changes**, because the adapter already handles flushing and abort.
- **Cons**
  - Each tab holds one long-lived connection, so the HTTP/1.1 limit of 6 applies.
  - Fan-out through an in-process `EventEmitter` works only while there is one Node process.
  - `EventSource` only does GET, and cookies are the only auth it can carry.
  - The stream is one-way only ([MDN, Using SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)), which is fine for notifications.
  - Node's `EventEmitter` warns ("possible EventEmitter memory leak") past 10 listeners for one event name ([Node events](https://nodejs.org/api/events.html#eventsdefaultmaxlisteners)). Per-user event names keep counts low. A single shared event name needs `setMaxListeners`.

## Option B: Polling

### B1. `useRevalidator` on an interval / on focus

- **What the API is for.** `useRevalidator().revalidate()` re-runs the page's loaders "for reasons outside of normal data mutations like `Window` focus or polling on an interval" ([RR useRevalidator](https://reactrouter.com/api/hooks/useRevalidator)).
- **Why it is costly here.** It revalidates every matched route loader, including `layout.app.tsx`'s loader (all users, recent courses with per-course progress counts, team admin check). Every tick would redo that work for every open tab.
- **`shouldRevalidate` does not solve it.** It can opt a route's loader out, but it "does not skip parent or sibling loaders" ([RR route module, shouldRevalidate](https://reactrouter.com/start/framework/route-module#shouldrevalidate)). Tuning it per route to make polling cheap is fragile.
- **Making toasts from it.** We would have to diff the previous and new loader data to decide what to toast.

### B2. `useFetcher().load()` polling a small resource route

- **How it works.** `fetcher.load("/api/notifications?since=<id>")` on a `setInterval` (for example every 15–30 s while visible) plus on focus, using `visibilitychange` ([MDN visibilitychange](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event)). `fetcher.data` holds the loader result ([RR useFetcher](https://reactrouter.com/api/hooks/useFetcher)). A plain `fetch()` works just as well.
- **Cost.** It touches only the notifications query, with no long-lived connections, no fan-out and no single-instance assumption. The DB is the source of truth, so it works on any number of instances.
- **Cons.** Latency equals the polling interval, and most requests return nothing. Load grows as tabs × 1/interval.

### Pros / cons (polling overall)

- **Pros**
  - Simplest to build.
  - Stateless, so it scales horizontally without any change.
  - Not affected by the 6-connection limit.
- **Cons**
  - Delivery is delayed by the interval.
  - Wasted requests.
  - Revalidator-based polling in particular re-runs heavy layout loaders.

## Option C: WebSockets

- **What it gives.** Two-way, interactive sessions ([MDN WebSockets API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)). Notifications only need server-to-client delivery, so the second direction goes unused.
- **The stock server can't accept it.** The Node HTTP server ignores upgrade requests unless something listens for the `'upgrade'` event: "By default all HTTP upgrade requests are ignored … unless you listen to this event" ([Node http 'upgrade'](https://nodejs.org/api/http.html#event-upgrade_1)). `react-router-serve` never registers one and never exposes its `server` object. Source: `node_modules/@react-router/serve/dist/cli.js` lines 122–127. The `ws` package is not installed.
- **What switching would take.**
  - Moving from `@react-router/serve` to a custom Express server built on `@react-router/express` `createRequestHandler`. React Router documents this ("Migrating from the React Router App Server") with an [Express template](https://github.com/remix-run/react-router-templates/tree/main/node-custom-server) ([RR adapter docs](https://reactrouter.com/api/other-api/adapter)).
  - Attaching a WebSocket server to the `upgrade` event.
  - Parsing the session cookie by hand during the upgrade.
  - Building our own reconnect/backoff and heartbeat, since there is no `Last-Event-ID` equivalent.
  - Keeping dev (Vite) working, since Vite already uses WebSockets for HMR.
- **No backpressure.** The browser `WebSocket` interface does not support it ([MDN WebSockets API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)). That is irrelevant at notification volumes.
- **Same scaling limit as SSE.** Single-instance fan-out applies here too.

**Pros:** two-way, and not counted against the HTTP/1.1 per-origin request pool the way SSE is. **Cons:** needs a custom server, the most code, auth and reconnect done by hand, and no benefit for one-way notifications.

## Comparison

| | SSE (resource route) | Fetcher / fetch polling | `useRevalidator` polling | WebSockets |
| --- | --- | --- | --- | --- |
| Latency | Near-instant | ≤ interval | ≤ interval | Near-instant |
| Works with stock `react-router-serve` | Yes: headers and chunks flushed, abort wired (`@react-router/express` L48, L63; `@react-router/node` L121) | Yes | Yes | **No**: needs a custom server plus an `upgrade` handler ([Node http](https://nodejs.org/api/http.html#event-upgrade_1)) |
| Works in `react-router dev` | Yes (`node-fetch-server` abort, L175) | Yes | Yes | Needs custom dev wiring |
| Auth | Session cookie, same-origin | Session cookie | Session cookie | Manual cookie parse on upgrade |
| Reconnect / catch-up | Built in, `retry` + `Last-Event-ID` ([WHATWG](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-last-event-id-header)) | N/A (stateless) | N/A | Hand-rolled |
| Multi-tab cost | 1 connection per tab; 6/origin on HTTP/1.1 ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)) | 1 request per tab per interval | Re-runs all matched loaders per tab per interval | 1 socket per tab |
| Multi-instance | Needs a shared pub/sub (in-process emitter is single-instance only) | Works as is | Works as is | Needs a shared pub/sub |
| Server load | Idle connections + 15 s pings | Mostly-empty queries | Heavy (layout loader) | Idle sockets + heartbeats |
| Complexity | Low–medium | Low | Low, but costly at runtime | High |

## Recommendation and caveats

1. **Primary path: SSE** from `api.notifications.stream.ts`, as described in Option A.
   - Fan-out goes through a module-level `EventEmitter` keyed `notification:<userId>`, with the listener removed on `request.signal` abort.
   - Send `id:` set to the notification row id, and replay from `Last-Event-ID` on reconnect.
   - Ping every ~15 s.
   - Set `Cache-Control: no-cache, no-transform` so `compression` skips the stream, even though flushing already makes compressed streams work.
2. **The recent-notifications list stays DB-backed.** Load it through a normal loader or fetcher, and refetch it on reconnect (`EventSource` `open`) and on `visibilitychange` → `visible`. SSE only triggers toasts and invalidation. Anything missed is recovered from the DB, never from the stream.
3. **Caveat: single instance only.** The in-process `EventEmitter` reaches only the subscribers held by the same Node process. That holds today, since better-sqlite3 on one file already implies one process. With more than one instance, the emitter must be replaced by a shared pub/sub, or the app must fall back to polling.
4. **Caveat: HTTP/1.1 tab limit.** Close the stream in hidden tabs, or make sure production is served over HTTP/2 at the edge ([MDN EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)).
5. **Fallback:** if SSE causes trouble (a proxy buffering it, or the connection limit), switch the client to `fetch`/`fetcher.load` polling of `/api/notifications?since=<id>` every 15–30 s plus on focus. The server side is the same query as the SSE replay. Avoid `useRevalidator` interval polling because of the layout loader cost.
6. **Not recommended: WebSockets.** They need a custom server and give no benefit for one-way notifications.

## Open questions

- **Where will production run, and does its edge speak HTTP/2?** If it does, the 6-connection issue is moot. Does the edge buffer `text/event-stream` responses, or time out idle connections? Some proxies need an `X-Accel-Buffering: no` style header. That is not verified here.
- **Will Cadence ever run more than one Node instance?** If yes, decide between a pub/sub layer and polling now.
- **Does Vite dev's HMR full reload close open `EventSource`s cleanly?** The module-level emitter is re-created on server module reload, so old listeners could leak in dev. To verify, use the `run` skill.
- **Should hidden tabs keep streaming?** The choice is lower latency for background toasts versus connection budget.
- **Should toasts be de-duplicated across tabs?** Every open tab would show the same toast. A BroadcastChannel leader-election pattern is possible but out of scope.

## Sources

- React Router, Resource Routes: https://reactrouter.com/how-to/resource-routes
- React Router, `useRevalidator`: https://reactrouter.com/api/hooks/useRevalidator
- React Router, Route Module `shouldRevalidate`: https://reactrouter.com/start/framework/route-module#shouldrevalidate
- React Router, `useFetcher`: https://reactrouter.com/api/hooks/useFetcher
- React Router, Server adapters / custom Express server: https://reactrouter.com/api/other-api/adapter
- React Router node-custom-server template: https://github.com/remix-run/react-router-templates/tree/main/node-custom-server
- MDN, EventSource (connection limit, withCredentials): https://developer.mozilla.org/en-US/docs/Web/API/EventSource
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- MDN, WebSockets API: https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
- MDN, `visibilitychange`: https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event
- WHATWG HTML, Server-sent events: https://html.spec.whatwg.org/multipage/server-sent-events.html
- Node.js, `http` `'upgrade'` event: https://nodejs.org/api/http.html#event-upgrade_1
- Node.js, `events.defaultMaxListeners`: https://nodejs.org/api/events.html#eventsdefaultmaxlisteners
- Installed source (react-router 8.3.0 family, compression 1.8.2):
  - `node_modules/@react-router/serve/dist/cli.js` (L113 compression, L122–127 handler and `app.listen`)
  - `node_modules/@react-router/express/dist/index.js` (L41–48 abort on close, L63 `flushHeaders` for event-stream)
  - `node_modules/@react-router/node/dist/index.js` (L107–122 flush after each write)
  - `node_modules/compression/index.js` (L92–97 `res.flush`, L164–172 cleanup on close, L299–321 filter and no-transform)
  - `node_modules/@remix-run/node-fetch-server/dist/lib/request-listener.js` (L175 abort on close, dev path)
  - `node_modules/@react-router/dev/dist/vite.js` (L27, L37–41, L1659–1660 dev request handling)
