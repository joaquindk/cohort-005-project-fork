# Web Push feasibility for Cadence

Research date: 2026-11-10. Scope: planning only. No app code was changed.

Question: can Cadence (React Router 8.3 framework mode, SSR, served by `react-router-serve` on one Node process, Vite 7, SQLite through Drizzle/better-sqlite3) send OS/browser push notifications through the standard Web Push API, and what would that take?

## Answer

**Yes.** Cadence can do it with no new infrastructure and no third-party account. The work is one static service worker file, a VAPID key pair in env vars, one new table, two small API routes, and the `web-push` npm library on the server. No build plugin is needed.

- **Service worker:** a hand-written `public/sw.js`. Vite copies `public/` to the site root unchanged, so it is served from `/sw.js` and can claim scope `/`. Register it from a client-only effect in `app/root.tsx`. `vite-plugin-pwa` is not required ([Vite](https://vite.dev/guide/assets), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)).
- **Keys:** generate one VAPID (P-256) key pair once. The public key goes to the browser as `applicationServerKey`. The private key stays on the server. Changing keys orphans every existing subscription, so treat them as long-lived secrets ([RFC 8292 §4–5](https://www.rfc-editor.org/rfc/rfc8292.html)).
- **Subscriptions:** each one belongs to a single browser on a single device, so a user can have many. Store them in a new `push_subscriptions` table keyed by a unique `endpoint`. Delete a row when the push service answers 404 or 410 ([web.dev](https://web.dev/articles/push-notifications-common-issues-and-reporting-bugs)).
- **Browser support:** Chrome, Edge and Firefox on desktop and Android; Safari 16+ on macOS Ventura and later; iOS/iPadOS 16.4+ **only for web apps added to the Home Screen**. iOS needs a web app manifest with `display: standalone`, and the repo has none today. That manifest is the one real new requirement ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/), [MDN BCD](https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/PushManager.json)).
- **Cost:** the push services cost nothing to use. Apple needs no developer account, and VAPID needs no Google or Firebase project. The ongoing work is keeping the keys safe, pruning dead subscriptions, and staying within about 4 KB per payload. Delivery is not guaranteed, so push can only add to in-app notifications, never replace them ([RFC 8030](https://www.rfc-editor.org/rfc/rfc8030.html), [RFC 8291](https://www.rfc-editor.org/rfc/rfc8291.html)).

---

## 1. Service worker registration under React Router / Vite

**Where the file lives**
- Vite serves anything in `public/` at `/` in dev and copies it "to the root of the dist directory as-is", without hashing. Reference these files with root-absolute paths ([Vite: The public directory](https://vite.dev/guide/assets)).
- React Router's build puts the browser assets in `build/client`. `react-router-serve` serves static files with `express.static` ([React Router: @react-router/serve](https://reactrouter.com/api/other-api/serve), [Quick Start](https://reactrouter.com/tutorials/quickstart)). So `public/sw.js` becomes `https://<host>/sw.js` in both dev and prod.
- Consequence: the worker is **not** bundled. It must be plain browser JavaScript and cannot `import` from `app/`. That is fine here, because the push handler is small. (Inferred from Vite's "copied as-is" rule.)

**Scope**
- By default a worker's scope is the directory that holds the script. A worker at `/sw.js` can therefore control `/`. Asking for a scope broader than the script's path fails unless the server sends a `Service-Worker-Allowed` header ([MDN: Using Service Workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)). Serving from the root avoids that header, which matters because `react-router-serve` cannot be configured to add headers ([React Router: serve](https://reactrouter.com/api/other-api/serve)).
- The worker must be same-origin with the page ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)).

**HTTPS and localhost**
- Service workers only run in secure contexts. `localhost` counts as secure for development ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)).
- The Push API interfaces are `[SecureContext]`. `subscribe()` rejects with `NotAllowedError` when the scope's scheme is not `https` ([W3C Push API](https://w3c.github.io/push-api/)). Production therefore needs TLS in front of the Node server. `react-router dev` on `http://localhost:5173` works for local testing.
- Registration can also fail when cookies are blocked or in private browsing ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)).

**Registering**
- Call `navigator.serviceWorker.register("/sw.js", { scope: "/" })` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)). With SSR, `navigator` does not exist on the server, so the call goes in a `useEffect` in `app/root.tsx`, behind a check for both `"serviceWorker" in navigator` and `"PushManager" in window` ([web.dev: Subscribing a user](https://web.dev/articles/push-notifications-subscribing-a-user)).
- Registering alone does not prompt the user. It is safe to do on every page load.

**Is vite-plugin-pwa needed?** No. Its value is generating precache manifests and offline caching. Push needs only a `push` and a `notificationclick` handler in a static file. Not adding it keeps the Vite config (`tailwindcss(), reactRouter(), tsconfigPaths()`) as it is. This is a judgement call, not a sourced claim.

Illustrative worker (not an implementation):

```js
// public/sw.js
self.addEventListener("push", (event) => {
  const data = event.data?.json() ?? {};
  event.waitUntil(
    self.registration.showNotification(data.title ?? "Cadence", {
      body: data.body, tag: data.tag, data: { url: data.url },
    })
  );
});
self.addEventListener("notificationclick", (event) => {
  event.notification.close();
  event.waitUntil(clients.openWindow(event.notification.data?.url ?? "/"));
});
```

`showNotification` options (`body`, `icon`, `badge`, `tag`, `data`, `renotify`, `requireInteraction`, `actions`) come from [MDN: showNotification](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/showNotification). The `clients.openWindow` pattern is common practice, but the MDN page fetched for this note does not document it (**unverified here**).

## 2. VAPID keys

- **What they are:** an ECDSA P-256 key pair. The server signs a JWT with the private key (ES256) and sends it as `Authorization: vapid t=<jwt>, k=<publicKey>`. The `exp` claim must be no more than 24 hours ahead. `aud` is the origin of the push service ([RFC 8292 §2–3](https://www.rfc-editor.org/rfc/rfc8292.html)). `web-push` does all of this for you.
- **Generating them:** run `npx web-push generate-vapid-keys [--json]` or call `webpush.generateVAPIDKeys()`. Both return URL-safe base64 `publicKey` and `privateKey`. Generate once, store, and reuse ([web-push README](https://github.com/web-push-libs/web-push)).
- **Public key to the client:** pass it as `applicationServerKey`, either as a base64url string or as an `ArrayBuffer`/`Uint8Array`. It must be a 65-octet uncompressed P-256 point ([W3C Push API](https://w3c.github.io/push-api/), [MDN: subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)). Chrome and Edge require `applicationServerKey` ([MDN BCD](https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/PushManager.json)). In this repo it can come from a root `loader` that reads `process.env.VAPID_PUBLIC_KEY`. That matches how `courses.$slug.welcome.tsx` reads `process.env.DISCORD_INVITE_URL` today. Exposing the public key is safe.
- **Private key:** server only. Read it only in `*.server.ts` modules so it never reaches the client bundle.
- **`subject`:** `setVapidDetails(subject, publicKey, privateKey)` requires an `https:` or `mailto:` URI ([web-push README](https://github.com/web-push-libs/web-push)). RFC 8292 says `sub` SHOULD be a contact URI ([RFC 8292 §2.1](https://www.rfc-editor.org/rfc/rfc8292.html)). The README warns that Apple may reject localhost-style subjects with `403 BadJwtToken`, so use a real address even in dev.
- **Rotation:** a subscription is bound to the key it was created with. To change keys, the browser has to create a new subscription restricted to the new key, and the server must remember which key each subscription used ([RFC 8292 §4–5](https://www.rfc-editor.org/rfc/rfc8292.html)). In practice, rotating means re-subscribing every device. Do not rotate casually. If rotation ever has to be possible, store a key identifier per row.
- **Suggested env vars:** `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT`.

## 3. Subscription lifecycle

**Permission and the user gesture**
- Ask from a click handler ("Enable notifications"), never on page load. MDN calls this best practice, and Firefox 72+ (Android 79+) only allows `subscribe()` from a user gesture ([MDN: subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe), [MDN BCD](https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/PushManager.json)).
- Safari on macOS and on iOS requires an explicit user gesture ([WebKit: Meet Web Push](https://webkit.org/blog/12945/meet-web-push/), [WebKit iOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)).
- `Notification.requestPermission()` resolves to `granted`, `denied` or `default`; treat `default` as `denied` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static)). A denial is hard for users to undo, so explain the value before asking ([web.dev](https://web.dev/articles/push-notifications-subscribing-a-user)).

**Subscribing**
- Call `registration.pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)).
- `userVisibleOnly: true` is mandatory in practice. Chrome and Edge reject without it ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe), [web.dev](https://web.dev/articles/push-notifications-subscribing-a-user)). Safari requires it and revokes the subscription if a push does not show a notification ([WebKit](https://webkit.org/blog/12945/meet-web-push/)). **Every push must display a notification. There is no silent push.**

**The subscription JSON:** `subscription.toJSON()` gives `{ endpoint, expirationTime, keys: { p256dh, auth } }`, with both keys base64url and unpadded ([W3C Push API](https://w3c.github.io/push-api/)).
- `p256dh` is the browser's P-256 ECDH public key. `auth` is a 16-octet secret. Both are used to encrypt the payload ([RFC 8291](https://www.rfc-editor.org/rfc/rfc8291.html)).
- `expirationTime` is ms since the epoch, or `null` ([W3C](https://w3c.github.io/push-api/)). It is usually `null`, but browsers can still expire subscriptions that go unused ([web.dev](https://web.dev/articles/push-notifications-subscribing-a-user)).
- POST the JSON to the server and save it ([web.dev: Sending messages](https://web.dev/articles/sending-messages-with-web-push-libraries)).

**Multiple devices:** subscriptions are per browser and per device, so users subscribe on each one ([web.dev](https://web.dev/articles/push-notifications-subscribing-a-user)). The data model is one user to many subscriptions.

**`pushsubscriptionchange`**
- The spec fires this event with `oldSubscription` and `newSubscription` when the browser refreshes a subscription. `newSubscription` is `null` when it cannot refresh, or possibly when permission is revoked ([W3C](https://w3c.github.io/push-api/)).
- Support is uneven:
  - Chrome added it in 138.
  - Firefox supports it only partly, without `oldSubscription` and `newSubscription` (bug 1497429).
  - Safari on macOS has it from 16.
  - **Safari on iOS does not support it.**
  - Sources: [MDN BCD](https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/ServiceWorkerGlobalScope.json), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerGlobalScope/pushsubscriptionchange_event).
- So this event cannot be the only refresh path. Also re-check the subscription on each app load: call `pushManager.getSubscription()` and upsert it to the server. That upsert is idempotent because of the unique `endpoint`.

**Pruning**
- The push service MUST return 404 when you send to an expired subscription ([RFC 8030 §7.3](https://www.rfc-editor.org/rfc/rfc8030.html)).
- web.dev says to delete the stored subscription on 404 or 410. An `unsubscribe()` reproduces a 410 ([web.dev: common issues](https://web.dev/articles/push-notifications-common-issues-and-reporting-bugs), [web.dev: sending](https://web.dev/articles/sending-messages-with-web-push-libraries)).
- Other codes:
  - 429: back off using `Retry-After`.
  - 400: malformed request.
  - 413: payload too large.
- Unsubscribing from the UI should call `subscription.unsubscribe()` on the client and delete the row on the server.

**Proposed table**, matching `app/db/schema.ts` conventions: integer autoincrement `id`, snake_case column names, ISO-string `text` timestamps via `$defaultFn`, `.references(() => users.id)` without cascade like the other tables, and indexes declared in the third argument.

```ts
export const pushSubscriptions = sqliteTable(
  "push_subscriptions",
  {
    id: integer("id").primaryKey({ autoIncrement: true }),
    userId: integer("user_id").notNull().references(() => users.id),
    // Push service URL. Unique per browser+device; the upsert key.
    endpoint: text("endpoint").notNull().unique(),
    p256dh: text("p256dh").notNull(),
    auth: text("auth").notNull(),
    // Informational only, for a "your devices" list.
    userAgent: text("user_agent"),
    // ms epoch from the browser, usually null.
    expirationTime: integer("expiration_time"),
    createdAt: text("created_at").notNull().$defaultFn(() => new Date().toISOString()),
    lastSuccessAt: text("last_success_at"),
  },
  (table) => [index("push_subscriptions_user_idx").on(table.userId)]
);
```

Design notes:
- When an endpoint is re-registered by a different signed-in user (a shared device), reassign its `userId` on upsert.
- `lastSuccessAt` and `expirationTime` are optional extras. The required minimum is `userId`, `endpoint`, `p256dh`, `auth`, `createdAt`.
- Schema changes go through the repo's `database-migrations` skill (generate migration, update seed, reseed).

## 4. Browser and OS support

Versions where `PushManager.subscribe` and the `push` event shipped ([MDN BCD PushManager](https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/PushManager.json), [MDN BCD ServiceWorkerGlobalScope](https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/ServiceWorkerGlobalScope.json)):

| Platform | Since | Notes |
|---|---|---|
| Chrome desktop and Android | 42 (push event 40) | needs `applicationServerKey` and `userVisibleOnly: true` |
| Edge | 17 | as Chrome |
| Firefox desktop / Android | 44 / 48 | user gesture required from 72 / 79 |
| Samsung Internet, Opera | follow Chrome | |
| Safari macOS | 16 (macOS Ventura+) | standard Web Push, no Apple developer account ([WebKit](https://webkit.org/blog/12945/meet-web-push/)) |
| Safari iOS/iPadOS | 16.4 | **Home Screen web apps only** |

**iOS/iPadOS requirements** ([WebKit blog 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)):
- The site needs a web app manifest with `display: "standalone"` or `"fullscreen"`. Without one, Add to Home Screen saves a plain bookmark.
- The user installs the app through Share, then Add to Home Screen.
- Permission can only be requested from a direct user action inside the installed app.
- No Apple Developer account is needed.
- If you allowlist endpoints, allow `*.push.apple.com`.
- `apple-touch-icon` takes precedence over manifest icons. The manifest `id` identifies the app.
- The Badging API (`setAppBadge`) is available.

**Repo check:** `public/` contains only `favicon.svg` and `images/` (`course-nodejs.svg`, `course-typescript.svg`). Nothing in `app/` or `public/` mentions a manifest, so **iOS push is impossible until one is added**. A manifest is linked with `<link rel="manifest" href="/manifest.webmanifest">`; the registered type is `application/manifest+json` ([MDN: Web app manifest](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest)). Whether `express.static` serves `.webmanifest` with that exact MIME type was not verified. Browsers generally accept `.json` too ([MDN](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest)).

**Desktop delivery caveat:** on desktop, a push only arrives while the browser process is running (windows may be closed). Android wakes the browser ([web.dev: Push FAQ](https://web.dev/articles/push-notifications-faq)).

**Later Safari change: Declarative Web Push**
- A push whose JSON body includes `"web_push": 8030`, a `notification` object (non-empty `title`, required `navigate` URL) and an optional `app_badge` is shown without running service worker JavaScript ([WebKit: Meet Declarative Web Push](https://webkit.org/blog/16535/meet-declarative-web-push/)).
- It shipped on iOS/iPadOS 18.4 for Home Screen web apps ([WebKit: Safari 18.4](https://webkit.org/blog/16574/webkit-features-in-safari-18-4/)) and on macOS in Safari 18.5 ([WebKit: Safari 18.5](https://webkit.org/blog/16923/webkit-features-in-safari-18-5/)).
- It is backwards compatible: older browsers handle the same JSON through the service worker `push` handler. Safari also exposes `window.pushManager`, so a subscription works without a service worker there ([WebKit](https://webkit.org/blog/16535/meet-declarative-web-push/)).
- **Recommendation:** shape payloads in the declarative format from day one (`{ web_push: 8030, notification: { title, body, navigate, tag } }`) and have `sw.js` read that shape. One payload then works everywhere.
- Whether Chromium or Firefox have shipped declarative handling was **not verified** in this research. Assume they need the service worker.

## 5. Sending library: `web-push`

- **Package:** `web-push` 4.0.0 on npm, engines `node >= 16`. Its dependencies are `jws`, `asn1.js`, `http_ece`, `https-proxy-agent` (npm registry metadata, checked 2026-11-10). The local Node is v26, which is compatible.
- **API** ([README](https://github.com/web-push-libs/web-push)):
  - `webpush.setVapidDetails(subject, publicKey, privateKey)` sets the VAPID details once at startup.
  - `webpush.sendNotification(subscription, payload?, options?)`: the payload is a string or Buffer, and encrypting it needs `keys.p256dh` and `keys.auth`. It returns a promise that exposes `statusCode`, `headers` and `body` on both success and failure.
- **Options** ([README](https://github.com/web-push-libs/web-push)):
  - `TTL`: seconds; defaults to four weeks.
  - `urgency`: `very-low`, `low`, `normal` (default) or `high`.
  - `topic`: at most 32 URL-safe base64 characters. Coalesces pending messages.
  - `contentEncoding`: defaults to `aes128gcm`; `aesgcm` is legacy.
  - Also: `headers`, `timeout` (socket timeout in ms), `proxy`, `agent`, per-call `vapidDetails`.
- **What the protocol says** ([RFC 8030](https://www.rfc-editor.org/rfc/rfc8030.html)):
  - TTL is mandatory; a request without it gets a 400. `TTL: 0` delivers only if the device is reachable right now.
  - The push service never forwards Urgency or Topic to the device.
  - A new message with the same Topic replaces an undelivered one.
- **Encryption:** RFC 8291 makes `aes128gcm` the only content encoding for Web Push ([RFC 8291](https://www.rfc-editor.org/rfc/rfc8291.html)).
- **Errors:** the README does not name an error class. Use the rejected object's `statusCode`:
  - 404 or 410: delete the row.
  - 429: honour `Retry-After`.
  - Anything else: log it ([web.dev](https://web.dev/articles/push-notifications-common-issues-and-reporting-bugs)).
- **Alternatives:** you could implement RFC 8291 and RFC 8292 yourself with WebCrypto. Not recommended, since web.dev calls the encryption and headers "fiddly" ([web.dev](https://web.dev/articles/sending-messages-with-web-push-libraries)). The Firebase Admin SDK is an option only if you want FCM-specific features. Neither is needed.

## 6. Running cost and operations

- **Cost: nothing to pay, no accounts.**
  - Apple: no Apple Developer Program membership needed ([WebKit](https://webkit.org/blog/12945/meet-web-push/)).
  - Google: VAPID means "Chrome's support for it doesn't require a Google or Firebase project" ([web.dev FAQ](https://web.dev/articles/push-notifications-faq)). FCM is listed as "No-cost" ([Firebase pricing](https://firebase.google.com/pricing)).
  - Mozilla autopush: that it is free with no account is **unverified from a primary source**. It is implied by the account-less VAPID model, but the autopush docs fetched here do not say so.
- **Ops: keys.** Keep `VAPID_PRIVATE_KEY` secret and back it up. Losing or rotating it means every device has to re-subscribe ([RFC 8292 §4–5](https://www.rfc-editor.org/rfc/rfc8292.html)).
- **Ops: pruning.** Delete rows on 404 or 410 during each send. Optionally sweep rows with no recent `lastSuccessAt`.
- **Ops: payload size.** Push services must accept bodies of at least 4096 octets. After `aes128gcm` overhead (86-octet header, 1 octet of padding, 16-octet tag) the safe plaintext limit is **3993 octets** ([RFC 8030](https://www.rfc-editor.org/rfc/rfc8030.html), [RFC 8291](https://www.rfc-editor.org/rfc/rfc8291.html)). Send a title, a short body and a URL, and fetch details when the user clicks.
- **Ops: TTL.** The `web-push` default of four weeks is too long for things like "live session starting". Set TTL per notification type. The push service may also keep a message for less time than you ask ([RFC 8030](https://www.rfc-editor.org/rfc/rfc8030.html)).
- **Delivery is not guaranteed.**
  - A 201 means the message was accepted, not delivered. The push service may stop retrying or drop the message ([RFC 8030 §5, §6.2](https://www.rfc-editor.org/rfc/rfc8030.html)).
  - Desktop needs the browser running ([web.dev FAQ](https://web.dev/articles/push-notifications-faq)).
  - The user can revoke permission at any time.
- **Inline send vs a queue on one Node process.**
  - Each send is one outbound HTTPS request per device. For a few devices per event, it is acceptable to send inline after the DB write commits, without awaiting it in the request path (fire-and-forget with error logging and pruning).
  - For fan-out (for example, notifying every enrolled student) a simple durable queue is safer: a `notification_outbox`-style SQLite table drained by an in-process interval. It survives restarts and avoids blocking requests.
  - `react-router-serve` in dev purges the require cache on every request, while module side effects like `setInterval` persist ([React Router: serve](https://reactrouter.com/api/other-api/serve)). A worker loop would need a singleton guard. This is a design recommendation, not a sourced fact.
- **Relationship to in-app delivery.** Push must be additive. The canonical record is the in-app notification row or feed. Push is one delivery channel that points back to it with a `navigate` URL. Send only to users who opted in, and still create the in-app item when push fails or the user has no subscription. (Cadence has no in-app notification model today; no matches for "notification" in `app/`.)

## Fit for this repo

What the repo looks like today:
- `vite.config.ts` uses plugins `tailwindcss()`, `reactRouter()`, `tsconfigPaths()`.
- `react-router.config.ts` sets `ssr: true`.
- `app/root.tsx` has `links` (favicon, fonts), a `Layout` with an inline dark-mode script, and no `loader` or effects.
- `public/` holds only `favicon.svg` and `images/`.
- Auth is a cookie session via `getCurrentUserId(request)` in `app/lib/session.ts`.
- JSON API routes follow `app/routes/api.video-tracking.ts`: a zod schema, then `parseJsonBody`, a 401 when there is no user, then a Drizzle insert.

Files to add:
- `public/sw.js`: `push` and `notificationclick` handlers in plain JS, reading the declarative payload shape.
- `public/manifest.webmanifest`: `name`, `short_name`, `id`, `start_url: "/"`, `display: "standalone"`, icons. This is required for iOS.
- PNG app icons (192 and 512) under `public/`, plus an `apple-touch-icon`. Today there is only an SVG favicon. Whether iOS accepts SVG manifest icons was **not verified**.
- `app/lib/push.server.ts`: `setVapidDetails` from env, `sendToUser(userId, payload)` that loops over the user's subscriptions and prunes on 404/410.
- `app/routes/api.push-subscriptions.ts`: POST upserts by `endpoint` for the current user, DELETE removes one. Register it in `app/routes.ts` next to `api/video-tracking`.
- A Drizzle migration in `drizzle/` (next after `0009_*`) and a seed update, using the `database-migrations` skill.
- An "Enable notifications" control, probably in `app/routes/settings.tsx`. It must use a click handler, feature detection, and handle the `denied` state. On iOS outside a Home Screen app it should show "Add to Home Screen first" instead of the button.

Files to change:
- `app/root.tsx`: add `{ rel: "manifest", href: "/manifest.webmanifest" }` (and `apple-touch-icon`) to `links`. Add a client-only `useEffect` that registers `/sw.js` and re-syncs `getSubscription()`. Expose `VAPID_PUBLIC_KEY` through a loader, either a root loader or the settings route loader.
- `app/db/schema.ts`: add the `pushSubscriptions` table (section 3).
- `package.json`: add the `web-push` dependency (and `@types/web-push` if needed; **unverified** whether 4.x ships its own types).
- Env: `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` (real `mailto:` or `https:`). There is no `.env.example` in the repo today.

Not needed:
- `vite-plugin-pwa`.
- Changes to `vite.config.ts` or `react-router.config.ts`.
- A custom Express server, since serving from the root avoids the need for a `Service-Worker-Allowed` header.

Tests: subscription upsert/delete and the prune-on-410 logic can be unit-tested with `vitest` by mocking `web-push`. The service worker and browser permission flow need manual testing: Chrome on localhost, plus an HTTPS deploy for iOS.

## Open questions for the PRD

1. Which events justify a push? (New lesson in an enrolled course, an instructor reply to a comment, a refund decision, a team invite.) Each needs its own TTL, urgency and topic.
2. Is there, or will there be, an in-app notification model that push points back to? Push should not be the source of truth.
3. Per-type opt-in preferences, or one global toggle? Do preferences live on the user or on each device?
4. Is iOS a must-have? If yes, the PWA manifest, icons and an "Add to Home Screen" onboarding are in scope.
5. Where is production TLS terminated? The domain must be HTTPS. Push will not work on a plain-HTTP preview.
6. Who owns the VAPID keys and the contact `subject`, and where are they stored in each environment? Do dev and prod use separate key pairs? They should, since subscriptions are bound to keys.
7. Should a large fan-out (a course-wide announcement) go through a persisted outbox, or is inline sending acceptable at current scale?
8. Should a shared device's subscription move to the newly signed-in user, and should logout delete the device's subscription?
9. Should payloads use the Declarative Web Push shape from the start (recommended)?

## Sources

- MDN, PushManager.subscribe: https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe
- MDN, Using Service Workers: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers
- MDN, pushsubscriptionchange event: https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerGlobalScope/pushsubscriptionchange_event
- MDN, Notification.requestPermission: https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static
- MDN, ServiceWorkerRegistration.showNotification: https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/showNotification
- MDN, Web app manifest: https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest
- MDN browser-compat-data, PushManager: https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/PushManager.json
- MDN browser-compat-data, ServiceWorkerGlobalScope: https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/ServiceWorkerGlobalScope.json
- W3C Push API: https://w3c.github.io/push-api/
- RFC 8030, Generic Event Delivery Using HTTP Push: https://www.rfc-editor.org/rfc/rfc8030.html
- RFC 8291, Message Encryption for Web Push: https://www.rfc-editor.org/rfc/rfc8291.html
- RFC 8292, VAPID: https://www.rfc-editor.org/rfc/rfc8292.html
- web.dev, Subscribing a user: https://web.dev/articles/push-notifications-subscribing-a-user
- web.dev, Sending messages with web push libraries: https://web.dev/articles/sending-messages-with-web-push-libraries
- web.dev, Common issues and reporting bugs: https://web.dev/articles/push-notifications-common-issues-and-reporting-bugs
- web.dev, Push notifications FAQ: https://web.dev/articles/push-notifications-faq
- WebKit, Meet Web Push (Safari 16): https://webkit.org/blog/12945/meet-web-push/
- WebKit, Web Push for Web Apps on iOS and iPadOS: https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
- WebKit, Meet Declarative Web Push: https://webkit.org/blog/16535/meet-declarative-web-push/
- WebKit, Features in Safari 18.4: https://webkit.org/blog/16574/webkit-features-in-safari-18-4/
- WebKit, Features in Safari 18.5: https://webkit.org/blog/16923/webkit-features-in-safari-18-5/
- Vite, Static asset handling / public directory: https://vite.dev/guide/assets
- React Router, @react-router/serve: https://reactrouter.com/api/other-api/serve
- React Router, Quick Start: https://reactrouter.com/tutorials/quickstart
- web-push README: https://github.com/web-push-libs/web-push
- web-push on npm (version and engines via `npm view web-push`): https://www.npmjs.com/package/web-push
- Firebase pricing (FCM no-cost): https://firebase.google.com/pricing
