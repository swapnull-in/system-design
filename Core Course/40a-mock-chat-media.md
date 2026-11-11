# Mock Interview Bank — Chat, Social & Media/Productivity

> **Interviewer-side companion.** This is the material you read *from* while running a 40-minute Staff/Principal mock on a candidate (or self-drilling). The actual reference designs live in the **18a/18b playbooks**; this file is the *question deck + rubric*, not a solution sheet.

## How to use this file

1. **Set a 40-min timer.** Read only the **Opening prompt** aloud. Say nothing else.
2. **Withhold the curveballs.** Don't volunteer scale numbers, read:write ratios, or consistency needs — make the candidate *ask*. Driving requirements is itself scored (see rubric). Only reveal a constraint when they ask for it.
3. **Throw the mid-interview constraint change** once they've committed to an architecture (~minute 20). It tests adaptability, not memory.
4. **Escalate the probes.** Walk the numbered follow-ups from medium → brutal. Stop when they tap out or run out of time. A Staff candidate should survive probe 6+; a Principal should survive the last one.
5. **Score at the end** with the [How to score](#how-to-score-staff-rubric) sheet. Score *behaviors*, not whether they "knew the answer."
6. **Altitude check:** at Staff/Principal you are listening for *trade-off articulation under pushback*, not feature recall. If they can only describe one design and can't defend why-not-the-alternative, that's a cap at Senior.

Cross-reference: **[20 Staff Execution](20-staff-execution.md)** and **[26 Google SD Playbook](26-google-sd-playbook.md)**.

## Problems

- [WhatsApp / Messenger](#whatsapp--messenger)
- [Slack](#slack)
- [Instagram](#instagram)
- [Twitter / X](#twitter--x)
- [Facebook News Feed](#facebook-news-feed)
- [LinkedIn](#linkedin)
- [YouTube (VOD)](#youtube-vod)
- [Netflix](#netflix)
- [Spotify](#spotify)
- [Zoom](#zoom)
- [Google Docs](#google-docs)
- [Dropbox](#dropbox)
- [Gmail](#gmail)
- [News aggregator (Google News)](#news-aggregator-google-news)
- [Live comments (Facebook Live)](#live-comments-facebook-live)
- [Strava (fitness activity tracking)](#strava-fitness-activity-tracking)
- [Facebook post search (no search engine)](#facebook-post-search-no-search-engine)
- [Tinder (location-based matching)](#tinder-location-based-matching)
- [Quora (Q&A platform)](#quora-qa-platform)
- [TikTok (short-video backend)](#tiktok-short-video-backend)
- [How to score (Staff rubric)](#how-to-score-staff-rubric)

---

### WhatsApp / Messenger

- **Opening prompt:** "Design WhatsApp. Focus on 1:1 and group messaging with delivery and read receipts across multiple devices. Take it where you think the hard parts are."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~2B users, ~100B messages/day, peak ~10M msgs/sec. Avg message tiny (<1 KB); media goes through a separate blob path.
  - Read:write: messaging is roughly 1:1 write-to-deliver but groups amplify (a 256-member group = 1 write → 255 deliveries). Group cap: start at 256, then change it (see below).
  - Consistency: per-conversation ordering is **required**; global ordering is not. At-least-once delivery with client-side dedup is acceptable.
  - E2E encryption is in scope (this is the differentiator vs a toy chat).
  - **Mid-interview constraint change #1:** "Now support *community* groups of 100k members." Watch them realize naive fan-out-on-write per message collapses.
  - **Mid-interview constraint change #2:** "A user has 4 active devices and adds a 5th offline phone." Forces the multi-device + key-distribution story.

- **The crux (what they MUST nail):**
  1. **Message ordering & the offline/multi-device delivery model** — per-conversation sequence numbers, server-side per-recipient queues, ack/retry, idempotent dedup, and how a newly-online device catches up.
  2. **E2E with multi-device** — Signal Double Ratchet is per-pairwise-session; with N devices you maintain N sessions (sender-keys for groups). They must explain the server stores *ciphertext + opaque key bundles*, never plaintext, and how that interacts with fan-out.

- **Follow-up probes (escalating):**
  1. How do you guarantee per-conversation ordering when a sender retries a message that actually did arrive?
  2. A message shows "single tick" forever — walk me through every layer where it could be stuck.
  3. Two of my devices are online; how do they stay in sync on which messages I've read? What's the read-receipt fan-out cost?
  4. Now E2E: where do keys live, what does the server see, and how do you add a 5th device without leaking history?
  5. Double Ratchet gives forward secrecy — what *breaks* if a device's session state is lost or restored from an old backup? (Test: out-of-order / skipped message keys.)
  6. Groups: fan-out-on-write per recipient vs sender-keys. For a 100k community, defend your choice and give me the write amplification math.
  7. Presence / "typing…" for a 100k group — what's the fan-out cost and how do you *not* do it the naive way?
  8. A region's message store is partitioned away. What's your availability vs durability call, and what does the user see?
  9. How do you do server-side spam/abuse detection when you can't read message content?
  10. **Brutal:** Encrypted backups to iCloud/Google Drive — design the key escrow so a user can restore on a new phone with just a PIN, without you (the server) being able to read messages. What's the attack surface?

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Per-recipient durable queue + sequence numbers; idempotent dedup keyed on client msg-id | "Just use Kafka" with no per-recipient delivery/ack model |
| Sender-keys for large groups (encrypt once, distribute the group key per-device) | Per-message per-recipient RSA/DH for a 100k group |
| Server stores only ciphertext + key bundles; plaintext never server-side | Hand-waves "we encrypt it" with no key-distribution story |
| Multi-device = N pairwise sessions or a device-key fan-out; new device gets no history by default | Assumes one device per user |
| Presence/typing throttled, coalesced, or pull-on-open for huge groups | Broadcasts every keystroke to every member |
| Honest at-least-once + client dedup; ordering is *per-conversation* not global | Claims exactly-once over the network |

- **Common traps:**
  - Promising global total ordering across all conversations (unnecessary, expensive).
  - Fan-out-on-write per recipient for very large groups — works at 256, dies at 100k.
  - Forgetting that read receipts and presence are their own fan-out problem, often *larger* than messages.
  - Treating E2E as a checkbox; not handling the multi-device key problem or backup/escrow.

---

### Slack

- **Opening prompt:** "Design Slack — channels, threads, and the real-time message + presence experience for a large enterprise workspace."

- **If they ask scope (curveballs to reveal):**
  - Scale: workspaces up to ~500k users; a busy channel has thousands of members; tens of thousands of concurrent WebSocket connections per large org.
  - Read:write: heavily read-skewed; history is re-read constantly (search, scrollback, unreads).
  - Consistency: ordering per channel matters; "unread count" can be eventually consistent.
  - Not E2E (enterprise needs server-side search, compliance, eDiscovery) — call this out as a deliberate difference from WhatsApp.
  - **Mid-interview constraint change:** "Add Slack Connect — a channel shared across two separate companies' workspaces." Watch the data-ownership / federation story.

- **The crux (what they MUST nail):**
  1. **Real-time fan-out at the connection layer** — how thousands of WebSockets in a channel get a new message; the role of a per-channel pub/sub and presence service; and why unread counts are computed lazily, not pushed.
  2. **Channel model & ordering** — message ts as ordering key, threads as a parent-reply structure, and how search/history is served separately from the realtime path.

- **Follow-up probes (escalating):**
  1. Walk a single message from "send" to appearing on 5,000 connected clients. Where's the fan-out?
  2. Presence for a 500k-user org — how do you compute "who's online" without melting the system?
  3. Unread counts and the red badge: push or pull? What's the cost of getting it wrong?
  4. Threads: data model, and how a thread reply notifies the right subset without spamming the channel.
  5. Search across years of history with permissions — how, and why is it not on the realtime path?
  6. A WebSocket gateway node dies holding 50k connections — what happens to those users and their in-flight messages?
  7. Slack Connect: messages in a shared channel — who stores the canonical copy, who enforces retention, and how do you handle one side's compliance hold?
  8. Enterprise key management / eDiscovery: how does compliance export coexist with "secure"?
  9. **Brutal:** A bot posts 10k messages/sec to a channel (runaway integration). How does your fan-out and unread-count system degrade gracefully instead of cascading?

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Per-channel pub/sub; gateway holds connections, subscribes to channels user is in | Broadcasting every message to every connection in the org |
| Unread = lazily computed from last-read marker vs channel ts cursor | Maintaining a live counter per user per channel pushed in realtime |
| Presence sampled/coalesced, not per-event | Per-event presence broadcast |
| Search/history as separate read path (inverted index, permission filter) | Search hits the realtime store |
| Honest "not E2E because enterprise search/compliance" | Bolting on E2E that breaks search without acknowledging it |

- **Common traps:**
  - Pushing unread counts in realtime instead of last-read-marker math.
  - Ignoring connection-layer scaling (the WebSocket fleet *is* the hard part).
  - No graceful degradation for hot channels / runaway bots.

---

### Instagram

- **Opening prompt:** "Design Instagram — photo/video upload, the feed, and likes/comments at billion-user scale."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~2B users; hundreds of millions of posts/day; feed reads vastly outnumber writes (read:write ~100:1+).
  - Media: images + short video; needs a transcode + multiple-resolution pipeline and a CDN.
  - Consistency: like-counts and feed freshness can be eventually consistent; a user must see *their own* new post immediately (read-your-writes).
  - **Mid-interview constraint change:** "A creator with 200M followers posts." Forces the celebrity/hybrid fan-out conversation.

- **The crux (what they MUST nail):**
  1. **Feed generation: fan-out-on-write vs fan-out-on-read, and the celebrity hybrid** — precompute timelines for normal users (push), but pull-and-merge for accounts with huge follower counts; explain the threshold and the merge-at-read.
  2. **Media pipeline** — async upload → object store → transcode to N resolutions → CDN; the post is "available" before transcode finishes (progressive).

- **Follow-up probes (escalating):**
  1. Fan-out-on-write vs on-read — pick a default and justify with the read:write ratio.
  2. Now the 200M-follower creator: what does pure fan-out-on-write cost per post? Design the hybrid and define the threshold.
  3. Read-your-writes: the celebrity's *own* post isn't in their precomputed feed — how do they see it instantly?
  4. Like counts on a viral post (1M likes/min): how do you display a count without a hot-row write bottleneck?
  5. Walk the upload path. When is the post "live"? What does the user see before transcode completes?
  6. CDN strategy: what's cached, cache key design, and how do you handle a deleted/edited post still in cache?
  7. Feed ranking is added — where does the ML scoring sit relative to fan-out, and what's the latency budget?
  8. Stories (24h TTL) — different access pattern; how does TTL + fan-out differ from the permanent feed?
  9. **Brutal:** A celebrity posts and 50M followers open the app in the same 60 seconds. Trace the thundering-herd through your read path and show every cache/queue that absorbs it.

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Hybrid fan-out with an explicit follower threshold + merge-at-read | One fan-out model for everyone |
| Approximate / sharded counters for viral like counts | Single-row `UPDATE count = count+1` for a viral post |
| Async transcode pipeline, progressive availability, N resolutions + ABR for video | Synchronous transcode blocking the post |
| Read-your-writes path for the author | Author can't see their own fresh post |
| CDN cache-key + invalidation on edit/delete | Ignores stale/cached deleted content |

- **Common traps:**
  - Committing to one fan-out strategy and not handling celebrities.
  - Hot-row counter for likes.
  - Synchronous media processing on the request path.

---

### Twitter / X

- **Opening prompt:** "Design Twitter's home timeline — tweet, follow, and the timeline read path at scale."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~500M users, ~500M tweets/day, timeline reads ~100x writes.
  - The follower graph is power-law: most users have few followers; a handful have 100M+.
  - Consistency: timeline can be eventually consistent; ordering is roughly chronological-or-ranked, not strict.
  - **Mid-interview constraint change:** "Switch the home timeline from reverse-chron to ML-ranked." Where does ranking live, and what's the cost?

- **The crux (what they MUST nail):**
  1. **The hybrid fan-out** (this is *the* Twitter question) — fan-out-on-write into per-user timeline caches for normal accounts, fan-out-on-read for celebrities, merged at read. They should reference the real threshold reasoning.
  2. **Timeline storage & merge** — per-user materialized timeline (e.g., Redis list of tweet IDs) + hydration from a tweet store; how the celebrity pull is merged in and deduped.

- **Follow-up probes (escalating):**
  1. Give me the write amplification of fan-out-on-write when @celebrity with 100M followers tweets.
  2. Define the threshold between push and pull. What signal triggers a user being treated as a celebrity?
  3. Timeline cache eviction: an inactive user comes back after a month — rebuild or pull? Cost?
  4. Hydration: timeline holds IDs; how do you fetch 50 tweets' content + author + counts in one screen's latency budget?
  5. Now ranking: where does the scorer run, what features, and what's the budget so the timeline still loads in <200ms?
  6. A tweet is deleted after fan-out — how does it disappear from 100M timelines?
  7. "Who to follow" / graph queries — how do you store and traverse the follow graph at this scale?
  8. Search + trending: realtime counting of hashtags — approximate counting (Count-Min / HLL)?
  9. **Brutal:** Compute the steady-state cost (writes/sec into timeline caches) for the whole system and tell me whether fan-out-on-write is even affordable globally — defend the hybrid with numbers.

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Hybrid push/pull with explicit celebrity threshold and merge-at-read | Pure fan-out-on-write "because reads are fast" with no celebrity handling |
| Timeline = IDs, content hydrated separately; bounded list length | Storing full tweet copies in every follower's timeline |
| Inactive-user rebuild-on-demand to bound cache cost | Maintaining timelines for all 500M users forever |
| Ranking as a separate scoring stage with a latency budget | Ranking inline blocking the read with no budget |
| Approximate counting for trends | Exact counting of every hashtag in realtime |

- **Common traps:**
  - Not doing the write-amplification math; just asserting a design.
  - Storing tweet bodies in timelines instead of IDs.
  - Forgetting deletion/edit propagation across materialized timelines.

---

### Facebook News Feed

- **Opening prompt:** "Design the Facebook News Feed — a ranked, mixed-source feed (friends, groups, pages, ads) for billions of users."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~3B users; feed is *ranked*, not chronological; sources are heterogeneous (friends, followed pages, groups, ads).
  - Read:write very read-heavy; freshness within minutes is fine; ad insertion is a hard requirement.
  - **Mid-interview constraint change:** "Now the ranking model is 10x more expensive per candidate." Forces candidate-generation vs ranking separation and budget discipline.

- **The crux (what they MUST nail):**
  1. **Candidate generation → ranking → blending** as distinct stages: cheaply gather candidates (fan-out/pull from many sources), then score a bounded set with the expensive model, then blend ads/diversity rules. This multi-stage funnel is the FB-specific signal.
  2. **Mixed-source aggregation** — unlike Twitter's single follow graph, the feed pulls from friends *and* groups *and* pages *and* ads, and must rank across them with a single objective.

- **Follow-up probes (escalating):**
  1. Why not just fan-out-on-write like a chronological feed? What does ranking force you to change?
  2. Candidate generation: how many candidates do you pull, from how many sources, and how do you bound it?
  3. The ranking model is expensive — how do you keep per-request cost sane (two-stage ranking, caching scores)?
  4. Ad insertion: where in the pipeline, and how do you reconcile the ad auction with organic ranking?
  5. Feed freshness vs cost — do you precompute ranked feeds or rank at read time? Trade-offs.
  6. Negative feedback ("hide this") and diversity rules — where do they apply?
  7. A user has 5,000 friends and 200 groups — how do you avoid scoring 50k candidates per load?
  8. **Brutal:** Give me the end-to-end latency budget for a feed load and allocate ms across candidate-gen, hydration, ranking, blending. Where do you spend the cache?

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Explicit multi-stage funnel: candidate-gen → light rank → heavy rank → blend | "Fetch posts and sort by a score" monolith |
| Bounded candidate set; two-stage ranking to control model cost | Running the heavy model on every possible candidate |
| Ads as a blended stage with the auction reconciled | Ads bolted on with no ranking interaction |
| Score/feature caching, freshness windows | Re-ranking everything on every refresh |
| Multi-source aggregation with one objective | Treating it as Twitter (single follow graph) |

- **Common traps:**
  - Conflating it with Twitter; missing the ranking funnel.
  - Running the expensive model on an unbounded candidate set.
  - No latency budget allocation.

---

### LinkedIn

- **Opening prompt:** "Design LinkedIn's core: the professional graph plus the feed, with emphasis on graph-distance features (1st/2nd/3rd degree)."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~1B members; the *graph* and "degrees of connection" are the differentiator.
  - Read:write: graph reads (degree, mutual connections, "people you may know") dominate; feed is secondary and ranked.
  - Consistency: connection acceptance should reflect quickly; degree computations can be cached/approximate.
  - **Mid-interview constraint change:** "Compute 2nd/3rd-degree distance and mutual-connection counts in the search results latency budget." This is the real LinkedIn graph problem.

- **The crux (what they MUST nail):**
  1. **Graph-distance at scale** — you cannot BFS the live graph per search query. They must propose precomputed/cached degree info, an in-memory graph service (sharded), and approximation for 2nd/3rd degree.
  2. **People-You-May-Know / mutual connections** — offline graph computation feeding an online serving layer.

- **Follow-up probes (escalating):**
  1. For a search result of 100 people, show me each one's degree (1st/2nd/3rd) — how, within the latency budget?
  2. Why is live BFS per query a non-starter? What do you precompute instead?
  3. Mutual-connection count between me and a stranger — exact or approximate? How stored?
  4. "People you may know" — offline pipeline design and how it updates as the graph changes.
  5. A member with 30k connections (super-connector) — how does that distort 2nd-degree computation and serving?
  6. Connection accepted — what has to invalidate/update, and how fast?
  7. Feed ranking with graph signals (affinity to connections) — where does the graph feature join happen?
  8. **Brutal:** Shard the graph across machines and explain how you serve a 2nd-degree query that spans shards without N network hops.

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| In-memory sharded graph service; precomputed/cached degree | Live BFS over a database per query |
| Approximation for 2nd/3rd degree and large mutual sets | Exact traversal at request time for everyone |
| Offline PYMK pipeline → online serving store | Computing recommendations synchronously |
| Super-connector handling (caps, sampling) | Ignoring power-law degree distribution |
| Cache invalidation on connection change | Stale degree forever, or invalidating everything |

- **Common traps:**
  - Treating it as just another feed and missing the graph-distance problem.
  - Live BFS / recursive queries at read time.
  - Ignoring super-connectors blowing up 2nd-degree sets.

---

### YouTube (VOD)

- **Opening prompt:** "Design YouTube's video platform — upload, transcoding, playback, and view counts. Not live streaming."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~500 hrs uploaded/min; billions of views/day; long-tail catalog plus a hot head.
  - Read:write: reads (playback) dwarf writes; CDN is essential.
  - Consistency: view counts can be approximate/delayed; uploads are async.
  - **Mid-interview constraint change:** "A video goes viral — 10M concurrent viewers in an hour." Tests CDN/origin-shield and view-count approximation under load.

- **The crux (what they MUST nail):**
  1. **Transcoding pipeline + Adaptive Bitrate (ABR)** — upload → chunk → parallel transcode to a ladder of resolutions/bitrates → package (HLS/DASH) → CDN; the player switches bitrate based on bandwidth.
  2. **View-count approximation** — you cannot do a strongly-consistent increment per view at this scale; they must propose batched/streamed approximate counting and de-dup of fraudulent views.

- **Follow-up probes (escalating):**
  1. Walk the upload-to-playable pipeline. Why chunk before transcoding? What parallelizes?
  2. Explain ABR — what does the player decide, on what signal, and what does the server pre-produce?
  3. The bitrate ladder: how many rungs, and how do you choose them? Storage cost vs quality.
  4. View counts at billions/day — exact is impossible; design the approximate counting + later reconciliation.
  5. Fraud: bots inflating views — where do you filter, and does it change the count pipeline?
  6. CDN: head vs long-tail caching, origin shield, and how a cold long-tail video is served.
  7. A video goes viral — trace the request path and show where the 10M concurrent viewers are absorbed.
  8. Thumbnails, captions, recommendations metadata — async side-pipelines; how do they attach?
  9. **Brutal:** Storage math — 500 hrs/min × a 6-rung ladder. Is storing every rung for every video affordable? Propose tiering / on-the-fly transcode for the cold tail.

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Chunked, parallel transcode → ABR ladder → HLS/DASH packaging | Single-resolution, synchronous transcode |
| Player-driven bitrate switching on measured throughput | "We pick the resolution server-side" |
| Approximate view counting (stream + batch reconcile) + fraud filter | Strongly-consistent per-view increment |
| CDN head/tail strategy, origin shield for viral | Origin serves all bytes; no CDN tiering |
| Cold-tier storage / on-demand transcode for long tail | Storing every rung for every video forever, no cost check |

- **Common traps:**
  - Synchronous/single-resolution transcode.
  - Exact view counts.
  - Forgetting ABR and just "stream the file."
  - No storage-cost reasoning for the rung ladder.

---

### Netflix

- **Opening prompt:** "Design Netflix's video-delivery system. Assume a fixed catalog (no UGC upload). Focus on getting bits to 200M+ subscribers globally with great QoE."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~200M+ subscribers; catalog is curated and finite (unlike YouTube's firehose); peak is a huge share of internet traffic in prime-time windows.
  - The differentiator vs YouTube: **pre-positioning content** (you know the catalog in advance) and **ISP-embedded edge caches (Open Connect)**.
  - Consistency: playback state (resume point) per profile; recommendations offline.
  - **Mid-interview constraint change:** "A blockbuster releases globally at midnight in each timezone." Tests pre-warming Open Connect appliances ahead of demand.

- **The crux (what they MUST nail):**
  1. **Open Connect edge caching** — Netflix ships appliances into ISPs; popular content is *pushed* to the edge during off-peak ("fill windows") so prime-time serves from inside the ISP. Candidate must contrast this with pull-CDN and explain why a finite catalog makes pre-positioning possible.
  2. **Per-title encode + ABR** — because the catalog is finite, Netflix optimizes encodes per title (per-shot/per-title encoding) for quality-per-bit; ABR delivery.

- **Follow-up probes (escalating):**
  1. How is this fundamentally different from YouTube? (Looking for: finite catalog → pre-position vs ingest firehose.)
  2. Open Connect: who pushes what, when, and how do you decide which titles land on which appliance?
  3. Fill windows: appliances refresh during off-peak — how do you predict tomorrow's demand per region?
  4. A new global release at midnight — pre-warm strategy so the edge already holds it before the spike.
  5. Per-title/per-shot encoding — why does a finite catalog let you do this and YouTube mostly can't?
  6. An ISP appliance is full / cache-misses — what's the fallback hierarchy (regional → origin)?
  7. Resume point / watch state across devices — small but real consistency problem; how?
  8. Recommendations + artwork personalization — offline pipeline; where does it intersect delivery?
  9. **Brutal:** Capacity-plan an Open Connect appliance for an ISP serving 1M subscribers in prime time — how much catalog can it hold, what hit-rate do you need, and what fraction must come from regional cache?

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Push-based pre-positioning into ISP appliances during fill windows | Generic pull-CDN; misses the Open Connect insight |
| Demand prediction driving what's cached where | "Cache the popular stuff" with no prediction/fill model |
| Per-title encoding leveraging the finite catalog | Treats encoding identically to YouTube |
| Fallback hierarchy edge → regional → origin | Single cache tier; no miss path |
| Pre-warm for known releases | Reactive caching that melts at midnight launch |

- **Common traps:**
  - Designing YouTube again; missing that the catalog is finite and pre-positionable.
  - Pull-CDN instead of push pre-positioning.
  - No demand-prediction / fill-window story.

---

### Spotify

- **Opening prompt:** "Design Spotify — music streaming, playlists, and the recommendation/discovery experience for hundreds of millions of users."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~500M users; ~100M tracks (finite catalog, far smaller files than video); playlists are a core social/data object.
  - Read:write: playback reads dominate; playlist edits and "liked" writes are modest.
  - Consistency: playlist edits should reflect quickly across a user's devices; offline download is a feature.
  - **Mid-interview constraint change:** "Add collaborative playlists edited by multiple users simultaneously." Introduces a concurrency/merge problem.

- **The crux (what they MUST nail):**
  1. **Audio delivery + caching + offline** — small files vs video; aggressive client + CDN caching; pre-fetch next track; offline download with license/DRM.
  2. **Recommendations / Discover Weekly** — offline collaborative-filtering + content embeddings pipeline feeding an online serving layer; this is Spotify's signal.

- **Follow-up probes (escalating):**
  1. Audio is small vs video — how does that change your CDN/caching and pre-fetch strategy?
  2. Gapless playback / pre-fetching the next track — where does that logic live?
  3. Offline downloads with licensing — DRM, expiry, and re-validation when back online.
  4. Playlist data model and syncing edits across a user's 3 devices.
  5. Collaborative playlists with simultaneous edits — how do you merge concurrent adds/reorders without clobbering? (Looking for ordering/CRDT-ish or operation log reasoning.)
  6. Discover Weekly: outline the offline pipeline (collaborative filtering + embeddings) → online recs store.
  7. Play counts / royalties accounting — this *needs* accuracy (money). How does that differ from a view counter?
  8. **Brutal:** Royalty accounting must be auditable and exactly accounted per stream for payouts. Design that counting path and contrast it with the *approximate* counting you'd use for "trending" — when do you choose which?

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Small-file CDN caching + next-track pre-fetch + offline DRM | Treats audio like video; no pre-fetch |
| Offline pipeline (CF + embeddings) → online serving for recs | "Just recommend popular songs" |
| Concurrent-playlist merge via op-log / ordering (acknowledges CRDT-style) | Last-writer-wins clobbering collaborative edits |
| Auditable, exact play counting for royalties; approximate for trending | One counting model for both money and trends |
| Multi-device playlist sync | Single-device assumption |

- **Common traps:**
  - Designing it like video streaming.
  - Using approximate counting where money (royalties) demands exactness.
  - Last-writer-wins on collaborative playlists.

---

### Zoom

- **Opening prompt:** "Design Zoom — real-time multi-party video conferencing. Take me through how the media actually flows."

- **If they ask scope (curveballs to reveal):**
  - Scale: meetings from 2 to thousands of participants; latency budget <150–200ms end-to-end for interactivity.
  - The core decision: **SFU vs MCU** for media routing.
  - Consistency: not a data-consistency problem; it's a real-time media + bandwidth problem.
  - **Mid-interview constraint change:** "Now a webinar with 10,000 view-only attendees and 5 active speakers." Forces a different topology (broadcast/CDN-like) than a peer meeting.

- **The crux (what they MUST nail):**
  1. **SFU vs MCU and the bandwidth math** — MCU mixes server-side (high server CPU, one stream down to each client); SFU forwards selectively (lower server CPU, more downstream bandwidth, client decodes multiple streams). They must do the per-participant upload/download math for an N-party call under each.
  2. **Simulcast / SVC + adaptive layers** — each sender publishes multiple quality layers; the SFU forwards the right layer per receiver based on their bandwidth and whether they're the active speaker.

- **Follow-up probes (escalating):**
  1. SFU vs MCU — define both and give me the bandwidth and CPU trade-off for a 50-person call.
  2. For an N-party SFU call, what's each client's upstream and downstream? Where does it break?
  3. Simulcast vs SVC — what does each sender publish and how does the SFU decide what to forward?
  4. Active-speaker detection — how, and how does it drive what each receiver gets in gallery vs speaker view?
  5. A participant's bandwidth drops — what adapts, where, and how fast (congestion control)?
  6. Packet loss on UDP — FEC vs retransmission; how do you keep latency under budget?
  7. Geo: participants on 3 continents — where do you place SFUs and how do you route between regions?
  8. The 10k-attendee webinar — why is SFU-per-attendee wrong, and what topology do you switch to?
  9. **Brutal:** A single SFU node serving a 200-person all-active call saturates downstream. Lay out the cascaded/tree SFU topology and recompute the bandwidth so it scales.

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| SFU default with explicit bandwidth/CPU math; MCU only where decode cost matters | Picks one with no math or no reason |
| Simulcast/SVC layered forwarding per receiver | Forwards every stream at full quality to everyone |
| Active-speaker driven forwarding; gallery vs speaker view | Sends all N streams to all N participants always |
| Congestion control + FEC under a latency budget | TCP/retransmit-everything; ignores latency |
| Cascaded SFUs + regional placement for scale/geo | Single SFU for all sizes and geographies |
| Webinar = broadcast topology, not per-attendee SFU | SFU forwarding to 10k view-only attendees |

- **Common traps:**
  - Choosing SFU/MCU without the bandwidth math.
  - Sending all streams to everyone (ignoring simulcast + active speaker).
  - Same topology for a 5-person call and a 10k webinar.
  - Using TCP/retransmission and blowing the latency budget.

---

### Google Docs

- **Opening prompt:** "Design Google Docs — real-time collaborative document editing where many users type in the same doc simultaneously."

- **If they ask scope (curveballs to reveal):**
  - Scale: docs with dozens of concurrent editors; low-latency local echo (typing feels instant); offline editing then reconcile.
  - The core decision: **Operational Transformation (OT) vs CRDT**, and the consistency goal (convergence — all replicas reach the same state).
  - Consistency: **Strong Eventual Consistency (SEC)** — concurrent edits must converge to one identical document regardless of delivery order.
  - **Mid-interview constraint change:** "Now support offline editing on a flaky connection for hours, then sync." Stresses the conflict-resolution model hard.

- **The crux (what they MUST nail):**
  1. **OT vs CRDT and the convergence guarantee** — OT transforms incoming ops against concurrent ops (typically needs a central server to order/transform; Google Docs is server-authoritative OT). CRDTs make ops commutative so order doesn't matter (better for P2P/offline) at the cost of metadata/tombstone overhead. They must articulate SEC and why ordering/transformation is the hard part.
  2. **The server-authoritative model** — local echo for responsiveness, ops sent to server, server assigns order + transforms, broadcasts canonical ops; clients reconcile.

- **Follow-up probes (escalating):**
  1. Two users insert text at the same position simultaneously — what does each see, and how do you converge to one document?
  2. OT vs CRDT — define both, and which would you pick for Google Docs and why?
  3. What does "server-authoritative" buy you in OT? What's the latency cost?
  4. Local echo: I type and see it instantly, but the server hasn't confirmed — how do you reconcile if the server reorders?
  5. CRDT downside: tombstones / metadata growth — how bad does it get on a heavily-edited doc, and how do you GC?
  6. Offline for 3 hours then reconnect with 1,000 local ops — how does each model handle it?
  7. Cursor/selection presence for 30 users — fan-out cost (echoes the chat presence problem).
  8. Undo in a collaborative doc — why is per-user undo hard, and how does it interact with OT/CRDT?
  9. **Brutal:** Prove your model actually converges. Give me a 3-user concurrent-edit scenario and walk the transforms/merges to show all three end with the identical document.

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Clear OT vs CRDT trade-off; SEC as the goal | "Just lock the doc" or last-writer-wins |
| Server-authoritative ordering + transformation; local echo + reconcile | No central ordering; hopes ops "just merge" |
| CRDT tombstone/GC awareness | Ignores CRDT metadata growth |
| Offline reconcile story for both models | No offline conflict story |
| Presence/cursor fan-out handled | Forgets cursor presence cost |
| Can actually demonstrate convergence | Asserts convergence without showing it |

- **Common traps:**
  - Locking / last-writer-wins (defeats the purpose).
  - Saying "use CRDT" without the tombstone/GC cost, or "use OT" without the central-ordering requirement.
  - Not articulating SEC / convergence as the actual goal.

---

### Dropbox

- **Opening prompt:** "Design Dropbox — file sync and storage across a user's devices, efficiently, with large files."

- **If they ask scope (curveballs to reveal):**
  - Scale: hundreds of millions of users; large files (GBs); frequent small edits to large files; multi-device sync.
  - Read:write: sync-driven; the key efficiency is *not re-uploading unchanged data*.
  - Consistency: a file's latest version should converge across devices; conflicts get conflicted-copies, not silent loss.
  - **Mid-interview constraint change:** "A 2 GB file changes by 10 KB in the middle." This is the whole point — block-level sync, not whole-file.

- **The crux (what they MUST nail):**
  1. **Content-addressable chunking + dedup + delta sync** — split files into chunks (content-defined chunking), hash each chunk, store/transfer only chunks whose hashes changed; cross-user dedup on identical chunks. A small edit syncs only the changed chunks.
  2. **Metadata vs block storage separation** — a metadata service tracks file → ordered chunk-hash list + versions; a block store holds content-addressed chunks (CDN/object store).

- **Follow-up probes (escalating):**
  1. A 2 GB file changes by 10 KB — how much data crosses the wire, and how do you achieve that?
  2. Content-defined chunking vs fixed-size blocks — why does an insert in the middle of a file favor one?
  3. Dedup: two users upload the same movie — how do you store it once, and what's the privacy concern?
  4. The metadata model: how does a file become an ordered list of chunk hashes + versions?
  5. Two devices edit the same file offline — how do you detect and resolve the conflict?
  6. Sync notification: how does device B learn device A changed a file (push vs poll)?
  7. A chunk's hash collides (or an attacker forges one) — security implications of content-addressing.
  8. Cold storage / tiering for files untouched for a year — how and when to move them.
  9. **Brutal:** Cross-user dedup leaks information (I can probe whether a chunk already exists → confirm you have a specific file). How do you keep dedup's efficiency without that side-channel?

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Content-defined chunking + hash-addressed blocks + delta sync | Whole-file re-upload on any change |
| Metadata service (chunk list + versions) separate from block store | One service storing whole files, no chunk model |
| Cross-user dedup with the privacy side-channel acknowledged | Dedup with no awareness of the existence-probe leak |
| Conflict → conflicted copy, never silent loss | Last-writer-wins silently dropping edits |
| Push-based change notification | Polling every file constantly |

- **Common traps:**
  - Whole-file transfer instead of block/delta sync.
  - Fixed-size blocks (an insert shifts everything; content-defined chunking avoids this).
  - Dedup without the existence-probe privacy side-channel.
  - Silent conflict loss.

---

### Gmail

- **Opening prompt:** "Design Gmail — email storage, sending/receiving, search, and the inbox experience at billion-user scale."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~1.8B users; huge total storage; search over years of mail per user is a headline feature.
  - Read:write: heavy reads (open inbox, search); writes are sends/receives; storage grows monotonically.
  - Consistency: an inbox should reflect new mail quickly; labels are eventually consistent; no data loss (durability is paramount).
  - **Mid-interview constraint change:** "Make full-text search over a user's entire 15-year mailbox return in <200ms." Forces the per-user inverted index design.

- **The crux (what they MUST nail):**
  1. **Per-user search index** — a per-user inverted index (not one global index), kept consistent with the mailbox, sharded by user; this is what makes "search my mail" fast and is the Gmail-specific hard part.
  2. **Storage & sharding model** — mailbox sharded by user; metadata (labels, threads, read state) vs message blobs; SMTP ingestion + spam pipeline before delivery.

- **Follow-up probes (escalating):**
  1. Inbound mail (SMTP) → delivered to inbox — walk the pipeline including spam filtering.
  2. Storage: how do you shard 1.8B mailboxes, and what's the metadata vs blob split?
  3. Search across a 15-year mailbox in <200ms — per-user inverted index; how is it built and kept current?
  4. Labels/threads — a message has many labels; how do you model that and keep counts fast?
  5. A new mail arrives — how does the index update, and is search read-your-writes for it?
  6. Spam/phishing at ingestion — where does ML sit, and how do you handle false positives (quarantine)?
  7. Attachments — store inline or as content-addressed blobs (dedup identical attachments)?
  8. Push to mobile (new-mail notification) — fan-out to a user's devices; offline catch-up.
  9. **Brutal:** A user with 2M emails runs a search; meanwhile mail keeps arriving. Show how the per-user index stays consistent under concurrent writes and serves that search within budget — and how you'd shard a single mailbox too big for one node.

- **Strong answer covers / Weak answer red flags:**

| Strong answer covers | Weak answer red flags |
|---|---|
| Per-user inverted index, sharded by user, updated on delivery | One global search index for all mail |
| Metadata (labels/threads/read state) vs blob separation | Storing everything in one row/blob |
| SMTP ingestion + spam ML before inbox; quarantine path | Mail lands in inbox with no anti-abuse pipeline |
| Durability-first (no mail loss), replicated storage | Treats email like best-effort data |
| Attachment dedup via content-addressing | Re-storing identical attachments per copy |

- **Common traps:**
  - One global search index instead of per-user.
  - Ignoring the SMTP ingestion + spam pipeline.
  - Under-weighting durability (email loss is unacceptable).
  - Not handling a single oversized mailbox.

---

### News aggregator (Google News)

- **Opening prompt:** "Design a news aggregator like Google News. Pull articles from tens of thousands of publishers, group them into stories, and show users a fresh, ranked feed by topic and region."

- **If they ask scope (curveballs to reveal):**
  - ~50k publishers, ~1.5M articles/day; ~100M DAU; feeds are mostly the same for everyone in a region.
  - Freshness matters: breaking news should appear within ~2–5 minutes.
  - Hundreds of outlets republish the same wire story with small edits.
  - **Mid-interview change #1:** "A major event breaks; reads spike 10× in two minutes and 400 outlets publish within ten. What happens?" (CDN for anonymous pages, velocity fast path, degrade personalization.)
  - **Mid-interview change #2:** "Product wants a 'For you' feed." (Thin per-user re-rank over segment lists, not per-user fan-out.)

- **The crux (what they MUST nail):**
  1. **Recognize it's not a social feed:** few sources, shared content → precompute per segment (topic × region), not fan-out per user.
  2. **Story clustering:** SimHash-style near-duplicate detection for syndicated copies, plus embedding/entity clustering for "same event, different words", with a batch merge/split repair and stable story IDs.
  3. **Freshness pipeline:** adaptive polling + push (WebSub), a stream pipeline, and a ranking cycle with a breaking-news fast path.

- **Follow-up probes (escalating):**
  1. BOTEC: articles/s, feed QPS, number of distinct segments. What dominates?
  2. How often do you poll each source, and why not a single interval?
  3. How do you detect that two articles are the same wire story?
  4. Two outlets cover the same event with completely different wording. How do they end up in one story?
  5. Rank stories, not articles: what signals, and how does breaking news rise fast?
  6. How does "For you" work without fanning out to 100M users?
  7. Page 2 shows duplicates because rankings changed. Fix pagination.
  8. **Brutal:** online clustering merged two unrelated events into one story card on the homepage. Detect and repair without changing story IDs clients hold.
  9. **Brutal:** a spam network floods thousands of low-quality "news" sites. Keep them out of top stories.
  10. **Brutal:** a publisher demands a takedown in 15 minutes across every surface and cache. Walk the path.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Says "not fan-out-on-write" early and precomputes ~10k segment lists; CDN for anonymous traffic.
  - ✅ Separates near-duplicate detection from event clustering; batch repair with stable IDs.
  - ✅ Freshness budget end to end (fetch + pipeline + ranker + cache TTL).
  - ❌ Copies the Twitter design (per-user timelines).
  - ❌ Dedupes by URL only; no notion of a story.
  - ❌ Single global poll interval; no push path.

- **Common traps:** pattern-matching to social feeds; treating clustering as a one-off batch job; forgetting cache invalidation for takedowns.
- **Reference design:** [44 — News Aggregator](44-design-news-aggregator.md).

---

### Live comments (Facebook Live)

- **Opening prompt:** "Design live comments for Facebook Live. Viewers of a live video post comments, and every viewer sees new comments in near real time."

- **If they ask scope (curveballs to reveal):**
  - ~1M concurrent live videos, heavily skewed: most have a few viewers, the top ones have 1–10M+.
  - Post → visible to others in under ~2 s.
  - Comments are ephemeral; losing one occasionally is acceptable.
  - **Mid-interview change #1:** "A stream goes from 1k to 5M viewers in a minute." (Expand the gateway subset; reconnect storms; sampling.)
  - **Mid-interview change #2:** "Add paid comments (Super Chats) that must always be shown." (Separate reliable path.)

- **The crux (what they MUST nail):**
  1. **The asymmetry:** writes are small; delivery volume is viewers × comments shown, so cap what each viewer receives (~10/s).
  2. **Routing to gateways:** per-video pub/sub plus **co-locating viewers of a video** on a popularity-scaled subset of gateways, so fan-in is bounded.
  3. **Best-effort semantics:** at-most-once, approximate ordering, no chat-style guarantees.

- **Follow-up probes (escalating):**
  1. SSE or WebSocket, and why?
  2. BOTEC: concurrent connections, gateway count, naive deliveries for the hottest video.
  3. A comment is posted on server A; viewers are on 2,000 gateways. How does it reach them?
  4. Why does random viewer placement hurt, and what do you do instead?
  5. 5,000 comments/s on one video: what does each viewer actually see?
  6. A new viewer joins: what do they see first?
  7. Moderation: where does the toxicity classifier run, and what's the trade-off?
  8. **Brutal:** a gateway dies with 100k viewers; all reconnect at once. Protect the fleet.
  9. **Brutal:** 10M viewers on one stream. Is pub/sub alone enough? (Relay tree.)
  10. **Brutal:** paid comments must never be dropped or sampled. Change the design without slowing the free path.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Per-viewer cap with priority classes (own, friends, creator) and batching into frames.
  - ✅ Viewer co-location by video ID; relay tier at extreme scale.
  - ✅ Explicitly best-effort, with reasons; paid comments on a separate path.
  - ❌ Delivers every comment to every viewer.
  - ❌ Builds WhatsApp: acks, per-recipient queues, ordering guarantees.
  - ❌ Broadcasts every comment to every gateway.

- **Common traps:** sizing by write volume instead of delivery volume; ignoring reconnect storms; putting ML moderation on the synchronous post path.
- **Reference design:** [45 — Live Comments](45-design-live-comments.md).

---

### Strava (fitness activity tracking)

- **Opening prompt:** "Design Strava. Athletes record runs and rides on their phone, upload them, see stats and a map, follow friends' activities, and compete on segment leaderboards."

- **If they ask scope (curveballs to reveal):**
  - ~5M activities/day, weekend-morning peaks ~5×; ~3,600 GPS points per hour of activity.
  - Recording often happens with no signal (trails).
  - Tens of millions of user-created segments.
  - **Mid-interview change #1:** "Add live location sharing with chosen safety contacts." (A small, separate real-time path.)
  - **Mid-interview change #2:** "Users discovered their home address is visible from public activities." (Privacy zones applied on read.)

- **The crux (what they MUST nail):**
  1. **Not a location firehose:** record offline on device, upload one file via presigned URL, with a local outbox and idempotency key.
  2. **Segment matching:** cell-based inverted index for candidates, then an ordered alignment with a GPS tolerance and interpolated gate times.
  3. **Leaderboards with filters** precomputed per (segment, bucket).

- **Follow-up probes (escalating):**
  1. How does the GPS track get to the server, and why not stream it?
  2. Guarantee a 3-hour ride is never lost with flaky connectivity.
  3. What happens after upload? Draw the pipeline.
  4. A ride passes through 2,000 segments' areas. How do you find which ones it actually rode?
  5. "Rode the segment" vs "passed near it": how do you tell, including direction?
  6. Leaderboards by age group, gender and this year: how stored and updated?
  7. "Among people I follow" leaderboard: precompute or compute on read?
  8. **Brutal:** a matching bug shipped a week ago. Fix every affected leaderboard.
  9. **Brutal:** someone logs a car ride as a bike ride and takes KOMs. Detect and contain.
  10. **Brutal:** privacy zones changed for a user with 5,000 old public activities. Make it effective everywhere, instantly.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Offline-first client, file upload, idempotent async pipeline that can be re-run.
  - ✅ Filter-refine segment matching with tolerance and direction.
  - ✅ Privacy zones enforced on every read surface, including heatmaps.
  - ❌ WebSocket GPS streaming during the activity.
  - ❌ Point-in-polygon "near the segment" as the match test.
  - ❌ Privacy handled by editing stored tracks at upload.

- **Common traps:** copying the Uber write path; forgetting reprocessing; leaderboard as `ORDER BY` over all efforts.
- **Reference design:** [49 — Strava](49-design-strava.md).

---

### Facebook post search (no search engine)

- **Opening prompt:** "Design search over Facebook posts. Users search by keyword and get posts sorted by recency or by likes. You can't use Elasticsearch or any search engine; build it."

- **If they ask scope (curveballs to reveal):**
  - ~500M new posts/day; ~10B likes/day; searches ~1B/day.
  - New posts searchable within ~1 minute; like ordering may lag minutes.
  - Assume all posts are public (privacy is a follow-up).
  - **Mid-interview change #1:** "A hashtag trends and gets 50M posts in an hour." (Hot terms: caps, time-bucketed lists, caching.)
  - **Mid-interview change #2:** "Now support friends-only posts." (ACL post-filter vs visibility-aware index.)

- **The crux (what they MUST nail):**
  1. **Inverted index from primitives:** term → posting lists, sharded by term, fed by a stream.
  2. **The like firehose:** don't keep a likes-sorted index exact; coalesce and update on threshold crossings, then exact re-rank of a few hundred candidates.
  3. **Bounding memory:** stop words, capped hot lists, cold tier.

- **Follow-up probes (escalating):**
  1. What data structure answers "posts containing X"? Shard it how?
  2. BOTEC: index appends/s for posts; index writes/s if every like updated the sort.
  3. Sort by recency: what's the score, and how do you paginate stably?
  4. Sort by likes: why is it hard, and what do you do?
  5. Multi-word query: how do you combine lists cheaply?
  6. What about "the"? What about a hashtag with 100M posts?
  7. A post is deleted. How does it leave search?
  8. **Brutal:** an index shard is lost. Recover without downtime.
  9. **Brutal:** change the tokenizer for every post ever indexed.
  10. **Brutal:** friends-only posts: what are the options and their costs?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Term-sharded postings with two orders; rarest-term-first intersection.
  - ✅ Log-bucket thresholds for likes + exact re-rank.
  - ✅ Index rebuildable from the log; caps and cold tier.
  - ❌ `LIKE '%term%'` or document-sharded scatter to every node for every query.
  - ❌ Updating every term's list on every like.
  - ❌ Unbounded lists for every term.

- **Common traps:** ignoring the like stream's volume; no pagination cursor; forgetting deletes.
- **Reference design:** [51 — Facebook Post Search](51-design-post-search.md).

---

### Tinder (location-based matching)

- **Opening prompt:** "Design Tinder. Users see a stack of nearby people who match their preferences, swipe right or left, and are notified when two people swipe right on each other."

- **If they ask scope (curveballs to reveal):**
  - ~50M DAU, ~2B swipes/day, evening peaks ~4×.
  - Matches must never be missed or duplicated.
  - Location changes slowly; distance is shown on profiles.
  - **Mid-interview change #1:** "Both people swipe right within the same 10 ms." (The race.)
  - **Mid-interview change #2:** "A security researcher located a user to within 50 m using the distance field." (Trilateration.)

- **The crux (what they MUST nail):**
  1. **Exactly-once match** under concurrent swipes: one row per unordered pair, atomic update, conditional match creation.
  2. **Precomputed recommendation stacks** from a geo + mutual-preference query, refilled asynchronously.
  3. **Seen-filtering** (Bloom filter backed by the swipe log) and **location privacy**.

- **Follow-up probes (escalating):**
  1. BOTEC: swipes/s, likes/s, stack builds/s.
  2. Store swipes how? Partition by what?
  3. "Write my like, then check theirs": what's wrong with it?
  4. Fix the race. Prove a match is created exactly once.
  5. How is the stack of candidates built, and when?
  6. Never show someone you already swiped on: how, cheaply?
  7. A retried swipe request arrives twice. What happens?
  8. **Brutal:** design "Who liked me" without scanning every pair row.
  9. **Brutal:** trilateration attack: what exactly do you change?
  10. **Brutal:** the stack builder is down for an hour on Saturday night.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Pair-keyed row with direction flags; outbox for the match event.
  - ✅ Stack per user with low-watermark refill; image preloading.
  - ✅ Coarsened stored location and rounded/fuzzed distances.
  - ❌ Two per-direction rows and a read to decide matches.
  - ❌ Geo query on every swipe.
  - ❌ Exact distances returned by the API.

- **Common traps:** missing the race entirely; Bloom filter false negatives assumed possible (they aren't); ranking fairness never mentioned.
- **Reference design:** [54 — Tinder](54-design-tinder.md).

---

### Quora (Q&A platform)

- **Opening prompt:** "Design Quora. Users ask questions, write answers, upvote and comment, search existing questions, and get a personalised feed."

- **If they ask scope (curveballs to reveal):**
  - ~300M monthly visitors, about 80% anonymous and arriving from search engines.
  - Writes are tiny: a few questions and answers per second, ~100 votes/s.
  - Answers on a question page must be ordered by quality.
  - **Mid-interview change #1:** "The first answer posted always ends up on top, even when a later one is better. Fix it." (ranking by evidence, exposure, exploration.)
  - **Mid-interview change #2:** "The same question exists in 40 variants." (duplicate detection and merge.)

- **The crux (what they MUST nail):**
  1. **Workload shape:** long-tail, read-heavy, anonymous → pre-assembled pages from cache/CDN; DB is the miss path.
  2. **Answer ranking** as a stored, asynchronously computed quality score, not a sort by votes.
  3. **Feed is pull by topic**, not fan-out-on-write; duplicates caught at ask time.

- **Follow-up probes (escalating):**
  1. BOTEC: page views/s vs writes/s vs corpus size. What does that say about where to spend effort?
  2. How is a question page served to an anonymous visitor?
  3. Why is sorting answers by upvotes wrong? What do you use instead?
  4. One vote per user per answer: where is that enforced?
  5. A viral answer gets 5k votes/s. What takes the load?
  6. How does the feed work when a topic has 10M followers?
  7. How do you stop the same question being asked repeatedly?
  8. **Brutal:** A vote ring pushes spam to the top of a popular question.
  9. **Brutal:** Merge two duplicate questions with 200 answers each without breaking links.
  10. **Brutal:** The content database is hot. Walk through what you do before sharding.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Says it is not Twitter: search-driven, old content is hot, cache first.
  - ✅ Wilson-style lower bound, exposure-normalised, credibility-weighted score stored on the answer.
  - ✅ Pull feed from per-topic lists; similar-question search at ask time.
  - ❌ Fan-out-on-write for topic follows.
  - ❌ `ORDER BY upvotes` at read time.
  - ❌ Shards a sub-terabyte dataset on day one.

- **Common traps:** copying the timeline design; ignoring anonymous traffic; counters on the answer row.
- **Reference design:** [57 — Quora](57-design-quora.md).

---

### TikTok (short-video backend)

- **Opening prompt:** "Design TikTok. Users upload short videos and scroll an endless personalised For You feed."

- **If they ask scope (curveballs to reveal):**
  - ~1B DAU, ~200 videos watched per user per day; ~30M uploads/day.
  - The feed is recommendation-driven, not based on who you follow.
  - A swipe must start playing almost instantly.
  - **Mid-interview change #1:** "A brand-new creator uploads a great video. How does anyone ever see it?" (cold start, staged exploration.)
  - **Mid-interview change #2:** "Ranking capacity is cut in half during an incident." (degradation ladder.)

- **The crux (what they MUST nail):**
  1. **No fan-out:** a per-request funnel (recall → filter → pre-rank → rank → re-rank) with a latency budget per stage.
  2. **Feedback loop:** watch events become features and counters within seconds.
  3. **Instant playback:** small first segment, prefetch of the next videos, CDN.

- **Follow-up probes (escalating):**
  1. How is this different from designing Twitter's timeline?
  2. BOTEC: feed requests/s, events/s, model scores/s.
  3. Where do candidates come from? Why more than one source?
  4. Why can't the heavy model rank everything?
  5. How do the last three videos I watched affect the next batch?
  6. Never show me the same video twice: how?
  7. Upload to first view: what happens in between?
  8. **Brutal:** New video, zero data: how does it get distribution without hurting the feed?
  9. **Brutal:** Ranking is overloaded. What does the user see?
  10. **Brutal:** The recommender keeps narrowing what people see. How do you detect and counter it?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Funnel with per-stage budgets; multi-source recall including an exploration pool.
  - ✅ Streaming features; batches of ~10 with early re-request.
  - ✅ Degradation ladder ending in cached trending, never an empty feed.
  - ❌ Precomputed per-user timelines via fan-out.
  - ❌ One model scoring the whole corpus.
  - ❌ No answer for cold start.

- **Common traps:** treating it as YouTube; forgetting moderation before distribution; ignoring egress cost.
- **Reference design:** [58 — TikTok](58-design-tiktok.md).

---

## How to score (Staff rubric)

Score each axis **1 / 3 / 5**. A **Staff** hire generally lands ≥3 on every axis and ≥4 on Altitude, Trade-off articulation, and Failure handling. A **Principal** lands 4–5 across the board, especially Decisiveness and Altitude under pushback. Average alone lies — a single **1** on Failure handling or Trade-offs is usually a no-hire regardless of average.

| Axis | 1 — No-hire signal | 3 — Senior / borderline | 5 — Staff/Principal |
|---|---|---|---|
| **Requirements-driving** | Starts designing before asking anything; takes scope as given | Asks some clarifiers when prompted | Proactively drives scope, surfaces hidden constraints, states non-goals and SLAs unprompted |
| **Estimation** | No numbers, or hand-waved | Does QPS/storage math when asked | Volunteers back-of-envelope (fan-out amplification, storage ladder cost, bandwidth) and lets it *drive* the design |
| **Altitude** | Stuck in code/API detail or vague boxes-and-arrows | Mostly right level, drifts occasionally | Fluidly moves system ↔ component ↔ data-structure on demand; knows when to zoom |
| **Decisiveness** | Can't commit; lists options endlessly | Picks a design, defends it weakly | Commits with a stated reason, names the runner-up, says what would change the call |
| **Deep-dive depth** | Falls apart past probe 3 | Survives to probe ~5 | Survives the brutal probe; goes deeper than asked on the crux |
| **Failure / detect-defend** | Doesn't consider failures | Mentions replication/retries when prompted | Unprompted failure modes, detection (metrics/alerts), graceful degradation, blast-radius control |
| **Trade-off articulation** | One option, no alternatives | Names alternatives, shallow comparison | Crisp why-not-X for each big call, with the cost/benefit and the regime where the answer flips |
| **Communication** | Rambling, no structure; interviewer must extract | Understandable, some structure | Leads the conversation, signposts, checks in, makes the interviewer's job easy |

**Running it:** note the highest probe number they cleared per problem — that's a fast proxy for Deep-dive depth. The mid-interview constraint change is your single best read on **Adaptability** (folds into Decisiveness + Altitude): a Staff candidate adjusts the design without restarting; a weaker one either ignores the change or throws everything out.

See **[20 Staff Execution](20-staff-execution.md)** for the behavioral/leadership signals that ride alongside these technical axes, and **[26 Google SD Playbook](26-google-sd-playbook.md)** for Google-EM-specific framing and the reference designs these probes are pulled from.
