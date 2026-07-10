<div align="center">

# لمة · Lammah

**A social map app that turns "we should hang out sometime" into "we're at the café, come."**

*Built with React Native (Expo) · Next.js · PostgreSQL/PostGIS · Redis · Socket.IO*

</div>

---

> **ملاحظة بالعربي:** «لمة» يعني الجمعة/الاجتماع العفوي بين الربع. التطبيق يحل مشكلة نعرفها كلنا: التخطيط للقاء الأصدقاء صار أصعب من اللقاء نفسه. بضغطة وحدة ترسل «يلا نلتم في المكان الفلاني» لكل ربعك، ومن يقدر يجي — يجي. هذا المستودع عرض تفصيلي للمشروع (بدون كود المصدر لأنه مشروع تجاري قيد التطوير).

---

## What is this?

Lammah (Arabic for *"a gathering"*) is a GPS-first social app in the spirit of Zenly, built for how my friends and I actually meet up: spontaneously. Nobody sends calendar invites to go get coffee. Someone just says *"يلا نلتم"* — "let's gather" — and whoever's free shows up.

The core loop is deliberately short:

1. Tap one button (in the app, or straight from a **home-screen widget**).
2. GPS lists nearby cafés, restaurants and malls — **sponsored venues rank first** (that's the business model).
3. Pick a place, pick who to invite (everyone, or a private subset).
4. Friends get a push notification and join with one tap.
5. Everyone sees everyone's live distance to the spot, and a group chat spins up automatically.

**Why no source code?** Lammah is a commercial product currently in closed beta with real testers. This repo is a deep-dive portfolio write-up instead: the feature set, the architecture, and a few engineering problems I'm genuinely proud of solving. If you're reviewing this for hiring purposes and want to go deeper on any part of it, I'm happy to walk through the actual code in a call.

---

## Screenshots

| Launch | Live map | Gatherings |
|:---:|:---:|:---:|
| ![Splash](screenshots/02-splash-glow.png) | ![Map](screenshots/04-live-map.png) | ![Gatherings](screenshots/05-gatherings.png) |

| Contact sync | Invite via share | Settings |
|:---:|:---:|:---:|
| ![Contact sync](screenshots/06-contact-sync.png) | ![Invite](screenshots/07-contact-invite.png) | ![Settings](screenshots/08-settings.png) |

*(Captured on the Android emulator during development — the small gear overlay is the Expo dev-client menu, not part of the app.)*

---

## Feature surface (all shipped, not a wishlist)

**The gathering itself**
- Create a لمة at any nearby place, schedule it (now / tonight / custom), and it auto-expires.
- Invitees accept, decline, **apologize after accepting** (host gets a proper "can't make it" push, not a silent drop), or **rejoin** after declining.
- The detail screen shows each person's **live distance in meters** from the spot.
- Host tools: edit the time, remove members, cancel — everyone's view updates in realtime.
- **Bill splitting**: set a total at creation and it re-splits automatically as more people accept.
- Forward a gathering to your own contacts, and invite friends who *aren't on the app yet* through the native share sheet (their WhatsApp/SMS — no SMS provider costs).

**Presence & the map**
- Live friend locations over WebSockets, with privacy controls (precise/approximate, sharing duration, off).
- Hand-rolled **grid clustering** — with the rule that sponsored pins are never swallowed by a cluster.
- Custom dark map style, marker selection animations, pan-to-load with skeleton "radar" ghost pins.

**Chat (the part that quietly became a full messenger)**
- Per-gathering group chat + 1:1 DMs.
- **Voice notes** (recorded with expo-audio, streamed as native multipart — details below), images, file attachments.
- WhatsApp-style **reactions**, **read receipts** (✓ / ✓✓ / ✓✓-read), and delivery tracking.
- Automatic system notices: *"X is near the place"*, *"X arrived"* — so the chat wakes up exactly when people start showing up.
- A rejecter/apologizer is cleanly cut off from the group chat (and its notifications) until they rejoin — membership is the single source of truth.

**Growth & discovery**
- **Contact sync**: match your phone book against registered users — only phone digests leave the device, names never do, nothing is stored server-side.
- 24-hour **stories** with view receipts, friends-only.
- Friend requests with realtime + push notifications.
- **Sponsored venues**: the monetization backbone. Sponsors rank first in every discovery surface, get proximity ad copy, and venues apply through an in-app "become a sponsor" form.

**Platform polish**
- Android **home-screen widget** with category shortcuts (café / restaurant / mall) deep-linking into the gather flow; iOS WidgetKit twin in progress.
- Full **Arabic + English** with runtime language switching and complete RTL support.
- Three-theme design system (warm light / espresso dark / calm) — every color in the app is a semantic token, zero hardcoded hex outside the design system.
- Custom onboarding: an SVG-spotlight guided tour built from scratch (mask holes + spring animations) after third-party libs fell short.
- A cinematic black launch sequence: OLED-black native splash flowing into an animated "ember" choreography (radial SVG glow, spring physics, zoom-through exit).
- Push notifications for invites, joins, messages, friend requests — with channels, deep links, and offline fan-out.

---

## Architecture

Monorepo, two apps, clean architecture on both sides. Dependencies point inward; framework code stays at the edges.

```
Lammah/
├─ Lammah.Client/          Expo SDK 56 · React Native 0.85 · React 19 · TypeScript strict
│  └─ src/
│     ├─ app/              expo-router file routes (thin — no logic)
│     ├─ features/         vertical slices: gatherings, chat, map, contacts, stories…
│     ├─ core/             pure domain + application (no React, no Expo, no I/O)
│     ├─ services/         adapters: api, realtime, location, notifications, storage
│     └─ design-system/    tokens, themes, primitives (Unistyles 3 — C++ theming engine)
│
└─ Lammah.WebApi/          Next.js 16 App Router · Prisma 7 · PostgreSQL + PostGIS
   └─ src/
      ├─ app/api/          route handlers (thin: validate → use-case → Result → response)
      ├─ core/             entities, use-cases, ports — one job per use-case
      └─ infrastructure/   Prisma repos, Redis, Twilio, Google, Expo push, better-auth
```

**The parts that matter:**

- **Geospatial**: PostGIS with `ST_DWithin` on a GiST index for "what's near me", with a Haversine fallback path. Sponsored-first ordering is enforced in the query itself — a client can't reorder it away.
- **Realtime**: a standalone Socket.IO server (Node) with a Redis adapter, separate from the request/response API — presence, live locations, message/receipt/gathering events all fan out through per-user rooms.
- **State**: server state lives in React Query with careful invalidation + optimistic updates; client state in small Zustand slices; MMKV for fast persistence; SecureStore for tokens.
- **Errors**: use-cases return `Result<T, AppError>` — no throwing across boundaries; one place maps errors to HTTP; structured pino logs with PII redaction.
- **Delivery**: backend + socket server + Postgres + Redis on Railway; the client ships through EAS builds with OTA updates for JS-only changes.

---

## Stack

| Layer | Choices |
|---|---|
| Mobile | Expo SDK 56, React Native 0.85 (New Architecture), React 19, TypeScript strict |
| UI | Unistyles 3 (C++ theming), Reanimated 4 + Moti (spring physics everywhere), Skia, expo-glass |
| Navigation | expo-router (file-based), native tabs & headers |
| Data | TanStack Query 5, Zustand 5, react-hook-form + zod, MMKV, SecureStore |
| Backend | Next.js 16 App Router, Prisma 7, PostgreSQL 16 + PostGIS, Redis (ioredis) |
| Realtime | Socket.IO 4 + Redis adapter (standalone server) |
| Auth | better-auth (email + Google), Twilio Verify wired for phone OTP |
| Infra | Railway (API, socket, DB, Redis), EAS Build + Update, Expo push |
| i18n | i18next — Arabic + English, full RTL |

---

## Engineering stories worth telling

These are the problems that didn't have a Stack Overflow answer.

**1 — The $155 Google Places bill.**
A handful of beta testers panning the map burned ~8,000 paid API calls in six days. Root cause: a 9-type category fan-out × radius tiers × a 2-minute cache. The fix was architectural, not a band-aid: a **two-layer cache** — expensive external results cached for 7 days per geohash cell (single-flighted so concurrent requests can't stampede), while our own sponsored venues merge in *fresh on every request* so a new sponsor appears instantly. Plus fan-out reduced to 4 core types, a hard daily API quota as a spend ceiling, and budget alerts. Result: Google gets called roughly once per area per week; the sponsored layer never goes stale; worst-case monthly cost is ~$0.

**2 — Voice notes that played locally but not in production.**
Classic "works on my machine": senders replay their own clip from the local file, so nobody noticed the server URL was broken. The real bug: our media endpoint returned a plain `200` with no **HTTP Range** support — and `.m4a` keeps its `moov` index atom at the *end* of the file, so native players (ExoPlayer/AVPlayer) must range-seek before they can start. Implemented proper `206 Partial Content` (suffix ranges included) and playback came alive everywhere.

**3 — Read receipts without a per-message read table.**
Instead of writing a receipt row per message per user (write amplification nightmare), each participant carries two **watermarks** — `lastDeliveredAt` / `lastReadAt`, stamped with the *database's* clock to dodge API clock skew. A message's ✓✓ state is derived from the MIN watermark across other participants. WhatsApp-grade UX, O(participants) storage.

**4 — The upload freeze, twice.**
Voice uploads first shipped as base64 JSON — and `JSON.stringify` on a megabyte-scale string visibly froze the JS thread. Moved to multipart… and React Native 0.85's bridgeless networking rejected the classic `FormData` file-part shape entirely. Final fix: native streaming upload (`uploadAsync` multipart), which reads the file off the JS thread. Wrapped it in an **optimistic outbox** — clips appear instantly, failures stay visible with a retry affordance instead of silently vanishing.

**5 — The Android crash that only hit testers.**
App opened fine in dev, crashed instantly for testers. The Maps API key was missing from the *built* manifest: the react-native-maps config plugin **removes** the key when its own config field is unset, even if the key exists elsewhere. Found it by dumping the actual APK manifest (`aapt dump xmltree`) instead of trusting configuration — a lesson that stuck: verify the artifact, not the intent. Also added a keyless-guard fallback screen so a config regression can never crash-loop users again.

**6 — Arabic is a first-class citizen, and it fights back.**
Cursive Arabic breaks if you apply `letterSpacing` (letters disconnect) — so the design system bans it on Arabic text. Native chrome (tab bars, headers, alerts) follows the OS appearance, not the app theme, causing light/dark flicker — solved globally by syncing `Appearance.setColorScheme` with the in-app theme. RTL direction changes need a full reload to apply cleanly. None of this is in the docs; all of it is in the app.

---

## Status

In closed beta with a live production backend, real testers, and OTA-updatable clients. Actively developed — the commit log this repo *doesn't* show is embarrassingly long.

## Contact

**AbdulHadi (Dev Abady)** — dev.abady@gmail.com

*I'd rather show you the code than describe it — happy to screen-share any part of the system in an interview.*
