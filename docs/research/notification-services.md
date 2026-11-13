# Third-party notification services for Cadence

Research for [issue #6](https://github.com/adamp01/cohort-005-project/issues/6): which hosted notification services could give Cadence an in-app feed, Web Push, email through our existing Postmark account, and per-user preferences — and at what cost and lock-in?

- **Date checked:** 2026-11-13. Pricing and feature pages change often; re-check before committing.
- **Sources:** vendor docs, pricing pages, and official SDK repos only. Third-party write-ups and vendors' claims about *competitors* were not used. Where a vendor's own pages are silent, the entry says "not found" rather than guessing.
- **Cadence context:** React Router 8 (framework mode) on Node `react-router-serve`, one SQLite file via Drizzle, transactional email already sent through Postmark (`app/lib/mailer.server.ts`).

## Background: what Web Push on iOS requires (vendor-independent)

Every service below sits on the same browser standard, so its limits apply to all of them. Apple supports standard Web Push (Push API, Notifications API, service workers, VAPID) for **Home Screen web apps on iOS/iPadOS 16.4+** and for Safari 16 on macOS 13+. The permission request must come from a user gesture, and Safari revokes permission if a push arrives and the service worker doesn't show a notification ([Apple: Sending web push notifications in web apps and browsers](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)).

So for Cadence, iOS push means the app needs a web app manifest and an "Add to Home Screen" step, whichever vendor is chosen (or if we build it ourselves).

## Knock

| Aspect | Finding |
| --- | --- |
| React in-app feed | `@knocklabs/react` ships `NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton`, real-time toasts, a preference-center guide, and headless hooks for custom UI ([Knock React overview](https://docs.knock.app/in-app-ui/react/overview)). Feed retention is unlimited on all plans ([pricing](https://knock.app/pricing)). |
| Web Push | No first-party web push channel. Knock's own tutorial says "Knock does not have a first-party web push channel" and routes browser push through Firebase Cloud Messaging; the browser token is stored as Knock channel data ([Knock FCM web push tutorial](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm)). Supported push providers: APNs, FCM, Expo, Amazon SNS, OneSignal ([push overview](https://docs.knock.app/integrations/push/overview)). |
| iOS Safari | Through FCM. The tutorial's sample code tells iPhone users to "Add to Home Screen" first ([tutorial](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm)). |
| Postmark | First-class integration: you give it a Postmark server API key and From address per environment. It supports delivery and bounce webhooks back into Knock, and payload overrides (for example `MessageStream`). By default it sends to Postmark's `outbound` transactional stream ([Knock Postmark docs](https://docs.knock.app/integrations/email/postmark)). Postmark is still billed separately ([pricing FAQ](https://knock.app/pricing)). |
| Preferences | Each user has a `PreferenceSet` (JSON) that opts them out by channel type, category, and/or workflow. Knock checks it on every workflow run; a hosted preference center is also available ([Knock preferences](https://docs.knock.app/concepts/preferences)). Preferences are on every plan; per-tenant preferences are Enterprise-only ([pricing](https://knock.app/pricing)). |
| Pricing (small scale) | Free: 10,000 messages/month. Starter: $250/month including 50,000 messages, then $0.005 each. One "message" is one user on one channel, so in-app plus email counts as 2. Messages a preference suppresses aren't billed. Knock branding is removed only on Starter and up ([pricing](https://knock.app/pricing)). |
| Self-host / OSS | Not found. Client and server SDKs are on GitHub (for example [knock-node](https://github.com/knocklabs/knock-node)); the platform is hosted only. |
| Data residency | US. The sub-processor list puts AWS hosting in the USA ([sub-processors](https://knock.app/legal/subprocessors)). Knock says it "expect[s] to support EU data residency by the end of 2026" ([security docs](https://docs.knock.app/developer-tools/security)). |
| Server-side trigger | `npm install @knocklabs/node`, then `client.workflows.trigger("key", { recipients, data, actor })` ([knock-node README](https://github.com/knocklabs/knock-node); [trigger API](https://docs.knock.app/send-notifications/triggering-workflows/api)). |

## Novu

| Aspect | Finding |
| --- | --- |
| React in-app feed | `@novu/react` has an `<Inbox />` built from Bell, Notifications, InboxContent, and Preferences sub-components, plus hooks for custom UI. It needs React 18+ ([Novu Inbox overview](https://docs.novu.co/platform/inbox/overview)). |
| Web Push | Through push providers: FCM, Expo, APNs, OneSignal, Push Webhook, Pusher Beams, Pushpad ([Novu push integrations](https://docs.novu.co/platform/integrations/push)). There's no first-party VAPID channel; the push page lists only third-party providers plus a generic webhook. |
| iOS Safari | Depends on which push provider is used; Novu's docs don't address it directly. |
| Postmark | Supported as an email provider: you give it the Postmark Server API token and a verified From address ([Novu Postmark docs](https://docs.novu.co/platform/integrations/email/postmark)). |
| Preferences | Three layers: workflow channel defaults, a subscriber's global channel preferences, and a subscriber's per-workflow channel preferences (the most specific wins). Workflows marked "critical" can't be opted out of ([Novu preferences](https://docs.novu.co/platform/concepts/preferences)). |
| Pricing (small scale) | Free: 10,000 workflow runs/month, up to 5 workflows, 3 team members, 24-hour activity retention. Pro: from $30/month for 30,000+ runs. Team: from $250/month ([pricing](https://novu.co/pricing)). Billing is per *workflow run*, not per channel; the pricing page doesn't define a run precisely. |
| Self-host / OSS | Open core: the core is MIT and the `/enterprise` folder is under a commercial licence ([repo LICENSE-MIT](https://github.com/novuhq/novu/blob/next/LICENSE-MIT); [README, "License" section](https://github.com/novuhq/novu/blob/next/README.md)). Community self-host needs MongoDB, Redis (AOF on the queue cluster), and S3-compatible storage; a single VM is 4 vCPU / 8 GB plus its datastores ([self-hosting overview](https://docs.novu.co/community/self-hosting-novu/overview)). Self-hosted lacks Inbox snooze, branding removal, email delivery/engagement tracking, more than 2 environments, translations, MFA/RBAC, and allows only 1 team member ([Self-Hosted vs Novu Cloud](https://docs.novu.co/community/self-hosted-and-novu-cloud)). |
| Data residency | US and EU on every plan, including Free. Other regions (UK, SG, AU, JP, KR) are Enterprise ([pricing](https://novu.co/pricing)). The EU API is `https://eu.api.novu.co` ([TypeScript SDK](https://docs.novu.co/platform/sdks/server/typescript)). |
| Server-side trigger | `npm add @novu/api`, then `novu.trigger({ workflowId, to: { subscriberId, email }, payload })`. Set `serverURL` for EU or self-hosted ([TypeScript SDK](https://docs.novu.co/platform/sdks/server/typescript)). |

## Courier

| Aspect | Finding |
| --- | --- |
| React in-app feed | `@trycourier/courier-react` (React 18+) provides `CourierInbox`, `CourierInboxPopupMenu`, `CourierToast`, a preferences component, and a `useCourier()` hook for custom UI. Your backend issues a JWT per user ([Courier React SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web); [React inbox guide](https://www.courier.com/docs/guides/add-an-inbox-to-react)). |
| Web Push | Only through a third-party push provider: APNs, FCM, Expo, OneSignal, Airship, Amazon SNS, Pusher Beams, Pusher, MagicBell, and others ([Courier push providers](https://www.courier.com/docs/integrations/push/overview)). No first-party VAPID web push channel is listed. |
| iOS Safari | Depends on the push provider chosen. |
| Postmark | First-class: Server API token plus From address, with `MessageStream` overrides and support for Postmark templates. "Postmark bills you for email delivery, and Courier bills per send" ([Courier Postmark docs](https://www.courier.com/docs/external-integrations/email/postmark)). |
| Preferences | Subscription topics grouped into sections. Each topic defaults to `OPTED_IN`, `OPTED_OUT`, or `REQUIRED`. Users can pick their own channels per topic (`custom_routing`). The layers are topic default, then tenant default, then the user's choice. There's an embedded component or a Courier-hosted page ([preferences overview](https://www.courier.com/docs/recipients/preferences/overview); [model](https://www.courier.com/docs/recipients/preferences/model)). Preference-blocked sends still count toward usage ([overview](https://www.courier.com/docs/recipients/preferences/overview)). |
| Pricing (small scale) | Developer: free, 10,000 sends/month. Business: $0.005 per send, *including the first 10,000* (50k sends = $250). One send is one template to one recipient, however many channels it goes out on ([pricing](https://www.courier.com/pricing)). |
| Self-host / OSS | Not found. SDKs are public; the platform is hosted only. |
| Data residency | Separate US and EU (Ireland) datacenters with fully separate data. The EU API is `https://api.eu.courier.com` ([Courier datacenters](https://www.courier.com/docs/workspaces/datacenters)). The pricing page lists EU residency under Business ([pricing](https://www.courier.com/pricing)). |
| Server-side trigger | `npm install @trycourier/courier`, then `client.send.message({ message: { to: { user_id }, routing: { method, channels }, content } })` ([React inbox guide](https://www.courier.com/docs/guides/add-an-inbox-to-react)). |

## MagicBell

| Aspect | Finding |
| --- | --- |
| React in-app feed | `@magicbell/react` provides `Inbox`, `FloatingInbox`, `UserPreferences` (channel toggles per category), and a `ContextProvider` that takes a user JWT ([MagicBell React](https://www.magicbell.com/docs/libraries/magicbell-react)). |
| Web Push | First-party Web Push channel: you supply your own VAPID key pair, and it stores standard `PushSubscription` endpoint/keys as channel tokens ([Web Push channel](https://www.magicbell.com/docs/channels/web_push)). There's a `WebPushButton` component, and your service worker must `importScripts` MagicBell's script ([Web Components docs](https://www.magicbell.com/docs/libraries/magicbell-webcomponents)). |
| iOS Safari | Supported for Home Screen PWAs on iOS 16.4+ ([MagicBell web push page](https://www.magicbell.com/web-push)). The button has a dedicated "request install" state for iOS ([Web Components docs](https://www.magicbell.com/docs/libraries/magicbell-webcomponents)). |
| Postmark | **No native Postmark integration in the current docs.** The email channel lists SMTP, SendGrid, Mailgun, Ping Email, and Amazon SES ([email channel](https://www.magicbell.com/docs/channels/email)), and the v2 API's `/integrations/*` endpoints include `smtp` but no `postmark` ([API reference](https://www.magicbell.com/docs/api/reference)). MagicBell's marketing page does name Postmark ([email page](https://www.magicbell.com/email)), which contradicts the docs. One workable route is the SMTP integration pointed at Postmark's SMTP endpoint ([Postmark SMTP](https://postmarkapp.com/developer/user-guide/send-email-with-smtp)); Postmark calls SMTP "a migration route" with fewer features than its API (same source). |
| Preferences | Per-channel by notification category ([MagicBell React `UserPreferences`](https://www.magicbell.com/docs/libraries/magicbell-react); [in-app channel](https://www.magicbell.com/docs/channels/in_app)). |
| Pricing (small scale) | Builder: $0, 1,000 deliveries/month, 1 project. Startup: $249/month including 50,000 deliveries, then $0.0025 each. A delivery is one user on one channel ([pricing](https://www.magicbell.com/pricing)). |
| Self-host / OSS | Not found on MagicBell's own pages. |
| Data residency | No hosting region or residency option is stated on the pricing, docs, or privacy pages checked ([privacy policy](https://www.magicbell.com/privacy-policy)). Treat it as unknown and ask the vendor. |
| Server-side trigger | `npm install magicbell-js`, then `new Client({ token: PROJECT_JWT }).broadcasts.createBroadcast({ title, content, recipients: [{ external_id }] })` ([magicbell-js docs](https://www.magicbell.com/docs/libraries/magicbell-js)). |

## OneSignal

| Aspect | Finding |
| --- | --- |
| React in-app feed | **None for web.** OneSignal's in-app messages are mobile-SDK only ("Can I use in-app messages on web? No.") ([in-app overview](https://documentation.onesignal.com/docs/en/in-app-messages-setup)). OneSignal "does not currently store the history of messages sent to each individual user"; to build a feed you store the history yourself ([activity feed guide](https://documentation.onesignal.com/docs/en/create-an-activity-feed)). `react-onesignal` covers web push setup only ([React setup](https://documentation.onesignal.com/docs/en/react-js-setup)). |
| Web Push | Core product: Chrome, Firefox, Safari, Edge, and others. OneSignal handles VAPID ([web push setup](https://documentation.onesignal.com/docs/en/web-push-setup); [FAQ](https://documentation.onesignal.com/docs/en/web-push-setup-faq)). |
| iOS Safari | iOS/iPadOS 16.4+ Home Screen web app with a `display: standalone` manifest; the user subscribes by tapping inside the installed app ([iOS web push setup](https://documentation.onesignal.com/docs/en/web-push-for-ios)). |
| Postmark | **Not supported.** The email options are OneSignal Email, SendGrid, Mailgun, or Mailchimp Transactional ([email setup](https://documentation.onesignal.com/docs/en/email-setup)). |
| Preferences | No built-in preference UI ("OneSignal does not provide any APIs for creating the Preference Center layout"). You model topics as user tags and turn channel subscriptions on or off via the API ([preference center guide](https://documentation.onesignal.com/docs/en/preference-center)). |
| Pricing (small scale) | Free: 10,000 email sends/month, web push capped at 10,000 subscribers per send. Growth: from $19/month, plus $0.004 per web push subscriber, plus email at 20,000 free then $1.50 per 1,000 ([pricing](https://onesignal.com/pricing)). |
| Self-host / OSS | Not found. |
| Data residency | "Our primary data centers are located in the European Union" ([data & security FAQ](https://documentation.onesignal.com/docs/en/data-questions)). |
| Server-side trigger | `@onesignal/node-onesignal`: build a `Notification` and call `client.createNotification(...)` ([server SDK reference](https://documentation.onesignal.com/docs/en/server-sdk-reference)). |

## SuprSend

| Aspect | Finding |
| --- | --- |
| React in-app feed | `@suprsend/react` has drop-in `Inbox`, `NotificationsFeed`, and `Preferences` components; `@suprsend/react-core` has hooks for custom UI ([React SDK](https://docs.suprsend.com/docs/react-sdk)). The Free plan shows SuprSend branding in the feed ([pricing](https://www.suprsend.com/pricing)). |
| Web Push | First-party: SuprSend generates the VAPID key pair and needs no extra vendor account. You serve `serviceworker.js` and call `webpush.registerPush()` from a user action ([add a channel](https://docs.suprsend.com/docs/add-a-channel); [React web push](https://docs.suprsend.com/docs/react-webpush)). |
| iOS Safari | The docs pages checked don't say. SuprSend's own blog says iOS 16.4+ works only as an installed PWA ([SuprSend blog](https://www.suprsend.com/post/web-push-notifications-production-architecture)), which matches Apple's docs. |
| Postmark | First-class email vendor: Server Token plus verified From address, with a Postmark webhook back to SuprSend for delivery/open/click status ([SuprSend Postmark](https://docs.suprsend.com/docs/postmark)). |
| Preferences | Channel preferences plus category/sub-category preferences, a ready-made preference-centre UI, and a hosted preference page ([pricing comparison](https://www.suprsend.com/pricing); [React preferences](https://docs.suprsend.com/docs/preference-react-sdk)). **The "User Preferences" feature is listed under the Business plan** ($250/month), not Free or Essentials ([pricing](https://www.suprsend.com/pricing)). |
| Pricing (small scale) | Free: 10,000 notifications/month. Essentials: $100/month (billed annually) for 50,000, then $2 per 1,000. Business: $250/month (billed annually) for 50,000, then $5 per 1,000. One notification is one user on one channel; opted-out sends aren't billed ([pricing](https://www.suprsend.com/pricing)). |
| Self-host / OSS | Self-hosted and "Bring Your Own Cloud" deployments on Enterprise ([pricing](https://www.suprsend.com/pricing); [enterprise page](https://www.suprsend.com/enterprise)). Not open source. |
| Data residency | The SaaS offers "US or EU data residency" ([enterprise page](https://www.suprsend.com/enterprise)); the pricing page lists a custom data-centre region under Enterprise. Which plans get EU isn't stated clearly; confirm with the vendor. |
| Server-side trigger | `@suprsend/node-sdk` (v1.10.0 or later): `new Suprsend(key, secret).workflows.trigger(new WorkflowTriggerRequest({ workflow, recipients, data }))`. Recipients can be created inline ([Node trigger docs](https://docs.suprsend.com/docs/node-trigger-workflow-from-api)). |

## Pusher Beams

| Aspect | Finding |
| --- | --- |
| React in-app feed | None. Beams is a push-only API for iOS, Android, and Web ([Beams overview](https://pusher.com/docs/beams/)). |
| Web Push | Yes. With "Authenticated Users" your backend issues a Beams token and the browser calls `setUserId` ([publish to specific user](https://pusher.com/docs/beams/guides/publish-to-specific-user/web/)). |
| iOS Safari | Beams' Safari guide says "Safari does not support Web Push Notifications on iOS or iPadOS devices" ([Beams Safari](https://pusher.com/docs/beams/getting-started/web/configure-safari/)). Apple's current docs say iOS 16.4+ Home Screen web apps are supported ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)), so the Beams page is either out of date or describes a limitation of Beams itself. Unverified either way. |
| Postmark / email | None; Beams has no email channel. |
| Preferences | None built in. "Device Interests" (pub/sub topics) are the only targeting primitive besides user IDs ([Beams overview](https://pusher.com/docs/beams/)). |
| Pricing (small scale) | Sandbox: free, 1,000 subscribers. Startup: $29/month (10,000). Pro: $99/month (50,000) ([Beams pricing](https://pusher.com/beams/pricing/)). |
| Self-host / OSS | Not found. |
| Data residency | Not stated for Beams. Pusher's EU cluster is documented for Channels, not Beams ([Channels clusters](https://pusher.com/docs/channels/miscellaneous/clusters/)). Pusher services are provided by MessageBird UK Ltd ([Pusher security](https://pusher.com/security/)). |
| Server-side trigger | `@pusher/push-notifications-server`: `beamsClient.publishToUsers([...ids], { web: { notification: { title, body } } })`, 1–1,000 users per call ([Node server SDK](https://pusher.com/docs/beams/reference/server-sdk-node/)). |

## Comparison

The "approx. monthly cost" column uses an **illustrative assumption, not a forecast**: 1,000 active users each receiving ~20 notifications/month, all in-app and also emailed. That is 20,000 triggers and 40,000 channel-deliveries. Postmark costs are extra in every case.

| Service | React in-app feed | Web Push (iOS PWA) | Postmark | Preferences | Approx. monthly cost at the assumption | Self-host | EU residency |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Knock | Yes, prebuilt + hooks | Via FCM only | Native | Yes, all plans | $250 (Starter; 40k > 10k free) | No | Not yet (US; EU expected end-2026) |
| Novu | Yes, prebuilt + hooks | Via providers (FCM, OneSignal, Pushpad…) | Native | Yes, 3 layers | ~$30 (Pro; billed per workflow run) | Yes, MIT core (Mongo + Redis + S3) | Yes, all plans |
| Courier | Yes, prebuilt + hook | Via providers (OneSignal, FCM, Beams…) | Native | Yes, topics + channel choice | ~$100 (20k sends × $0.005) | No | Yes (Business) |
| MagicBell | Yes, prebuilt | First-party VAPID; iOS PWA | SMTP only (docs) | Category × channel | $249 (Startup; 40k > 1k free) | Not found | Not stated |
| OneSignal | No web feed | First-party; iOS PWA | Not supported | Build your own (tags) | ~$19+ plus usage (no feed) | No | EU primary DCs |
| SuprSend | Yes, prebuilt + hooks | First-party VAPID | Native | Yes, Business plan only | $250 (Business, for preferences; annual) | Enterprise only | Yes (plan unclear) |
| Pusher Beams | No | Yes; iOS unclear | No email | No | Free–$29 (push only) | No | Not stated |

## Implications for the build-vs-buy decision

These are neutral observations to feed the decision. They don't make it.

- **Only some vendors cover all four needs at once** (in-app feed, Web Push, Postmark, preferences). Knock, Novu, Courier, and SuprSend come closest. Knock, Novu, and Courier need a second provider (usually FCM or OneSignal) for browser push. SuprSend has first-party web push but puts preferences on its $250 tier. MagicBell has first-party web push but no documented native Postmark integration. OneSignal and Pusher Beams lack an in-app feed, so the feed would be built in-house anyway.
- **Billing units differ, and that changes cost.** Courier bills per template send regardless of channel count. Knock, MagicBell, and SuprSend bill per channel-delivery. Novu bills per workflow run. Adding Web Push as a third channel later would raise per-channel bills but not per-send bills. Every option bills Postmark separately.
- **Web Push on iOS needs PWA work in every scenario**, bought or built: a manifest, an "Add to Home Screen" flow, a user-gesture prompt, and a service worker ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)). A vendor removes VAPID and subscription storage, not that work.
- **Lock-in shows up in four places:** (1) workflows and templates authored in the vendor's dashboard, (2) user, preference, and feed history stored by the vendor, (3) the vendor's React components and service-worker script embedded in Cadence's UI, and (4) the server trigger call. Wrapping the trigger behind one internal function (the way `mailer.server.ts` wraps Postmark today) contains (4). (1) through (3) can't be contained that way.
- **Self-hosting isn't "free".** Novu's open core is the only self-host option for a team our size. It needs MongoDB, Redis, and object storage, which is far heavier than Cadence's single SQLite file. Its community edition also drops some features (snooze, branding removal, email activity tracking, 1 team member).
- **Data residency may decide it.** If Cadence must keep personal data in the EU, Knock (US-only today), MagicBell, and Pusher Beams (not stated) need confirming with the vendor. Novu, Courier, OneSignal, and SuprSend document EU options.
- **Identity bridging is needed for every hosted feed.** Each one needs a per-user token or JWT minted by our backend from the session (Courier, MagicBell, and Knock document this pattern explicitly). That's a new auth surface next to the existing DevUI session.
