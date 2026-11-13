# What would hand-rolled Web Push take?

Research for issue #8. All sources checked **2026-11-13** unless noted. Claims are cited inline; the source list is at the bottom.

## Question

What does hand-rolling browser Web Push involve for Cadence (React Router 8.3 framework mode, Vite 7, served by `react-router-serve`, SQLite/Drizzle, no service worker today)? Covers VAPID keys, the `web-push` npm library, service worker registration, permission-prompt rules, storing `PushSubscription`s, expired/410 handling, and browser support — especially iOS/iPadOS.

## TL;DR

- Web Push is three standards working together: the Push API, the Notifications API and Service Workers. The server side talks to the browser vendor's push service over HTTP using RFC 8030, with VAPID authentication (RFC 8292) and payload encryption ([WebKit: Meet Web Push][webkit-meet]; [RFC 8030][rfc8030]; [RFC 8292][rfc8292]).
- `web-push` (Node) handles VAPID signing and payload encryption. v4.0.0 shipped this month, is now ESM-only and ships no bundled TypeScript types ([npm registry][npm-wp]; [GitHub release v4.0.0][wp-release]).
- Cadence's existing stack can serve a plain `public/sw.js` with no new build tooling: Vite copies `public/` as-is ([Vite docs][vite-assets]), and `react-router-serve` serves the build's client directory at the site root (`node_modules/@react-router/serve/dist/cli.js`).
- All engines expect, and Firefox and Safari enforce, that the permission prompt comes from a user gesture. Chrome strongly discourages prompting on page load and now shows non-blocking prompts that can time out ([Mozilla][moz-gesture]; [Apple][apple-webpush]; [Chrome][chrome-android-prompts]).
- iOS/iPadOS: push only works for web apps added to the Home Screen, from iOS 16.4 ([WebKit][webkit-ios]; [MDN BCD][bcd]). Since iOS 26, any site added to the Home Screen opens as a web app by default, so a manifest is no longer strictly required. Before iOS 26, a manifest with `display: standalone` or `fullscreen` (or the legacy meta tag) was required ([WebKit: Safari 26.0][webkit-26]).
- Expired subscriptions show up as 404 or 410 from the push service (Apple and Mozilla both document 410). iOS Safari does not fire `pushsubscriptionchange`, so on iOS cleaning up on the server after a failed send is the only signal you get ([Apple][apple-webpush]; [Mozilla autopush][autopush]; [MDN BCD][bcd]).

## 1. How the pieces fit

1. The page registers a service worker, then calls `registration.pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })`. The result is a `PushSubscription` containing an `endpoint` URL plus `keys.p256dh` and `keys.auth` ([W3C Push API][push-api]; [web-push README][wp-readme]).
2. The page sends that subscription JSON to our server, which stores it against the user ([Apple][apple-webpush]).
3. To notify, the server POSTs an encrypted payload to `endpoint` with `TTL`, `Authorization: vapid …` and `Content-Encoding` headers ([RFC 8030 §5][rfc8030]; [RFC 8292][rfc8292]; [Apple][apple-webpush]).
4. The push service wakes the browser. The service worker gets a `push` event and **must** show a notification via `registration.showNotification()`. Safari revokes permission if it doesn't ([WebKit: Meet Web Push][webkit-meet]; [Apple][apple-webpush]).
5. A click on the notification fires `notificationclick` in the service worker, which opens or focuses a tab.

The push service is the browser vendor's (FCM for Chrome, Mozilla autopush for Firefox, `*.push.apple.com` for Safari). Using it costs nothing. Apple says no Apple Developer Program membership is needed ([WebKit][webkit-ios]; [Apple][apple-webpush]).

## 2. VAPID keys

- VAPID is an ECDSA P-256 key pair. The server signs a JWT using `ES256` with an `aud` claim (the push service's origin), an `exp` claim no more than 24 hours ahead, and an optional `sub` contact URI (`mailto:` or `https:`) ([RFC 8292 §2][rfc8292]).
- The browser receives the public key as `applicationServerKey`: an uncompressed P-256 point (65 bytes, starting `0x04`), base64url-encoded as a string or passed as a `BufferSource` ([W3C Push API][push-api]). The push service rejects messages not signed by the matching private key ([W3C Push API][push-api]).
- Generate the key pair **once** and keep it. Rotating it invalidates every existing subscription, because subscriptions are bound to the key ([web-push README][wp-readme]; Apple's `VapidPkHashMismatch` error, [Apple][apple-webpush]). `npx web-push generate-vapid-keys --json` generates one ([web-push README][wp-readme]).
- Apple-specific constraints: don't refresh the JWT more than once an hour ([Apple][apple-webpush]). A `sub` of `mailto:…@localhost` or an `https://localhost` URI gets `403 BadJwtToken` ([web-push README][wp-readme]).
- Practical upshot for Cadence: two env vars (`VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`) plus a subject address. The public key is passed to the client, for example via a loader.

## 3. The `web-push` library

| Fact | Source |
| --- | --- |
| Latest is **4.0.0**, published to npm 2026-11-06, GitHub release 2026-11-10; previous was 3.6.x (Aug 2023) | [npm registry][npm-wp]; [releases][wp-release] |
| v4 migrated the package to **ESM** (`"type": "module"`); requires Node ≥ 16 | [release notes][wp-release]; [npm][npm-wp] |
| No bundled types; `@types/web-push` is at 3.6.4 (written for v3) | [npm][npm-types] |
| Dependencies: `jws`, `asn1.js`, `http_ece`, `https-proxy-agent`. License MPL-2.0 | [npm][npm-wp] |
| Repo is active (last push 2026-11-10, 43 open issues, not archived) | [GitHub API][wp-repo] |

API surface we'd use ([README][wp-readme]; [source][wp-src]):

- `setVapidDetails(subject, publicKey, privateKey)`, called once at startup.
- `sendNotification(subscription, payload, { TTL, urgency, topic, timeout })`. Encrypts with `aes128gcm` by default. The default TTL is four weeks ([README][wp-readme]). Push services may keep messages for less time than requested; Apple and Mozilla cap retention at 30 days ([Apple][apple-webpush]; [autopush][autopush]; [RFC 8030 §5.2][rfc8030]). Most notifications go stale far sooner than that, so set it explicitly.
- On any non-2xx response it rejects with `WebPushError { statusCode, headers, body, endpoint }` ([`src/web-push-error.js`][wp-src]). Callers branch on `statusCode`.
- `generateRequestDetails()` returns the raw request without sending it, useful for tests or a custom HTTP client.
- The README still documents GCM API keys. These are legacy and not needed for VAPID-based push to modern browsers ([README][wp-readme]).

The library does **not** queue, retry, rate-limit, fan out to many devices, store subscriptions, or clean up expired ones. All of that is ours.

## 4. Registering the service worker in Vite + React Router 8

**Constraints from the platform**

- The script must be same-origin, served with a JavaScript MIME type, over a secure context (HTTPS; `localhost` counts) ([MDN register()][mdn-register]).
- Default scope is the script's directory. To control the whole app, the file must live at `/sw.js`, or the server must send `Service-Worker-Allowed: /` ([MDN register()][mdn-register]). Push itself doesn't need page control, but `pushManager.subscribe` needs an **active** worker, or it rejects with `InvalidStateError` ([W3C Push API][push-api]).

**Option A: plain file in `public/`**

- Vite serves `public/` at `/` in dev and copies it as-is to the build output ([Vite: The public Directory][vite-assets]).
- In this repo, `react-router-serve` mounts `express.static(build.assetsBuildDirectory)` at the public path (i.e. `build/client`, where Vite copied `public/`). It also mounts `express.static("public", { maxAge: "1h" })` as a fallback. Only `/assets/*` gets `immutable, maxAge: 1y` (`node_modules/@react-router/serve/dist/cli.js`, lines 115–120, v8.3.0). So `/sw.js` would be served from the site root without far-future caching.
- Trade-off: the file isn't bundled or type-checked by the app build. Writing it in TypeScript needs a separate compile step or a hand-written JS file. Type-checking a worker needs the `WebWorker` lib ([vite-plugin-pwa docs][vpwa-inject] describe the same tsconfig requirement).
- Registration: a `useEffect` in `app/root.tsx` (client-only) calling `navigator.serviceWorker.register("/sw.js")`. `root.tsx` currently has no manifest link or SW code (checked in repo).

**Option B: a Vite plugin**

- `vite-plugin-pwa` 2.0.0 (2026-10-03) supports Vite 3–8. Its `injectManifest` strategy compiles a custom SW, for example from `app/sw.ts`, and can also generate the web manifest. It is built on Workbox and assumes precaching, which you can turn off with `injectionPoint: undefined` ([vite-plugin-pwa inject-manifest guide][vpwa-inject]; [npm][npm-vpwa]).
- `@vite-pwa/react-router` exists but is **0.0.3** (published 2026-11-09) with a peer dependency on `rolldown`. It is very new, and its fit with this repo's Vite 7 setup is unverified ([npm][npm-vpwa-rr]).
- Trade-off: TypeScript SW and generated manifest, in exchange for a new dependency and Workbox concepts we don't otherwise need (Cadence has no offline/caching requirement).

## 5. Permission prompt rules

- **Spec**: `subscribe()` asks for the `"push"` permission if needed and rejects with `NotAllowedError` if denied. Permission persists once granted ([W3C Push API][push-api]).
- **Firefox**: since v72 (desktop) and v79 (Android), `Notification.requestPermission()` only shows a prompt in response to a user gesture (click, tap, key press). Otherwise it just shows a URL-bar icon ([Mozilla blog][moz-gesture]; [MDN BCD][bcd]).
- **Safari (macOS and iOS)**: requesting a push subscription "requires an explicit user gesture" ([WebKit: Meet Web Push][webkit-meet]). Apple says to call subscribe "immediately from the gesture's event handler" ([Apple][apple-webpush]). On iOS, `Notification` is undefined unless the page is a Home Screen web app ([MDN BCD][bcd]).
- **Chrome**: no hard gesture requirement in stable as far as we found. It auto-enrolls users and sites into a "quieter" prompt UI based on low acceptance rates ([Chromium blog, 2020][chrome-quiet]). Lighthouse flags prompting on load ([Chrome docs][lh-notif]). From **Chrome 155 on Android** the prompt is non-blocking and can **time out**, resolving `requestPermission()` to `"default"`. Users can opt in later from Site Controls, so Google recommends also listening for `permissions.query({name:"notifications"})` `onchange` ([Chrome blog, 2026-07-22][chrome-android-prompts]).
- **`userVisibleOnly: true` is mandatory in practice.** Chrome rejects `false` ([web.dev][webdev-subscribe]). Safari requires it and revokes subscriptions that receive pushes without showing a notification ([WebKit: Meet Web Push][webkit-meet]; [Apple][apple-webpush]).
- Design implication: a "Turn on notifications" button whose click handler calls `subscribe()` directly, with no `await` on unrelated work first, since Apple asks for the call to be immediate. It should handle `granted`, `denied` and `default`.

## 6. Storing PushSubscriptions

- Store the output of `subscription.toJSON()`: `endpoint`, `expirationTime` (often `null`), `keys.p256dh`, `keys.auth` ([W3C Push API][push-api]; [web-push README][wp-readme]). Apple says to store "endpoint and encryption keys together with the user's account information" ([Apple][apple-webpush]).
- One user can have many subscriptions (one per browser/device or install). Endpoints are opaque and the push service may change their format, so don't parse them ([Mozilla autopush][autopush]). `endpoint` works as a unique key.
- A sketch for Drizzle (not a decision): a `push_subscriptions` table with `id`, `userId → users.id`, `endpoint` (unique), `p256dh`, `auth`, `expirationTime` (nullable), `userAgent`/label (optional), `createdAt`, `lastSuccessAt`, `failureCount`. A schema change triggers this repo's `database-migrations` workflow.
- Endpoints needed: `POST` subscribe (upsert by endpoint, bound to the session user) and `DELETE`/unsubscribe. The client should re-send its subscription on each app load (via `pushManager.getSubscription()`) to self-heal after silent rotation.

## 7. Expired subscriptions (404/410) and rotation

| Push service | Status codes for a dead subscription | Source |
| --- | --- | --- |
| RFC 8030 (standard) | **404** for an expired subscription | [RFC 8030 §7.3][rfc8030] |
| Apple (`*.push.apple.com`) | **410** "device token has expired"; also 413 (>4 KB payload), 429 and 403 (`BadJwtToken`, `VapidPkHashMismatch`, …) | [Apple][apple-webpush] |
| Mozilla autopush | **404** invalid endpoint, **410** "no longer valid"; 413 over 4 KB; TTL max 2,592,000 s (30 days) | [autopush docs][autopush] |
| Google's guidance (FCM) | **404** expired → delete; **410** gone → delete; 429 with `Retry-After`; 413; push services must accept ≥ 4096-byte payloads | [web.dev web push protocol][webdev-protocol] |

What the server must do:

- On `WebPushError.statusCode` 404 or 410, delete the stored subscription.
- On 429 or 5xx, retry with backoff, honoring `Retry-After`.
- On 400 or 403, log it as our bug (bad headers or VAPID).
- On 413, shrink the payload.

`web-push` surfaces the status but does none of this ([source][wp-src]).

Client-side rotation:

- The spec's `pushsubscriptionchange` event fires in the SW when the browser refreshes or loses a subscription ([W3C Push API][push-api]).
- Support is uneven ([MDN BCD][bcd], data as of the 2026-11-13 commit): Chrome since **138**; Firefox since 44 but **partial** (no `oldSubscription`/`newSubscription` properties); Safari macOS since 16; **iOS Safari: not supported**.
- So server-side 404/410 pruning, plus re-syncing on app load, is the dependable path. `pushsubscriptionchange` is a bonus.

## 8. Browser support (MDN browser-compat-data, main branch commit 2026-11-13)

| Browser | `PushManager` | Notes |
| --- | --- | --- |
| Chrome / Edge | 42 / 17 | Android mirrors desktop |
| Firefox | 44 (Android 48) | gesture required for prompt (72 / Android 79) |
| Safari macOS | 16 | "macOS Ventura and later" |
| Safari iOS/iPadOS | **16.4** | "web apps saved to the home screen" only |
| Samsung Internet | mirrors Chrome | — |

Source: [MDN BCD][bcd]. The `web-push` README's own table agrees on these versions ([README][wp-readme]).

### iOS / iPadOS specifics

- **Home Screen only.** From iOS/iPadOS 16.4, Web Push works for Home Screen web apps, and the permission request must come from direct user interaction. Notifications show on the Lock Screen, in Notification Center and on Apple Watch, and integrate with Focus ([WebKit, 2023-02-16][webkit-ios]). In regular Safari tabs on iOS, the `Notification` interface is undefined ([MDN BCD][bcd]).
- **Manifest, iOS 16.4–18.x:** to open as a web app (rather than a bookmark), the site needed a manifest with `display: "standalone"` or `"fullscreen"`, or the legacy `apple-mobile-web-app-capable` meta tag ([WebKit][webkit-ios]; [MDN BCD note][bcd]: "The app's manifest must have a non-default `display` value").
- **Manifest, iOS/iPadOS 26+:** "By default, every website added to the Home Screen opens as a web app". The user can untick "Open as Web App". "There are now zero requirements for 'installability'". A manifest still supplies name, icons and so on when present ([WebKit: Safari 26.0][webkit-26]).
- **Recommended manifest content anyway:** `name`, `short_name`, `start_url`, `display: "standalone"`, `icons`. Add `id`, which iOS 16.4+ uses to sync Focus settings across devices ([WebKit][webkit-ios]). Icons can come from the manifest (since iOS 15.4) or `apple-touch-icon`, and `apple-touch-icon` wins if both exist ([WebKit][webkit-ios]). Cadence currently has only `public/favicon.svg` and no manifest (checked in repo).
- **Third-party browsers** on iOS 16.4+ can offer Add to Home Screen ([WebKit][webkit-ios]).
- **Discoverability:** there is no install prompt API on iOS. Users must use Share → Add to Home Screen themselves, so an in-app explainer is needed. (Inference from the above; no Apple source offers an install prompt.)
- **Server side:** allow outbound connections to `*.push.apple.com`. Apple requires TLS with SNI, caps payloads at 4 KB and stores messages for at most 30 days ([Apple][apple-webpush]).
- **Declarative Web Push:** Safari 18.4 (iOS/iPadOS Home Screen apps) and 18.5 (macOS) add `window.pushManager` and a JSON payload format (`{"web_push": 8030, "notification": {"title", "navigate", …}}`). The OS shows these notifications **without a service worker**, and a SW can still modify them ([WebKit: Meet Declarative Web Push][webkit-declarative]; [Safari 18.4][webkit-184]; [Safari 18.5][webkit-185]). It's now in the W3C Push API Working Draft of 2026-11-12 ([W3C Push API][push-api]). MDN BCD lists `window.pushManager` as Safari-only, with no Chrome, Edge or Firefox support ([MDN BCD][bcd]). The format is backwards compatible: non-declarative browsers deliver the same JSON to the SW `push` handler ([WebKit][webkit-declarative]).
- **EU:** Apple briefly removed Home Screen web apps in the EU in iOS 17.4 betas, then reversed that before release (statement of 2024-03-01, quoted in [9to5Mac][9to5-eu]; secondary source, as Apple's original Q&A text was no longer on its DMA page when checked). Current alternative-engine rules don't mention web apps ([Apple][apple-engines]).

## 9. What we'd build (hand-rolled)

1. **VAPID setup**: generate keys once, add `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` and `VAPID_SUBJECT` (a real `mailto:`/`https:`, not localhost) to env config and secret storage.
2. **Dependency**: add `web-push@4` (ESM, matches `"type": "module"`), plus local type declarations or `@types/web-push` (v3-era types).
3. **Manifest**: `public/manifest.webmanifest` with `id`, `name`, `short_name`, `start_url`, `display: "standalone"`, icons (needs PNG icons, plus optionally `apple-touch-icon`). Link it from `app/root.tsx` `links`.
4. **Service worker**: `public/sw.js` (or `app/sw.ts` via `vite-plugin-pwa` `injectManifest`) with handlers for `push` (always `showNotification`, tolerate the declarative JSON shape), `notificationclick` (focus or open the URL from `data`), and `pushsubscriptionchange` (re-subscribe and POST to the server where supported).
5. **Registration**: a client-only effect in `root.tsx` that registers `/sw.js` when `"serviceWorker" in navigator`.
6. **Opt-in UI**: a settings toggle or button. Its click handler calls `pushManager.subscribe({ userVisibleOnly: true, applicationServerKey })` immediately. Handle `default`/`denied`/`granted`, the Chrome-Android timeout, and `permissions.query` `onchange`. On iOS outside a Home Screen app (where `Notification`/`PushManager` are missing), show "Add to Home Screen" instructions instead.
7. **Schema + migration**: a `push_subscriptions` table (§6) via the repo's database-migrations skill, plus seed updates.
8. **API routes**: subscribe (upsert by endpoint for the session user) and unsubscribe. Re-sync `getSubscription()` on app load.
9. **Send service**: `app/lib/push.server.ts`, wrapping `sendNotification` with explicit `TTL`, `urgency` and `topic`, and a payload of at most 4 KB. Fan out to all of a user's subscriptions. Prune on 404/410, back off on 429/5xx, log 400/403/413.
10. **Delivery path**: decide whether sends happen inline in the request or via a background job or queue. Nothing like that exists today (inference: no queue or worker in `package.json`).
11. **Tests**: unit-test the send service's status handling by mocking `web-push` or using `generateRequestDetails`. Manually test on Chrome, Firefox, macOS Safari and an iOS 16.4+ device from the Home Screen. Push needs HTTPS outside `localhost`, so real-device iOS testing needs a deployed or tunnelled HTTPS origin.
12. **Ops**: allow egress to FCM, Mozilla and `*.push.apple.com`. Monitor failure rates. Never rotate VAPID keys casually.

## Implications for the decision

Neutral observations to weigh; not a recommendation.

- **What the library covers vs. what's left.** `web-push` covers the cryptography and the HTTP protocol, which are the hardest parts to get right. Everything else in §9 (manifest, SW, opt-in UX, storage, pruning, retries, fan-out, iOS onboarding) is app work a vendor SDK would partly absorb. The browser-side requirements (SW or declarative payload, user gesture, Home Screen on iOS) apply whether we build or buy, because they come from the browsers, not the sender.
- **Cost and lock-in.** Hand-rolled has no per-message fee: the browser vendors' push services are free and need no Apple developer account. Subscriptions are bound to *our* VAPID key, so they're portable to any sender holding that key. A vendor that generates its own VAPID key would make switching away mean re-subscribing users.
- **Dependency risk.** `web-push` 4.0.0 is days old, with an ESM migration and class refactor, and has no first-party types. 3.6.x was stable for three years. Either version is usable, but expect some friction on v4.
- **Stack fit.** The current stack can serve a `public/sw.js` unchanged. A Vite PWA plugin adds TypeScript SW and manifest generation, but the React Router–specific integration is at 0.0.x.
- **iOS reach is the biggest UX variable.** Push on iPhone/iPad requires users to add Cadence to their Home Screen first, which is easier on iOS 26 but still a manual step with no install prompt. No vendor can bypass this.
- **The platform is still moving.** Recent changes: Chrome 155 Android prompts (July 2026), Chrome 138 `pushsubscriptionchange`, Declarative Web Push in the 2026-11-12 Working Draft, iOS 26 installability. A hand-rolled system owns tracking these. A vendor would track them for us, at the cost of its abstractions.
- **Infrastructure gap.** Reliable sending (retries, fan-out, pruning) implies some background-job mechanism Cadence doesn't have. That cost applies to hand-rolled push and also to any notification channel we build ourselves.

## Sources (all checked 2026-11-13)

- [push-api]: W3C Push API, Working Draft 12 Nov 2026 — https://www.w3.org/TR/push-api/
- [rfc8030]: RFC 8030 Generic Event Delivery Using HTTP Push — https://www.rfc-editor.org/rfc/rfc8030.html
- [rfc8292]: RFC 8292 VAPID — https://www.rfc-editor.org/rfc/rfc8292.html
- [bcd]: MDN browser-compat-data (`api/PushManager.json`, `api/ServiceWorkerGlobalScope.json`, `api/Notification.json`, `api/Window.json`), main @ 2026-11-13 — https://github.com/mdn/browser-compat-data
- [mdn-register]: MDN `ServiceWorkerContainer.register()` — https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register
- [webkit-meet]: WebKit, "Meet Web Push" (2022-06-07) — https://webkit.org/blog/12945/meet-web-push/
- [webkit-ios]: WebKit, "Web Push for Web Apps on iOS and iPadOS" (2023-02-16) — https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
- [webkit-declarative]: WebKit, "Meet Declarative Web Push" (2025-03-27) — https://webkit.org/blog/16535/meet-declarative-web-push/
- [webkit-184]: WebKit Features in Safari 18.4 — https://webkit.org/blog/16574/webkit-features-in-safari-18-4/
- [webkit-185]: WebKit Features in Safari 18.5 — https://webkit.org/blog/16923/webkit-features-in-safari-18-5/
- [webkit-26]: WebKit Features in Safari 26.0 — https://webkit.org/blog/17333/webkit-features-in-safari-26-0/
- [apple-webpush]: Apple, "Sending web push notifications in web apps and browsers" — https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers
- [apple-engines]: Apple, "Using alternative browser engines in the European Union" — https://developer.apple.com/support/alternative-browser-engines/
- [9to5-eu]: 9to5Mac quoting Apple's 2024-03-01 statement (secondary) — https://9to5mac.com/2024/03/01/apple-home-screen-web-apps-ios-17-eu/
- [moz-gesture]: Mozilla, "Restricting Notification Permission Prompts in Firefox" (2019-11-04) — https://blog.mozilla.org/futurereleases/2019/11/04/restricting-notification-permission-prompts-in-firefox/
- [autopush]: Mozilla autopush HTTP API (marked obsolete in favour of GitHub docs; error codes still current there) — https://autopush.readthedocs.io/en/latest/http.html
- [chrome-quiet]: Chromium blog, "Introducing quieter permission UI for notifications" (2020-01-07) — https://blog.chromium.org/2020/01/introducing-quieter-permission-ui-for.html
- [chrome-android-prompts]: Chrome for Developers, notification prompts on Android (2026-07-22) — https://developer.chrome.com/blog/notification-prompts-android
- [lh-notif]: Lighthouse "requests notification permission on page load" — https://developer.chrome.com/docs/lighthouse/best-practices/notification-on-start
- [webdev-subscribe]: web.dev, "Subscribing a user" — https://web.dev/articles/push-notifications-subscribing-a-user
- [webdev-protocol]: web.dev, "The Web Push Protocol" — https://web.dev/articles/push-notifications-web-push-protocol
- [wp-readme]: web-push README — https://github.com/web-push-libs/web-push/blob/master/README.md
- [wp-src]: web-push source (`src/web-push-lib.js`, `src/web-push-error.js`) — https://github.com/web-push-libs/web-push/tree/master/src
- [wp-release]: web-push releases — https://github.com/web-push-libs/web-push/releases
- [wp-repo]: GitHub API repo metadata — https://api.github.com/repos/web-push-libs/web-push
- [npm-wp]: npm registry `web-push` — https://registry.npmjs.org/web-push
- [npm-types]: npm registry `@types/web-push` — https://registry.npmjs.org/@types/web-push
- [vite-assets]: Vite, Static Asset Handling — https://vite.dev/guide/assets.html
- [vpwa-inject]: vite-plugin-pwa, injectManifest guide — https://vite-pwa-org.netlify.app/guide/inject-manifest.html
- [npm-vpwa]: npm registry `vite-plugin-pwa` — https://registry.npmjs.org/vite-plugin-pwa
- [npm-vpwa-rr]: npm registry `@vite-pwa/react-router` — https://registry.npmjs.org/@vite-pwa/react-router
- Repo facts: `package.json`, `vite.config.ts`, `react-router.config.ts`, `app/root.tsx`, `public/`, `node_modules/@react-router/serve/dist/cli.js` (v8.3.0) in this repository.

[push-api]: https://www.w3.org/TR/push-api/
[rfc8030]: https://www.rfc-editor.org/rfc/rfc8030.html
[rfc8292]: https://www.rfc-editor.org/rfc/rfc8292.html
[bcd]: https://github.com/mdn/browser-compat-data
[mdn-register]: https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register
[webkit-meet]: https://webkit.org/blog/12945/meet-web-push/
[webkit-ios]: https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
[webkit-declarative]: https://webkit.org/blog/16535/meet-declarative-web-push/
[webkit-184]: https://webkit.org/blog/16574/webkit-features-in-safari-18-4/
[webkit-185]: https://webkit.org/blog/16923/webkit-features-in-safari-18-5/
[webkit-26]: https://webkit.org/blog/17333/webkit-features-in-safari-26-0/
[apple-webpush]: https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers
[apple-engines]: https://developer.apple.com/support/alternative-browser-engines/
[9to5-eu]: https://9to5mac.com/2024/03/01/apple-home-screen-web-apps-ios-17-eu/
[moz-gesture]: https://blog.mozilla.org/futurereleases/2019/11/04/restricting-notification-permission-prompts-in-firefox/
[autopush]: https://autopush.readthedocs.io/en/latest/http.html
[chrome-quiet]: https://blog.chromium.org/2020/01/introducing-quieter-permission-ui-for.html
[chrome-android-prompts]: https://developer.chrome.com/blog/notification-prompts-android
[lh-notif]: https://developer.chrome.com/docs/lighthouse/best-practices/notification-on-start
[webdev-subscribe]: https://web.dev/articles/push-notifications-subscribing-a-user
[webdev-protocol]: https://web.dev/articles/push-notifications-web-push-protocol
[wp-readme]: https://github.com/web-push-libs/web-push/blob/master/README.md
[wp-src]: https://github.com/web-push-libs/web-push/tree/master/src
[wp-release]: https://github.com/web-push-libs/web-push/releases
[wp-repo]: https://api.github.com/repos/web-push-libs/web-push
[npm-wp]: https://registry.npmjs.org/web-push
[npm-types]: https://registry.npmjs.org/@types/web-push
[vite-assets]: https://vite.dev/guide/assets.html
[vpwa-inject]: https://vite-pwa-org.netlify.app/guide/inject-manifest.html
[npm-vpwa]: https://registry.npmjs.org/vite-plugin-pwa
[npm-vpwa-rr]: https://registry.npmjs.org/@vite-pwa/react-router
