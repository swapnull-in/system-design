# Mobile System Design — Google EM / Staff Track

> Companion track to the main course and [Deep Dives](../Deep%20Dives/00-INDEX.md). Backend system design asks *"how do a thousand servers serve a billion users?"* — mobile system design asks *"how does **one device** with a flaky radio, a hostile OS, and a 4-week release train deliver a flawless experience?"* Different physics, different interview.

## Why this track exists

At Google, mobile engineers and EMs get **client-side system design** rounds: "Design the Photos app", "Design an image loading library", "Design offline sync for Docs". The evaluation altitude shifts with level — Senior defends component trade-offs, **Staff defends long-term evolution and cross-team contracts, EM defends business risk, rollout safety, and privacy posture**. The main course won't save you here: consistent hashing never comes up, but process death always does.

## Sources

- **[weeeBox / mobile-system-design](https://github.com/weeeBox/mobile-system-design)** — the de-facto standard framework (5-step interview loop) + solved exercises (chat app, image library, file downloader, caching library)
- **[Educative — Grokking the Mobile System Design Interview](https://www.educative.io/courses/mobile-system-design)** — SCADET framework + case studies (newsfeed, chat, Google Maps, YouTube, stock trading, ride hailing) *(premium — used as outline)*
- **[Manuel Vivo — *Mobile System Design Interview*](https://medium.com/@manuelvicnt/mobile-system-design-interview-book-b5874008d941)** — 5-step playbook + 7 solved cases (news feed, chat, stock trading, pagination library, hotel reservation, Google Drive, YouTube) *(book — used as case-study roadmap; all 7 are written in this track)*
- Android/iOS platform docs for the numbers (startup budgets, background limits, cache sizing)

## How to use

- Start with **[01 — The Framework](01-framework.md)**: the interview loop, the standard client anatomy, and the Google altitude ladder. Everything else hangs off it.
- Foundations (02–07) are the **deep-dive menu** — every case study picks 2–3 of them as its crux.
- Case studies (08–15, 23–24) and libraries (16–22) are **reps**: run each as a 35-minute self-interview before reading, then check yourself against the Follow-Ups section.
- Each note ends with an **Interview Cheat-Sheet** — the 10-minute pre-interview refresher.

## Foundations

| # | Topic | The one-line take | Status |
|---|-------|-------------------|--------|
| 01 | [The Framework & the Google Altitude](01-framework.md) | 5 steps, one standard anatomy, and what Staff/EM answers sound like vs Senior | ✅ Done |
| 02 | [Client Architecture Patterns](02-client-architecture.md) | UDF (state down, events up) + repository owns truth + screen = f(synced state); process death as the test | ✅ Done |
| 03 | [Networking & Real-Time on Mobile](03-networking-realtime.md) | Radio tail → batch or die; round trips decide the protocol; the real-time ladder (lowest rung that meets latency) | ✅ Done |
| 04 | [Data, Caching & Offline-First Sync](04-data-caching-sync.md) | Caches hold what you can lose, DB holds what you can't; outbox + delta-sync; the conflict ladder; staleness tiers | ✅ Done |
| 05 | [Performance, Battery & Startup](05-performance-battery-startup.md) | The numbers block as design rules; cold start renders from DB; w×h×4; battery = wakes | ✅ Done |
| 06 | [Security & Privacy on Device](06-security-privacy.md) | Four adversaries, named controls; keys in hardware, best storage is absence; the APK is public; deletion is a code path | ✅ Done |
| 07 | [Release Engineering & Experimentation](07-release-engineering.md) | Can't roll back a binary → ship dark, release by config; trains, gated ramps, the version tail, flag-debt ratchet | ✅ Done |

## App case studies

> Every written case study includes escalating **Follow-Ups** (the probes after the probes) and a dedicated **Staff/EM layer** section.

| # | Question | Crux | Status |
|---|----------|------|--------|
| 08 | [Design a Chat App](08-chat-app.md) | Outbox + client UUIDs, ordering, batched receipts, one sync path (home turf!) | ✅ Done |
| 09 | [Design a News Feed (Twitter/Instagram)](09-news-feed.md) | Opaque cursors + cache-then-network window + image budget + optimistic writes | ✅ Done |
| 10 | [Design Google Maps (mapping client)](10-google-maps.md) | Tile pyramid + vector rendering, on-device reroute, offline = tiles+index+graph | ✅ Done |
| 11 | [Design YouTube (video client)](11-youtube.md) | ABR, player as arbitrated resource, downloads + DRM, rebuffer-gated rollouts | ✅ Done |
| 12 | [Design Google Photos (upload/sync)](12-google-photos.md) | Hash-first dedup, chunked resumable queue, DONE is a proof state; free-up-space deletes only against live server proof | ✅ Done |
| 13 | [Design Stories (ephemeral content)](13-stories.md) | Prefetch ring follows the social graph; expiry = server clock + tombstones + render check; the cache that empties itself | ✅ Done |
| 14 | [Design a Ride-Hailing App (Uber)](14-ride-hailing.md) | Server-authoritative trip machine; driver produces location on a battery budget, rider renders smooth motion | ✅ Done |
| 15 | [Design a Stock Trading App](15-stock-trading-app.md) | Two planes: lossy-fast quotes vs certain-slow orders; fail-closed money | ✅ Done |
| 23 | [Design the Google Drive App](23-google-drive.md) | Metadata/content split, change-log sync, conflicted copies, shadow-mode rewrites | ✅ Done |
| 24 | [Design a Hotel Reservation App](24-hotel-reservation.md) | Persisted funnel state machine, staleness tiers, holds + 3-DS-proof payment | ✅ Done |
| 25 | [Design TikTok / YouTube Shorts](25-tiktok-short-video.md) | The inversion of [11](11-youtube.md): player pool, tiered preload, waste ratio as the governing metric | ✅ Done |
| 26 | [Design Spotify (offline-first music)](26-spotify-music.md) | Playback service + media session; encrypted downloads with contractual expiry; Connect = remote control | ✅ Done |
| 27 | [Design a Collaborative Editor (Docs/Notion)](27-collaborative-editor.md) | Local echo + off-main CRDT engine; OT-vs-CRDT argued; undo/cursors/week-offline merges | ✅ Done |
| 29 | [E-commerce Checkout: The Double-Purchase Problem](29-ecommerce-checkout.md) | Six ways one tap becomes two charges; key lifecycle + UNKNOWN-resolves-by-query; the server contract Staff negotiates | ✅ Done |
| 30 | [Banking: The Debit–Credit Problem (Money Transfer)](30-banking-money-transfer.md) | Double-entry ledger; one boundary = ACID txn, two = saga + suspense account; debit-first as law; reconciliation as subsystem | ✅ Done |

## Library / SDK design (the Staff favorite)

> Google loves these for Staff: no product to hide behind — pure API design, resource limits, and failure semantics. SDK-building experience is the hook.

| # | Question | Crux | Status |
|---|----------|------|--------|
| 16 | [Design an Image Loading Library (Glide/Coil)](16-image-loading-library.md) | Sized-bitmap memory cache vs bytes disk cache; downsample-at-decode; lifecycle cancellation | ✅ Done |
| 17 | [Design a File Downloader / Uploader Library](17-file-downloader.md) | Transfers outlive screens (the [16](16-image-loading-library.md) inversion); persisted state machine; Range+If-Range; rename is the commit | ✅ Done |
| 18 | [Design a Caching Library](18-caching-library.md) | Read-through `get(key, loader)` + single-flight; two budgeted tiers; journaled crash-safe disk; freshness as knobs | ✅ Done |
| 19 | [Design a Pagination Library](19-pagination-library.md) | The API surface is the deliverable: PageSource + generations + per-direction errors | ✅ Done |
| 20 | [Design an Analytics SDK](20-analytics-sdk.md) | The queue that's allowed to lose data — on purpose, counted; batching, dual timestamps, consent-first | ✅ Done |
| 21 | [Design an A/B / Feature-Flag SDK](21-ab-feature-flag-sdk.md) | Bootstrap layers + snapshot-per-session; exposure-on-evaluation; the kill-switch fast lane ("can't roll back an APK", delivered) | ✅ Done |
| 22 | [Design a Push Notification System (client side)](22-push-notification-system.md) | Token lifecycle as the real design; doorbell payloads → one delta-sync path; channels/grouping; reachability as pager metric | ✅ Done |
| 28 | [Design a Large-Scale App Modularisation Strategy](28-modularisation.md) | The purest EM/Staff question: Conway's law on purpose, api/impl seams, CI-enforced ratchet | ✅ Done |
