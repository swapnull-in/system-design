# Mock Interview Bank — Advanced & Google-Style

> **Interviewer-side companion.** This is the page *you* read when you run the mock — not the candidate. It is the adversarial counterpart to the design modules: each problem gives you an opening prompt, a set of curveballs to reveal scope-thinking, the *one crux the candidate must nail*, escalating follow-up probes (medium → brutal), a strong/weak rubric, and the traps to bait. The bias here is **Staff/Principal/Google-EM**: deterministic systems, exactly-once *effect* (not delivery), correct BOTECs, and "detect-defend-recover from failure" applied to everything.
>
> **This bank targets a Staff/Principal or Google EM system-design round.** It deliberately pushes into matching engines, migrations, schedulers, and primitives they must *build*, not name.

## How to use this page

1. **Open cold.** Read only the *Opening prompt* aloud. Do not volunteer scope. Let them clarify — silence is a probe.
2. **Hold curveballs in reserve.** Drop the *mid-interview constraint changes* the moment the design gets comfortable. A good candidate re-derives; a weak one re-architects from scratch or freezes.
3. **Grade the crux first.** If they miss the crux, the rest is decoration. Note it, then keep probing to see ceiling.
4. **Bait the traps.** Listed at the end of each problem. The big four to catch every time:
   - "**Exactly-once delivery**" claims → there is no such thing over a network; push them to *exactly-once effect* (idempotency + dedup).
   - **DAU-as-RPS BOTEC** → 100M DAU is *not* 100M RPS. Make them divide by seconds/day and apply a peak factor.
   - **Naming a product instead of building it** ("I'd use Kafka / DynamoDB / Flink") → "Pretend that product doesn't exist. Build the mechanism."
   - **Ignoring failure** → if they never mention what happens when a node dies mid-operation, that is an automatic ceiling on the score.
5. **Score** against the shared rubric — see [40a](40a-mock-chat-media.md).

## Table of Contents

- [Matching engine / stock exchange](#matching-engine--stock-exchange)
- [Large-scale zero-downtime migration](#large-scale-zero-downtime-migration)
- [Code deployment system](#code-deployment-system)
- [Real-time analytics / ad-click aggregation](#real-time-analytics--ad-click-aggregation)
- [Distributed job scheduler](#distributed-job-scheduler)
- [Live streaming](#live-streaming)
- [Feature-flag / experimentation platform](#feature-flag--experimentation-platform)
- [Collaborative whiteboard](#collaborative-whiteboard)
- [IoT ingestion](#iot-ingestion)
- [Online judge (LeetCode)](#online-judge-leetcode)
- [Online chess (Chess.com / Lichess)](#online-chess-chesscom--lichess)
- [ChatGPT-style LLM chat app](#chatgpt-style-llm-chat-app)
- [Google "Build From Scratch" drills](#google-build-from-scratch-drills)
- [Scoring](#scoring)

---

### Matching engine / stock exchange

- **Opening prompt:**
  "Design the core matching engine for an exchange — equities or crypto, your call. Orders come in (limit/market, buy/sell), they match against a central limit order book, fills go out, market data is published. I care about correctness and latency."

- **If they ask scope (curveballs to reveal):**
  - Symbols: ~3,000 active. One hot symbol can do 500k orders/sec at the open; aggregate ~5M orders/sec.
  - Latency target: **single-digit microsecond** matching, p99.9 (not p50) is the SLO.
  - Order types: start with limit + market + cancel. Mention IOC/FOK only if they bring it up.
  - Durability: a confirmed order can **never** be lost or double-executed — this is money and it's regulated.
  - **Mid-interview change #1:** "A single symbol now exceeds what one core can handle. What changes?" (Correct: partition *by symbol*, never shard within a symbol — determinism breaks.)
  - **Mid-interview change #2:** "The primary matching node crashes mid-trading-day with orders in flight. Walk me through recovery to the *exact* book state, with zero lost or duplicated fills."

- **The crux (what they MUST nail):**
  **Determinism + single-writer + event-sourced recovery, with no DB in the hot path.** The matching core is a single-threaded deterministic state machine per symbol: same input sequence → identical output on every replica and every replay. Every inbound command is journaled to a replicated append-only log *before* matching; recovery = replay the log (+ periodic snapshot), not load-from-DB. Book lives in RAM (sorted price levels, FIFO queues, price-time priority). If they reach for Kafka + Postgres + "eventually consistent" — they've failed the crux; this is the *anti-pattern* problem.

- **Follow-up probes (escalating, medium → brutal):**
  1. Walk me through how a marketable limit order *walks* the book and partially fills. What's the data structure at each price level?
  2. Why single-threaded? Defend giving up multi-core parallelism on the hot path.
  3. Where does durability happen — before or after matching? Why does the ordering of journal-vs-match matter for exactly-once?
  4. Your replica needs to take over. How does it know the exact sequence number to resume from, and prove it has identical state?
  5. The journal is replicated across 3 nodes. A write is acked by 2, then the leader dies before the 3rd. Is that order live or not? Who decides?
  6. BOTEC me the orders/sec from *market structure*, not from "users." Then size the latency budget — where do the microseconds go?
  7. A GC pause hits mid-session. How is your design already immune, or how do you make it immune?
  8. Snapshotting the in-RAM book every N seconds: how do you snapshot a live mutating book without stalling matching or corrupting the replay boundary?
  9. Now make it cross-region for DR. What can you *not* preserve, and what do you tell the regulator about RPO/RTO?
  10. Two orders arrive in the same microsecond from two gateways. Define "fair" and make it deterministic and auditable.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Names determinism / order book / latency as the crux in the first minute.
  - ✅ Journal-before-match; recovery by replay; snapshot to bound replay time.
  - ✅ Partition by symbol, single-writer per symbol; ring buffer / pre-allocated, no locks, no GC churn.
  - ✅ Correct BOTEC from market structure; p99.9 mindset.
  - ❌ "Put orders on Kafka, write fills to Postgres, read book from DB" — DB in hot path.
  - ❌ Shards a single symbol's book across machines (kills price-time determinism).
  - ❌ Says "exactly-once delivery" without sequence numbers / idempotency.
  - ❌ Multi-threaded matching core with locks "for performance."

- **Common traps:**
  - Treating it like a CRUD web service. This is a deterministic state machine — bait them into the DB-in-hot-path mistake and watch.
  - Confusing *fault tolerance* (replicas in lock-step via the same log) with *load balancing* (sharding a symbol).
  - Forgetting that market data dissemination is a separate fan-out problem from matching.

---

### Large-scale zero-downtime migration

- **Opening prompt:**
  "We have a single 10 TB MySQL table — call it `orders` — serving live production traffic. We need to move it onto a horizontally sharded cluster. Zero downtime, no data loss, no double-processing. Design the migration and tell me how you roll back."

- **If they ask scope (curveballs to reveal):**
  - Traffic: ~50k reads/sec, ~5k writes/sec, 24/7, no maintenance window allowed.
  - Shard key: they must *choose* one and justify (e.g., `customer_id`); resharding later must be possible.
  - Consistency: reads must never see stale-then-fresh-then-stale flapping during cutover.
  - **Mid-interview change #1:** "During backfill you discover the source has rows that violate the new schema's constraints. Now what?"
  - **Mid-interview change #2:** "You're 80% cut over and error rates spike. Roll back — but writes have already been going to *both* systems for two hours. How do you roll back without losing those writes?"

- **The crux (what they MUST nail):**
  **The expand → dual-write → backfill → verify → cutover → contract pipeline, with a reversible cutover and a reconciliation/verify step that runs continuously.** Specifically: dual-write to old+new (with the old system as source of truth until cutover), backfill historical rows in batches *without* clobbering live dual-writes, run a continuous diff/verify job, flip reads gradually (shadow → percentage → full), and keep the old path warm for instant rollback. They MUST address write-ordering: a backfill of an old row must not overwrite a newer live write (use updated-at / version guards, or write-if-newer).

- **Follow-up probes (escalating, medium → brutal):**
  1. Dual-write: app-level, or CDC/binlog-based? Trade-offs of each for *ordering* and *atomicity*?
  2. A dual-write succeeds on old but fails on new. What's the state, and how do you not silently diverge?
  3. Backfill is running while live writes flow. A row is backfilled, then a live write updated it 10ms earlier than the backfill read it. How do you prevent the stale backfill from winning?
  4. How does your *verify* step work at 10 TB? Full row compare is too expensive — what do you actually do?
  5. Cutover: how do you flip reads so a single user never sees old→new→old? (Hint: per-shard or per-key cutover flags, not a global switch.)
  6. Rollback after 2 hours of dual-write: which system is now source of truth, and how do you re-converge them?
  7. You chose `customer_id` as shard key. A few mega-customers are 100× the average. Describe the hotspot and your fix without re-migrating.
  8. Schema-violating rows found mid-backfill (curveball): quarantine, transform, or fail the migration? Defend the blast radius of each.
  9. How do you make the *entire* migration restartable from any point after a coordinator crash?
  10. The CFO asks for a one-paragraph rollback guarantee. Write it. What's your actual RPO during cutover?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Names the full pipeline; treats cutover as gradual + reversible, not a flag flip.
  - ✅ Solves the backfill-vs-live-write race explicitly (version/timestamp guards, write-if-newer).
  - ✅ Continuous verify with checksums/sampling, not one big compare at the end.
  - ✅ Per-key/per-shard cutover; keeps old path warm; concrete rollback story.
  - ❌ "Stop writes, copy, point app at new DB" — that's downtime.
  - ❌ Backfill that blindly overwrites live data.
  - ❌ No verification step / "we'll trust the copy."
  - ❌ Treats rollback as "just flip back" while ignoring the writes that landed in both systems.

- **Common traps:**
  - Forgetting that dual-write is **not atomic** across two systems — the partial-failure case is the whole game.
  - Picking a shard key with no resharding story.
  - Verifying once at the end instead of continuously.

---

### Code deployment system

- **Opening prompt:**
  "Design the system that takes a built artifact and safely rolls it out to a global fleet of, say, 50,000 hosts. I care about how you ship safely, detect a bad release automatically, and roll back — including when the release has a DB migration in it."

- **If they ask scope (curveballs to reveal):**
  - Fleet: 50k hosts across 20 regions; services range from stateless web to stateful stores.
  - Cadence: hundreds of deploys/day; a bad deploy must be caught in *minutes*, not by a paged human.
  - **Mid-interview change #1:** "This release contains a non-backward-compatible DB schema migration. Now design rollback."
  - **Mid-interview change #2:** "A dependency you pulled into the build was compromised upstream. How would your system have caught it, or contained it?"

- **The crux (what they MUST nail):**
  **Progressive delivery with automated canary *analysis* + ring rollout + a rollback path that accounts for migrations.** Not "deploy to canary then prod" — but *automated* statistical comparison of canary vs baseline (error rate, latency, business metrics) with an automatic halt/rollback, expanding through rings (1 host → 1% → 1 region → global) with bake time per ring. For migrations: the migration must be **decoupled from code** and made backward/forward compatible (expand-contract), because you can't always roll a schema back. Supply chain: provenance/signing/SBOM and build isolation.

- **Follow-up probes (escalating, medium → brutal):**
  1. Define canary *analysis*, not canary deployment. What signals, what baseline, what statistical test, what threshold?
  2. Canary looks fine but the bug only manifests at 30% traffic (resource contention). How does ring rollout catch what canary missed?
  3. Code rolls back in seconds. The schema migration that shipped with it can't. Reconcile — how do you make *every* deploy rollback-safe?
  4. Walk the expand-contract migration pattern. At which step is it safe to deploy the new code? To roll back?
  5. Automated rollback fires on a false positive at 3am and reverts a *good* deploy. How do you bound that damage and not erode trust in the system?
  6. The artifact's provenance: how do you guarantee the thing you deploy is the thing you built from the commit you reviewed?
  7. A transitive dependency is malicious (curveball). Where in the pipeline do you detect/contain — build, sign, admit, runtime?
  8. Two services deploy simultaneously and are mutually incompatible mid-rollout. How does your ordering/dependency model prevent a broken intermediate state?
  9. How do you deploy to *stateful* services (a database fleet) differently than stateless web?
  10. Global rollback of a release that's 60% rolled out: is it faster to roll forward or back? How does the system decide?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Canary *analysis* (automated metric comparison + auto-halt), not just "a canary box."
  - ✅ Rings with bake time; blast-radius increases monotonically.
  - ✅ Expand-contract migrations decoupled from code deploy; explicit "what can't roll back."
  - ✅ Supply chain: signing, provenance/SBOM, build isolation, admission control.
  - ❌ "Deploy to canary, if it's fine deploy everywhere" with no automated analysis.
  - ❌ Assumes schema changes roll back like code.
  - ❌ No mention of detecting a bad deploy automatically (relies on humans/alerts).

- **Common traps:**
  - Conflating canary *deployment* (a box) with canary *analysis* (a decision system).
  - Ignoring that DB migrations are the part that *doesn't* roll back.
  - Forgetting the build/supply-chain side entirely.

---

### Real-time analytics / ad-click aggregation

- **Opening prompt:**
  "Design a system to aggregate ad clicks in real time — per-campaign click counts and unique-user counts, updated within seconds, used for both billing and a live dashboard. Scale: ~1M click events/sec at peak."

- **If they ask scope (curveballs to reveal):**
  - Billing path needs **correct counts** (you charge advertisers); dashboard tolerates approximation.
  - Unique users per campaign: hundreds of millions of distinct users — exact set is too big to hold.
  - Late/out-of-order events arrive (mobile, retries) up to hours late.
  - **Mid-interview change #1:** "Clients retry on timeout, so the same click can arrive 2–3 times. Your billing count must not double-charge."
  - **Mid-interview change #2:** "Product wants 'unique users per campaign' across 100M+ users in ~12 KB of memory with <1% error. Do it."

- **The crux (what they MUST nail):**
  **Exactly-once *effect* (idempotent counting via dedup keys), plus the right approximate sketches for cardinality (HLL) and frequency (CMS), with a clear lambda-vs-kappa stance.** They must distinguish *delivery* (at-least-once, unavoidable) from *effect* (each click counted once via an event-id dedup window). For uniques they must reach for **HyperLogLog** (~0.8% error at ~12 KB), for heavy-hitter frequency **Count-Min Sketch**. They must separate the **exact/billing** path (idempotent, reconcilable, source-of-truth) from the **approximate/dashboard** path.

- **Follow-up probes (escalating, medium → brutal):**
  1. The client says "I sent the click once." The network disagrees. How do you count it exactly once for billing?
  2. Where does the dedup key live, how big is the window, and what happens to a retry that arrives *after* the window expires?
  3. Exact distinct-count of 100M users costs how much memory? Now do it in 12 KB — what structure, what error, why is that error acceptable for the dashboard but not billing?
  4. HLL gives ~0.8% error. The advertiser disputes a bill computed from it. What do you actually bill on?
  5. Lambda vs kappa here — pick one and defend it for *this* problem. What does the batch layer reconcile that streaming can't?
  6. An event arrives 3 hours late. Which window does it land in, and does your billing total change retroactively?
  7. BOTEC the 1M events/sec into bytes/sec, partitions, and aggregator fan-out. (Watch for DAU-as-RPS.)
  8. Two aggregators each hold a partial HLL for the same campaign. How do you merge them? Why is that mergeability the reason you chose HLL?
  9. A stream processor crashes mid-window with un-checkpointed state. How do you not lose or double-count the in-flight aggregates?
  10. Now segment uniques by (campaign × country × hour). The cardinality of *combinations* explodes. How do you keep it bounded?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Exactly-once *effect* via idempotency/dedup, explicitly rejecting "exactly-once delivery."
  - ✅ HLL for uniques (cites ~0.8% / ~12 KB), CMS for frequency; explains *mergeability*.
  - ✅ Separates billing (exact, reconcilable) from dashboard (approximate).
  - ✅ Coherent lambda/kappa choice; handles late/out-of-order with watermarks.
  - ❌ Claims "exactly-once delivery."
  - ❌ Tries to store every user ID in a set for distinct counts.
  - ❌ Bills directly off an approximate sketch with no reconciliation.
  - ❌ Ignores retries/dedup entirely.

- **Common traps:**
  - "Exactly-once delivery" — the headline bait.
  - Using HLL for the billing number (it's approximate).
  - No late-arrival / watermark story.

---

### Distributed job scheduler

- **Opening prompt:**
  "Design a distributed job scheduler — think cron-as-a-service plus on-demand jobs. Users register schedules ('every 5 min', 'at 02:00 daily') and one-off jobs. At fire time, each job runs **exactly once**. Scale to millions of scheduled jobs."

- **If they ask scope (curveballs to reveal):**
  - "Exactly once" = the *effect* runs once even if workers crash mid-execution.
  - Jobs can be long-running (minutes) or instant; some are idempotent, some are not (assume not, then push them).
  - Clock skew across scheduler nodes is real.
  - **Mid-interview change #1:** "10,000 jobs are all scheduled for `0 0 * * *` (midnight). All fire at once. Design for that."
  - **Mid-interview change #2:** "A worker picks up a job, runs it, but dies before acking completion. Did it run? Re-run it or not?"

- **The crux (what they MUST nail):**
  **Exactly-once *execution* via leader-elected/owned scheduling + lease-based work claiming + idempotency, and handling cron thundering-herd.** A job fires once because a single owner (leader election / partitioned ownership) decides it's due; a worker *claims* it with a lease/fencing token so two workers can't both run it; and because crash-after-run is indistinguishable from crash-before-ack, true exactly-once requires either idempotent jobs or a transactional outbox/dedup on the effect. They must address the herd: jitter, sharded time wheels, or smoothing.

- **Follow-up probes (escalating, medium → brutal):**
  1. Two scheduler nodes both think a job is due at 02:00. How do you guarantee one fires it? What primitive?
  2. A worker claims a job, then its lease expires because it GC-paused (but it's still running). Another worker claims it. Now what — and what's a fencing token?
  3. Worker runs the job, then dies before acking (curveball). You can't tell "ran" from "didn't run." How do you achieve exactly-once *effect* anyway?
  4. The midnight herd: 10k jobs at `0 0 * * *`. What breaks, and how do you smooth it without violating the schedule contract?
  5. Leader election: what do you use, and what happens during the election window — are firings delayed, dropped, or duplicated?
  6. Clock skew: node A's clock is 3s ahead. Does the job fire early/twice/late? How do you make firing decisions clock-skew tolerant?
  7. A job takes 20 min but is scheduled every 5 min. Overlap policy? (skip / queue / kill-and-restart) — defend a default.
  8. Persisted schedule store: how do you find "all jobs due in the next tick" efficiently across millions of schedules?
  9. The whole scheduler tier restarts. How do you not miss firings that were due during the outage, and not stampede catch-up firings?
  10. Make "exactly once" honest: which part is at-most-once, which is at-least-once, and where does idempotency bridge them?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Leader election / partitioned ownership for *deciding* a job is due.
  - ✅ Lease + fencing token for *claiming* execution; explains the GC-pause split-brain.
  - ✅ Honest exactly-once: at-least-once delivery + idempotent/transactional effect.
  - ✅ Thundering-herd mitigation (jitter, time-wheel sharding).
  - ❌ "A lock guarantees exactly-once" with no fencing / no crash-after-run reasoning.
  - ❌ Polls a DB `WHERE next_run <= now()` with no concurrency control.
  - ❌ Ignores the herd; ignores clock skew.

- **Common traps:**
  - Believing a distributed lock alone gives exactly-once (it doesn't — fencing + idempotency do).
  - Ignoring the crash-after-run-before-ack ambiguity (the heart of the problem).
  - Forgetting catch-up after a scheduler outage stampedes.

---

### Live streaming

- **Opening prompt:**
  "Design a live video streaming platform. A creator goes live; millions of viewers watch with low latency. Cover ingest, transcode, and fan-out. Tell me the latency you're targeting and how that choice drives the architecture."

- **If they ask scope (curveballs to reveal):**
  - Two regimes: **interactive** (sub-second, e.g. auctions, betting, video calls) vs **broadcast** (a few seconds glass-to-glass, millions of viewers).
  - Scale: one mega-event = 5M+ concurrent viewers on a single stream.
  - Devices: everything — so adaptive bitrate is mandatory.
  - **Mid-interview change #1:** "Now it must be *interactive* — sub-second so the streamer can react to chat/bids. Does HLS still work?"
  - **Mid-interview change #2:** "5M viewers join the same stream in 30 seconds (event start). Your origin must not melt."

- **The crux (what they MUST nail):**
  **The latency target dictates the protocol — LL-HLS/DASH (segments, CDN-friendly, ~2–5s, scales to millions) vs WebRTC (sub-second, peer/SFU, harder to fan out) — plus real-time transcode (ABR ladder) and CDN/edge fan-out for the read side.** They must connect the *requirement* (latency) to the *protocol* and own the consequence: HLS scales via CDN caching of segments to millions but is seconds-latent; WebRTC is sub-second but fan-out to millions needs an SFU cascade/tree, not raw peer mesh.

- **Follow-up probes (escalating, medium → brutal):**
  1. Glass-to-glass latency budget: enumerate where the seconds go (capture, encode, segment, CDN, buffer, decode).
  2. Why does HLS get to millions of viewers "for free" but WebRTC doesn't? What's the fan-out unit in each?
  3. Make HLS lower latency — what does LL-HLS actually change (chunked transfer, partial segments) and what does it cost?
  4. The interactive pivot (curveball): sub-second. Justify WebRTC + SFU and describe the SFU cascade for 5M viewers.
  5. Real-time transcode: you need an ABR ladder (1080p/720p/480p/…). How do you transcode live without adding seconds, and how do you parallelize it?
  6. Thundering join: 5M viewers in 30s. Shield the origin — request coalescing, tiered CDN, segment caching. Walk the request path of viewer #4,000,000.
  7. A viewer's bandwidth drops mid-stream. How does ABR react, and where is that decision made — client or server?
  8. The streamer's upload connection blips for 2s. What do viewers see, and how do you mask it?
  9. DRM / token-authenticated playback at CDN scale without a round-trip to origin per segment.
  10. BOTEC the egress bandwidth for 5M viewers at 5 Mbps. Now tell me why this is a CDN/peering problem, not a server problem.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Derives protocol from the latency requirement; owns the HLS-vs-WebRTC fan-out trade-off.
  - ✅ CDN/edge caching of segments for broadcast scale; SFU cascade for interactive.
  - ✅ Live ABR transcode ladder; origin shielding / request coalescing for the join storm.
  - ✅ Correct egress BOTEC; frames it as peering/CDN, not origin compute.
  - ❌ "Use WebRTC" for a 5M-viewer broadcast without a fan-out tree.
  - ❌ One bitrate, no ABR.
  - ❌ All viewers hit the origin (no CDN tiering / coalescing).
  - ❌ Can't explain why latency target changes the protocol.

- **Common traps:**
  - Picking a protocol before asking the latency target.
  - WebRTC peer-mesh at broadcast scale (collapses).
  - Forgetting the origin-shield problem on the join storm.

---

### Feature-flag / experimentation platform

- **Opening prompt:**
  "Design a feature-flag and A/B experimentation platform. Engineers define flags and experiments; client and server SDKs evaluate them per request with minimal latency; a stats engine tells PMs whether a variant won. Scale: thousands of flags, billions of evaluations/day."

- **If they ask scope (curveballs to reveal):**
  - Flag evaluation must be **fast** (added to every request) and **available** even if the control plane is down.
  - A user must get a **consistent** bucket — same variant across devices/sessions/services for the life of the experiment.
  - Stats must be trustworthy enough for PMs to ship decisions on.
  - **Mid-interview change #1:** "The flag service is unreachable from the edge. What value does evaluation return, and who decides?"
  - **Mid-interview change #2:** "A PM peeks at results daily and ships when p<0.05 appears. Why is that wrong, and how does your stats engine prevent the false-positive blowup?"

- **The crux (what they MUST nail):**
  **Edge/local evaluation (config pushed to SDKs, evaluate locally — no network call per flag) + deterministic consistent bucketing (hash(user_id + experiment_salt) → bucket, stable and stateless) + a statistically honest stats engine (handles peeking / multiple comparisons).** They must make evaluation local-and-fail-safe (default values, last-known-good config), bucketing a pure function (no server lookup, reproducible across services), and the stats engine resistant to the peeking problem (sequential testing / fixed horizons / CUPED, not naive daily p-value checks).

- **Follow-up probes (escalating, medium → brutal):**
  1. Why evaluate at the edge/in-SDK instead of an RPC per flag? What gets pushed and how fresh is it?
  2. Same user, two services, two devices — all must get variant B. How, with no shared per-user state lookup?
  3. Your bucketing hash must be stable when you *add* a new variant later. How do you avoid re-shuffling everyone?
  4. Control plane is down (curveball). What does `isEnabled("x")` return? Who owns that default and where does the last-known-good config live?
  5. Gradual rollout 1%→100%: when you go 1%→2%, do the original 1% stay in? Why does monotonicity matter?
  6. The peeking problem (curveball): a PM checks daily and ships at first p<0.05. Quantify the false-positive inflation and fix it.
  7. Mutually-exclusive experiments overlapping on the same users — how do you prevent interaction effects polluting both?
  8. Sample-ratio mismatch: assignment says 50/50 but exposure logs show 48/52. What does that tell you and would you trust the result?
  9. A flag is evaluated billions of times/day. How do you log *exposure* (who saw what) without that logging becoming the bottleneck?
  10. Kill switch: a flag is causing an incident. How fast can you turn it off globally, and what's the propagation/consistency guarantee?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Local/edge evaluation with pushed config; fail-safe defaults / last-known-good.
  - ✅ Deterministic stateless bucketing via hashing; stable under variant changes.
  - ✅ Stats engine that addresses peeking / multiple comparisons / SRM.
  - ✅ Fast global kill switch with a stated propagation guarantee.
  - ❌ An RPC to a flag service on every evaluation (latency + availability bomb).
  - ❌ Random bucketing with no stable hash → inconsistent variants across services.
  - ❌ Naive daily p-value checks ("ship at p<0.05") — the peeking trap.
  - ❌ No fail-safe when the control plane is down.

- **Common traps:**
  - Centralized per-request evaluation (kills both latency and availability).
  - Non-deterministic bucketing (variant flaps across devices/services).
  - Statistically naive engine (peeking, SRM, multiple comparisons ignored).

---

### Collaborative whiteboard

- **Opening prompt:**
  "Design a real-time collaborative whiteboard — multiple users draw shapes, sticky notes, freehand, on an infinite canvas simultaneously, and all see a consistent result. Cover concurrency, scale, and what happens when two people edit the same object at once."

- **If they ask scope (curveballs to reveal):**
  - Concurrency: dozens of editors per board; conflicting edits to the same object must converge to *one* state everyone agrees on.
  - Offline: a user edits offline, reconnects — their changes must merge, not clobber.
  - Canvas is huge: tens of thousands of objects, viewport shows a window.
  - **Mid-interview change #1:** "Two users drag the same shape to two different positions simultaneously. What's the final position and is it the same on every screen?"
  - **Mid-interview change #2:** "A user goes offline for 10 minutes, keeps drawing, then reconnects. Merge their work with everyone else's without a manual conflict dialog."

- **The crux (what they MUST nail):**
  **Conflict resolution that gives Strong Eventual Consistency (SEC) — CRDTs (or OT) so concurrent edits converge deterministically without a central arbiter — and a clear stance on server-authoritative vs local-first.** They must articulate that all replicas applying the same set of operations (in any order) reach the *same* state (SEC), why that enables offline/local-first, and the trade-off vs a server-authoritative model (simpler, but no true offline and a SPOF on the write path). Plus spatial scaling: don't broadcast every op to every client for a 100k-object board.

- **Follow-up probes (escalating, medium → brutal):**
  1. Two users move the same shape simultaneously. Define the converged result and prove every client agrees. Last-writer-wins by what clock?
  2. CRDT vs OT — pick one for this and defend it. What does OT need that CRDTs don't (and vice versa)?
  3. Define Strong Eventual Consistency precisely. What property must your merge operation have (commutative/associative/idempotent)?
  4. Offline 10 min then reconnect (curveball): how do the offline ops merge? Can they resurrect a shape someone else deleted? (the add/remove tension)
  5. Server-authoritative vs local-first: which did you choose and what did you give up? When would you flip?
  6. 100k objects, 50 editors. You can't broadcast every keystroke to everyone. How do you scale the fan-out — viewport/interest-based subscriptions?
  7. Freehand drawing emits hundreds of points/sec. How do you avoid drowning the network and the CRDT in tiny ops?
  8. Undo/redo with multiple users — is undo global or per-user? How does that interact with CRDT history?
  9. Persistence: how do you snapshot/compact CRDT state so a new joiner doesn't replay the entire op history of a year-old board?
  10. A malicious client sends garbage ops. In a local-first model with no central arbiter, how do you keep one bad client from corrupting everyone?

- **Strong answer covers / Weak answer red flags:**
  - ✅ CRDT/OT for convergence; states SEC and the commutativity requirement.
  - ✅ Explicit server-authoritative vs local-first stance with the trade-off.
  - ✅ Handles offline merge and the add/remove (resurrection) tension.
  - ✅ Spatial/interest-based fan-out for large canvases; op compaction/snapshots.
  - ❌ "Last write wins on a wall clock" with no convergence guarantee (and clock-skew blind).
  - ❌ Central lock per object (kills concurrency and offline).
  - ❌ Broadcasts every op to every client regardless of viewport.
  - ❌ Can't define how concurrent edits converge.

- **Common traps:**
  - Naive LWW on unsynchronized clocks (diverges / loses edits silently).
  - Ignoring the offline/merge requirement that motivates CRDTs in the first place.
  - Forgetting fan-out scaling on big boards.

---

### IoT ingestion

- **Opening prompt:**
  "Design the ingestion pipeline for an IoT platform: tens of millions of devices, each sending telemetry every few seconds over MQTT. You must accept the firehose, store time-series data queryable for dashboards and alerts, and not fall over when devices misbehave."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~10M concurrent connected devices (C10M), ~5M messages/sec aggregate.
  - Connections are long-lived MQTT; devices are flaky (mobile, intermittent).
  - Queries: recent data (live dashboards) + downsampled history (trends).
  - **Mid-interview change #1:** "Each device now reports a new tag per firmware version, per region, per sensor. Your time-series cardinality explodes. What breaks?"
  - **Mid-interview change #2:** "A firmware bug makes 2M devices reconnect-storm every 10 seconds. Keep the gateway alive."

- **The crux (what they MUST nail):**
  **C10M connection handling at the MQTT gateway (event-driven, not thread-per-connection), a write-optimized ingestion path decoupled by a log/buffer, and a TSDB design that controls cardinality.** They must (1) terminate millions of long-lived connections with an async/event-loop gateway (epoll/kqueue, connection state cheap), (2) absorb the write firehose into a durable buffer before the TSDB so back-pressure doesn't kill ingest, and (3) recognize **time-series cardinality** (tags × devices × sensors) as the silent killer of TSDB cost/performance, and bound it.

- **Follow-up probes (escalating, medium → brutal):**
  1. 10M long-lived connections on the gateway: why is thread-per-connection dead here? What's the model and what's the per-connection memory budget?
  2. BOTEC the message firehose into bytes/sec and partitions. (Watch DAU-as-RPS: it's per-device-interval, not "users.")
  3. The TSDB can't keep up with peak writes. What sits between the gateway and the TSDB, and what's your back-pressure policy — drop, buffer, or shed?
  4. Cardinality explosion (curveball): tags per firmware × region × sensor. Why does this destroy a TSDB, and how do you bound series count?
  5. Reconnect storm: 2M devices reconnect every 10s (curveball). How do you stop the auth/session path from melting — jittered backoff, connection rate-limits, where enforced?
  6. A single device goes rogue and sends 10k msg/sec. How do you isolate one device without inspecting every message centrally?
  7. Hot vs cold storage and downsampling: raw at 1s, rolled up to 1m/1h. Where does rollup happen and how do you keep dashboards fast?
  8. Out-of-order / late telemetry from a device that was offline for an hour. Where does it land and does it disturb existing rollups?
  9. At-least-once MQTT QoS means duplicates. How do you dedup telemetry without a per-message round-trip?
  10. The gateway tier needs to scale and a node dies holding 1M connections. Where do those devices reconnect and how fast does the fleet recover?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Async/event-driven gateway for C10M; cheap per-connection state.
  - ✅ Durable buffer/log between ingest and TSDB; explicit back-pressure/shed policy.
  - ✅ Names time-series **cardinality** as the core risk and bounds it.
  - ✅ Reconnect-storm mitigation (jittered backoff, rate limits) and per-device isolation.
  - ❌ Thread-per-connection / "a load balancer handles it."
  - ❌ Writes straight to the TSDB with no buffer (firehose kills it).
  - ❌ Unbounded tags → cardinality explosion unrecognized.
  - ❌ DAU-as-RPS BOTEC.

- **Common traps:**
  - Treating MQTT like HTTP (short-lived) — it's millions of *persistent* connections.
  - Ignoring cardinality (the #1 TSDB failure mode in production).
  - No reconnect-storm / back-pressure story.

---

### Online judge (LeetCode)

- **Opening prompt:** "Design LeetCode. Users submit code in many languages, get a verdict against hidden tests, and compete in timed contests with a live leaderboard."

- **If they ask scope (curveballs to reveal):**
  - ~60 submissions/s normally; contests with 100k participants peak at ~1–2k submits/s in the last 10 minutes.
  - Code is untrusted and anonymous.
  - The same code must get the same verdict every time.
  - **Mid-interview change #1:** "A user's submission tries to open a socket and read /etc/passwd." (Threat model.)
  - **Mid-interview change #2:** "The test data for a contest problem was wrong; 30k submissions need rejudging." (Rejudge pipeline.)

- **The crux (what they MUST nail):**
  1. **Isolation as the threat model:** fresh sandbox per job (microVM/gVisor), no network, cgroup limits, seccomp, read-only FS; each control mapped to an attack.
  2. **Async execution:** 202 + ID → priority lanes → workers → verdict via SSE/poll.
  3. **Fairness and the spike:** CPU-time limits on pinned cores; calendar-driven pre-scaling; API-timestamped penalties.

- **Follow-up probes (escalating):**
  1. Why async? What does the client see?
  2. BOTEC: cores needed at the contest peak.
  3. Docker container per submission: is that safe enough? Why or why not?
  4. Name five attacks and the control that stops each.
  5. Two runs of the same code: one Accepted, one TLE. Why, and fix it.
  6. "Run sample tests" floods the queue during the contest. Protect submits.
  7. Leaderboard data model with penalties and ties.
  8. **Brutal:** a kernel exploit escapes the sandbox. Contain the blast radius.
  9. **Brutal:** rejudge 30k submissions mid-contest without stalling live judging.
  10. **Brutal:** the queue backs up 5 minutes at the deadline. Does that change standings?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Hypervisor-level isolation from warm pools; disposable sandboxes.
  - ✅ Lanes with reserved contest capacity; pre-scale on the calendar.
  - ✅ CPU time, not wall time; borderline re-runs.
  - ❌ Long-lived containers "cleaned" between jobs.
  - ❌ Synchronous execution in the API request.
  - ❌ Reactive autoscaling for a 10-minute cliff.

- **Common traps:** test cases mounted where user code can read them; no output-size cap; leaderboard computed by `ORDER BY` at read time.
- **Reference design:** [46 — Online Judge](46-design-online-judge.md).

---

### Online chess (Chess.com / Lichess)

- **Opening prompt:** "Design online chess. Players are matched with an opponent of similar rating, play a timed game in real time, and ratings update afterwards. Spectators can watch."

- **If they ask scope (curveballs to reveal):**
  - ~10M games/day, ~500k concurrent games; bullet games (1 minute per side) are popular.
  - A server failure must not end or corrupt a game.
  - Players are global; latency varies 20–300 ms.
  - **Mid-interview change #1:** "A game server crashes with 50k games in progress." (Move log, leased ownership, replay, frozen clocks.)
  - **Mid-interview change #2:** "Players in India complain they lose bullet games on time to players in Europe." (Lag compensation.)

- **The crux (what they MUST nail):**
  1. **Server authority with one owner per game:** validate moves, keep both clocks, append each move durably before acknowledging.
  2. **Fair clocks:** server-measured elapsed time with bounded lag compensation; server-side flag timers.
  3. **Matchmaking:** per-time-control rating pools, widening window, atomic pair-claim.

- **Follow-up probes (escalating):**
  1. Message rate BOTEC: does throughput matter here?
  2. Who validates moves, and why not the client?
  3. Where does game state live, and what happens if that server dies?
  4. How is time measured? Can a client lie about its clock?
  5. Lag compensation without letting cheaters claim infinite lag.
  6. Matchmaking: how do you pair quickly and fairly? Prevent double-pairing.
  7. Rating updates: what if the game-over event is delivered twice?
  8. **Brutal:** failover without anyone losing on time.
  9. **Brutal:** a zombie server keeps accepting moves after failover. Stop it.
  10. **Brutal:** detect engine-assisted cheating without slowing the move path.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Move log before ack; leased, fenced ownership; replay on failover.
  - ✅ Server clocks with capped lag credit.
  - ✅ Idempotent rating transaction keyed by game ID.
  - ❌ Client-reported remaining time.
  - ❌ Game state only in memory.
  - ❌ Copies shooter netcode (prediction, interpolation) for a turn-based game.

- **Common traps:** clocks running during a server outage; double-pairing; anti-cheat on the synchronous path.
- **Reference design:** [52 — Online Chess](52-design-online-chess.md).

---

### ChatGPT-style LLM chat app

- **Opening prompt:** "Design ChatGPT. Users have conversations with an AI model, responses stream token by token, conversations are saved, and millions of users share a limited GPU fleet."

- **If they ask scope (curveballs to reveal):**
  - ~30M DAU, ~15 messages each; ~500 output and ~2,000 input tokens per request (state these as assumptions).
  - Free and paid tiers; paid must be protected under load.
  - Time to first token target ~1 s.
  - **Mid-interview change #1:** "A launch triples demand for a week and you can't add GPUs." (Admission, degradation ladder.)
  - **Mid-interview change #2:** "Users on trains complain responses vanish when the connection drops." (Resumable streams.)

- **The crux (what they MUST nail):**
  1. **GPU tokens/s as the budget:** size the fleet in output tokens per second; separate prefill from decode.
  2. **Serving efficiency:** continuous batching, paged KV cache, prefix caching with conversation affinity.
  3. **Fair admission and degradation:** token-bucket limits, priority with deadlines, free tier degraded first; resumable SSE.

- **Follow-up probes (escalating):**
  1. BOTEC in tokens: output tokens/s at peak and GPU count (with stated assumptions).
  2. What determines time to first token vs streaming speed?
  3. Why does batching matter so much, and what's continuous batching?
  4. Long conversations resend the whole history each turn. Make that cheap.
  5. How does the router pick a replica?
  6. Demand exceeds capacity: walk the degradation ladder.
  7. Regenerate and edit: model the conversation storage.
  8. **Brutal:** a client drops mid-stream. Resume without regenerating.
  9. **Brutal:** users click stop on long generations. Why does it matter, and how is it implemented?
  10. **Brutal:** a new model version is cheaper but slightly worse. How do you roll it out and decide?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Sizes in tokens, not requests; names prefill vs decode.
  - ✅ Prefix-cache-affine routing; continuous batching.
  - ✅ Token-bucket admission with deadlines and a written degradation order; resumable streams via a token buffer.
  - ❌ "Autoscale the GPUs."
  - ❌ Round-robin routing; QPS-based limits.
  - ❌ Streams tied to the client connection with no resume.

- **Common traps:** ignoring the context window; treating conversations as flat lists; forgetting cancellation frees GPU.
- **Reference design:** [53 — ChatGPT-style LLM Chat](53-design-chatgpt.md).

---

## Google "Build From Scratch" drills

> Google penalizes "just use DynamoDB / ZooKeeper / Kafka." These drills force the candidate to **build the primitive**. Open with the prompt; if they name a product, say: *"Pretend that product doesn't exist — build the mechanism."* Each has a crux + 4–5 probes. Pair with [21 Build From Scratch](21-build-from-scratch.md) and [26 Google SD Playbook](26-google-sd-playbook.md).

### Distributed key-value store

- **Prompt:** "Build a distributed, replicated KV store from scratch — gets and puts, scaled across many machines, surviving node failure. No managed database."
- **Crux:** Partitioning (consistent hashing to bound reshuffle), replication (N replicas), and a **clear consistency stance** (quorum R+W>N for strong-ish, or leaderless eventual with conflict resolution) — and how nodes find data (ring/membership). They must own a CAP position, not hand-wave it.
- **Probes:**
  1. How do keys map to nodes, and what moves when you add/remove a node? Why consistent hashing (+ virtual nodes)?
  2. A put with N=3: how many replicas must ack before you return success, and what does that buy you (R+W>N)?
  3. Two concurrent puts to the same key on different replicas — how do you resolve? (LWW vs vector clocks vs version vectors.)
  4. A node was down and missed writes. How does it catch up — read-repair, hinted handoff, anti-entropy/Merkle trees?
  5. Where exactly are you on the CAP triangle during a partition, and what does a client see?

### Distributed lock / lease service

- **Prompt:** "Build a distributed lock (or lease) service so that, across many processes, only one holds a given lock at a time. No ZooKeeper/etcd/Redis."
- **Crux:** Leases (time-bounded, not eternal locks) + **fencing tokens** to survive the holder pausing (GC/network) past its lease, backed by a consensus/replicated store so the lock state itself is fault-tolerant. The fencing token is the make-or-break: a lock alone does NOT give mutual exclusion under pauses.
- **Probes:**
  1. Why a *lease* (with TTL) rather than a lock held until release? What happens if the holder crashes holding a plain lock?
  2. The holder GC-pauses 30s, its lease expires, another process grabs the lock — now both think they hold it. Fix it. (Fencing tokens.)
  3. Where does the lock *state* live so it survives the lock-service node dying? Why does this need consensus?
  4. Clock skew: leases are time-based. Whose clock, and how do you tolerate skew safely?
  5. Prove that even with arbitrary process pauses, the protected resource only ever accepts one writer at a time.

### Unique ID generator (Snowflake)

- **Prompt:** "Build a service that generates unique, roughly time-sortable 64-bit IDs across many machines, no coordination per ID. No UUID library, no DB auto-increment."
- **Crux:** The **Snowflake bit layout** and why each field exists. Correct 64-bit layout:

  | Bits | Field | Purpose |
  |---|---|---|
  | 1 | sign | always 0 (keep IDs positive) |
  | 41 | timestamp (ms) | time-sortable; ~69 years from a custom epoch |
  | 10 | machine id | up to 1024 nodes, no coordination per-ID |
  | 12 | sequence | up to 4096 IDs per node per ms |

  **1 + 41 + 10 + 12 = 64 bits.** They must justify roughly-sortable (timestamp high bits) and per-node-local generation (no coordination on the hot path).
- **Probes:**
  1. Lay out the 64 bits and justify each field's width. How many IDs/ms/node and how many nodes? (4096/ms/node, 1024 nodes.)
  2. The clock moves *backwards* (NTP correction). You could now mint a duplicate or out-of-order ID. What do you do — wait, refuse, or borrow sequence?
  3. A node exhausts all 4096 sequence numbers within one millisecond. What happens next?
  4. How is a machine id assigned without a central coordinator allocating per-ID? What if two nodes boot with the same id?
  5. Trade roughly-sortable for strictly-monotonic — what do you give up, and is it worth it?

### Sharded counter / HyperLogLog

- **Prompt:** "Build (a) a counter that handles a million increments/sec to one logical count, and (b) a way to count *distinct* items across hundreds of millions with tiny memory. No managed analytics product."
- **Crux:** For the hot counter — **shard the counter** (N sub-counters incremented independently, summed on read) to avoid single-row contention. For distinct count — **HyperLogLog**: ~0.8% standard error at ~12 KB for hundreds of millions of distinct items, and crucially it's **mergeable** (union sketches across shards). They must not propose a giant set/bitmap for cardinality.
- **Probes:**
  1. A single counter row is a write hotspot at 1M/s. How does sharding it fix that, and what's the read-side cost?
  2. Exact distinct count of 500M items costs how much memory? Now do it in ~12 KB. What's the error, and why is HLL's error acceptable here?
  3. Why is HLL **mergeable**, and why does that property matter for a sharded/distributed pipeline?
  4. When is HLL the wrong tool — when must you keep exact distincts, and what does that cost?
  5. Combine both: distinct users per key across shards. How do per-shard HLLs roll up to a global distinct count?

### Message queue from a log

- **Prompt:** "Build a durable message queue / pub-sub from scratch on top of an append-only log. Producers write, consumers read, messages survive a broker crash. No Kafka/SQS/RabbitMQ."
- **Crux:** An **append-only partitioned log** with per-consumer **offsets** (the consumer tracks position; the broker doesn't track per-message acks), replication for durability, and the honest delivery story: **at-least-once by default; exactly-once *effect* only via idempotent consumers / dedup**, never exactly-once delivery.
- **Probes:**
  1. Why an append-only log instead of per-message delete-on-ack? What does offset-based consumption buy you (replay, multiple consumers)?
  2. How do you get parallelism *and* ordering? (Partitioning + ordering only within a partition — name the trade-off.)
  3. A consumer processes a message, then crashes before committing its offset. On restart it re-reads it. At-least-once, at-most-once, or exactly-once — and how do you make the *effect* exactly-once?
  4. The broker dies. How is the log not lost, and from what offset does a recovered consumer resume? (Replication + committed offset.)
  5. A slow consumer falls behind; the log fills disk. Retention policy and back-pressure — what gives?

---

## Scoring

Grade every problem in this bank against the **shared scoring rubric in [40a](40a-mock-chat-media.md)** (signal dimensions, level calibration, and the "did they detect-defend-recover from failure" overlay). For the underlying playbooks these mocks test, cross-link:

- [21 Build From Scratch](21-build-from-scratch.md) — the primitives behind the Google drills.
- [26 Google SD Playbook](26-google-sd-playbook.md) — the five graded attributes and the 45-minute structure.
- [28 Designing for Failure](28-designing-for-failure.md) — the detect / defend / recover discipline you should hear in every answer above.
