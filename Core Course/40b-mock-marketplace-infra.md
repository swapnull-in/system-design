# Mock Interview Bank — Marketplaces, Money & Infra Primitives

> **Interviewer-side companion.** This is *not* a design playbook — the full reference designs live in **[18c Marketplaces & Money](18c-design-marketplace-money.md)** and **[18d Infra Primitives](18d-design-infra-primitives.md)**. This file is what you hold while *running* the interview: the opening prompt, the curveballs you withhold, the escalating probes, and the rubric that separates a Staff/Principal hire from a senior who memorized a blog post.
>
> **Calibration target:** the candidate is interviewing at **Staff / Principal / Google EM** level. At this level, drawing the boxes is table stakes. You are listening for: where they put the **strong/eventual consistency boundary**, whether they reach for a **saga over 2PC**, whether they catch their *own* **dual-write-without-atomicity** bug, and whether their **BOTEC** is reasoned (peak factor, read/write split, working-set sizing) rather than "DAU = RPS."

## How to use this bank

1. **40-minute clock per problem.** ~5 min scope/BOTEC, ~20 min design, ~12 min probes, ~3 min wrap. Say the budget out loud so they self-pace; a candidate who burns 20 min on the API contract is self-selecting out.
2. **Withhold the curveballs.** Do *not* volunteer scope. If they don't ask "read-heavy or write-heavy?", "how stale can X be?", "single region or global?" — that silence is signal. Reveal constraints from the *If they ask scope* block only when earned.
3. **Drop the mid-interview constraint change** once their happy path is stable (each problem lists 1–2). This is the highest-signal moment: you're testing whether the design *bends* or *shatters*.
4. **Escalate the probes — medium → brutal.** Stop climbing when they top out; note the rung. Most seniors stall around probe 5–6; Staff clears 7–8; Principal volunteers the answer to probe 9 before you ask.
5. **Score against the shared rubric** (see end). Write the failure mode, not just a number — "passed but never said the word *idempotent* unprompted" is more useful to the panel than "3/5".

> **Interviewer mantra:** reward the candidate who *names the tradeoff and picks a side*. Punish the one who lists three options and asks you which to use.

## Table of Contents

- [Ride-hailing (Uber/Lyft)](#ride-hailing-uberlyft)
- [Food delivery (DoorDash/Swiggy)](#food-delivery-doordashswiggy)
- [Hotel / flight booking](#hotel--flight-booking)
- [E-commerce platform](#e-commerce-platform)
- [Payment system](#payment-system) ← *the consistency-critical one*
- [Ad serving / RTB](#ad-serving--rtb)
- [Recommendation engine](#recommendation-engine)
- [URL shortener](#url-shortener)
- [Rate limiter](#rate-limiter)
- [Notification system](#notification-system)
- [Distributed cache](#distributed-cache)
- [API gateway](#api-gateway)
- [Web crawler](#web-crawler)
- [Metrics / monitoring platform](#metrics--monitoring-platform)
- [Yelp (local search & reviews)](#yelp-local-search--reviews)
- [Online auction (eBay)](#online-auction-ebay)
- [Price tracking service (camelcamelcamel)](#price-tracking-service-camelcamelcamel)
- [Scoring rubric](#scoring--cross-links)

---

### Ride-hailing (Uber/Lyft)

- **Opening prompt:** "Design the backend that matches riders to nearby drivers in real time. Riders request a ride; nearby drivers get offered it; one driver is assigned. Drivers stream location continuously. Take me from a tap to an assigned driver."

- **If they ask scope (curveballs to reveal):**
  - Scale: ~10M drivers, ~3M concurrent, each driver pings location every **4s** → ~750K location writes/sec at peak. Ride requests are *tiny* by comparison (~thousands/sec). Make them notice the writes are dominated by telemetry, not matching.
  - City-partitioned; a ride never crosses cities. Latency target: match within a few seconds.
  - **Mid-interview change #1:** "It's New Year's Eve in the downtown cell — 50× normal demand, drivers are scarce. What happens?" (forces surge + hot-cell contention.)
  - **Mid-interview change #2:** "Two riders' searches return the *same* driver as the best candidate at the same instant. Walk me through who gets the driver and how you guarantee it's exactly one."

- **The crux (what they MUST nail):**
  1. **Build the geospatial index** — geohash / S2 cells / QuadTree, a `cell → set<driver>` inverted map, and **neighbor-cell fan-out** for radius search (the driver may be just across a cell boundary). "Use Redis GEO / PostGIS" is an acceptable *prototype* answer but they must explain what's underneath.
  2. **Separate the firehose from the matcher** — high-frequency location updates go to an in-memory geo-index (eventual is fine; a driver's dot being 4s stale is harmless). Matching/assignment is a *separate, strongly-consistent* step.
  3. **Single-assignment guarantee** — candidate generation can be sloppy/parallel, but the final pairing of one rider ↔ one driver must serialize so no driver is double-dispatched (conditional update / compare-and-set on driver state, or per-driver lock).

- **Follow-up probes (escalating):**
  1. How do you find drivers near a point when the nearest one is in an adjacent cell? (neighbor fan-out / variable precision)
  2. Geohash vs S2 vs QuadTree — pick one and defend it. (S2's hierarchical cells + even coverage near poles; geohash's boundary distortion.)
  3. 750K location writes/sec — where do they land? Do they hit the durable DB? (No — in-memory ring/geo-index; durable only for trip records.)
  4. A driver pings every 4s. Do you persist every ping? What's the read pattern that justifies your storage choice?
  5. Two concurrent searches pick the same driver. Show me the exact mechanism that makes assignment exactly-once. (Now we're at the crux — push hard.)
  6. Surge: how is the multiplier computed, where does it live, and is it strongly or eventually consistent? (supply/demand ratio per cell; eventual is fine — but the *price quoted to the rider must be locked* for the trip.)
  7. The matched driver declines. How do you re-offer without leaking the rider's request or re-charging surge?
  8. Hot cell on NYE: the single downtown cell is a contention point. How do you shard *within* a cell without breaking radius search?
  9. Driver's phone loses signal mid-trip. How do you distinguish "offline" from "in a tunnel," and what's the timeout before you reassign?
  10. **Brutal:** Region fails over. The geo-index is in-memory and gone. What's your RTO, and do in-flight trips survive? (Trip state must be durable; geo-index is rebuildable from the next round of pings.)

- **Strong answer covers / Weak answer red flags:**
  - ✅ Names the write firehose explicitly and keeps it *off* the durable path; geo-index is in-memory and rebuildable.
  - ✅ Candidate-gen is eventual/parallel; assignment is a single serialized CAS — states the boundary unprompted.
  - ✅ Surge is per-cell supply/demand, eventual to compute but *locked at quote time*.
  - ✅ Treats neighbor fan-out as obvious, not a surprise.
  - ❌ Routes every location ping through Postgres "for durability."
  - ❌ "Redis GEO handles it" with no idea what `GEORADIUS` does internally.
  - ❌ Double-dispatch hand-waved as "the queue handles ordering."
  - ❌ BOTEC: multiplies DAU by 1 to get RPS, ignores the 4s telemetry firehose entirely.

- **Common traps:**
  - Conflating the location-update path (write-heavy, eventual) with the matching path (consistency-critical) — they have *opposite* requirements.
  - Forgetting boundary cells → matching a "nearby" driver that's actually 3 cells away on the diagonal.
  - Letting surge be a global number instead of per-cell; or recomputing it on every request instead of on a rolling window.

---

### Food delivery (DoorDash/Swiggy)

- **Opening prompt:** "Design the system that takes a food order, gets the restaurant to accept it, charges the customer, and assigns a courier to pick up and deliver. Three parties — customer, restaurant, courier — and they fail independently."

- **If they ask scope (curveballs to reveal):**
  - Scale: millions of orders/day, bursty at lunch/dinner. Courier pool is geo-distributed. Delivery SLA ~30–45 min.
  - Couriers can **batch** (carry 2–3 orders going the same direction).
  - **Mid-interview change #1:** "Payment succeeds, then the restaurant rejects the order (out of an ingredient). Unwind it." (saga compensation.)
  - **Mid-interview change #2:** "Dinner rush — orders spike 10× and couriers are saturated. How does assignment degrade gracefully rather than thrashing?"

- **The crux (what they MUST nail):**
  1. **The order saga** — restaurant-accept → payment → courier-assign, each step compensatable (release courier, void/refund charge, cancel order). **Saga, not 2PC** — these are independent services that can each be down.
  2. **The dispatch/assignment optimizer** — courier-to-order matching under an SLA, with **batching**; this is an optimization problem (minimize total delivery time / courier idle), not a simple nearest-match.
  3. **The two-sided state machine** — order state ↔ courier state, kept consistent as either side transitions or fails.

- **Follow-up probes (escalating):**
  1. Walk the happy-path saga. Now kill payment after restaurant-accept — what compensates?
  2. Idempotency: the courier-assign call times out and you retry. How do you avoid assigning two couriers?
  3. Batching: how do you decide two orders share a courier? What's the optimization objective?
  4. A courier accepts then goes offline. How long until reassignment, and who eats the SLA?
  5. The assignment optimizer can't run synchronously per order at peak. Batch window vs streaming — pick one and justify the latency hit.
  6. Restaurant marks "ready" but no courier is there. Where does that order wait, and how do you prevent food sitting cold from blocking the queue?
  7. Surge/dynamic pricing for couriers — strong or eventual? Where's it quoted/locked?
  8. The saga orchestrator itself crashes mid-flight. How do you resume? (durable saga log + idempotent steps.)
  9. **Brutal:** A bug double-charges 0.1% of orders during a deploy. How do you *detect* it (not prevent — detect after the fact)? (reconciliation against the ledger / payment provider.)
  10. **Brutal:** Restaurant's POS integration lies — says "accepted" then never makes the food. How does the system notice and recover?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Saga with explicit compensations per step; names the orchestrator's durable state.
  - ✅ Idempotency key on assignment and payment, claimed *before* side effects.
  - ✅ Treats dispatch as an optimization with a batch window at peak, not per-order greedy.
  - ✅ Has a *detection* (reconciliation) answer, not just prevention.
  - ❌ "Exactly-once via a queue" — there is no exactly-once delivery; it's at-least-once + idempotency. Catch this every time.
  - ❌ 2PC across restaurant/payment/courier services.
  - ❌ Dual-write: writes order state to DB *and* publishes to the courier queue with no outbox/transaction — silent divergence on crash.

- **Common traps:**
  - "Exactly-once via Kafka." Press: *what happens on consumer crash between commit and ack?* The honest answer is at-least-once + dedupe.
  - Synchronous per-order optimization that melts at dinner rush.
  - Forgetting the saga orchestrator's *own* durability — if it's stateless in memory, a crash strands half-done orders.

---

### Hotel / flight booking

- **Opening prompt:** "Design hotel (or flight seat) booking. Users search availability, then book. The hard part is the *last* room — two people click 'book' on the only room left at the same millisecond. Exactly one wins."

- **If they ask scope (curveballs to reveal):**
  - Search is read-heavy (huge fan-out across dates/properties); booking is comparatively rare but consistency-critical.
  - Inventory is finite and *cannot oversell* (unlike e-commerce, you can't backorder a hotel night).
  - **Mid-interview change #1:** "User holds a room for 10 min in checkout but never pays. The room must come back." (hold-with-TTL + sweeper.)
  - **Mid-interview change #2:** "A travel agency books 200 rooms across 50 hotels in one atomic request — all or nothing." (distributed reservation / saga across inventories.)

- **The crux (what they MUST nail):**
  1. **The inventory-reservation primitive** — how the last room is held without oversell: **atomic decrement**, **hold-with-TTL**, and an **expiry sweeper** that returns leaked holds.
  2. **Strong consistency on the inventory row** — the decrement must be serializable; this is non-negotiable for finite inventory.
  3. **Separate search (eventual, cached, read-replica) from booking (strong)** — and *reserve at checkout, not at search*.

- **Follow-up probes (escalating):**
  1. Show me the exact decrement that prevents oversell. (`UPDATE ... SET avail = avail - 1 WHERE avail > 0` conditional, or SELECT FOR UPDATE — make them say which and why.)
  2. Hold-with-TTL: where does the hold live, and what releases it if the user vanishes? (sweeper / TTL; idempotent re-claim.)
  3. Search shows "1 room left" but it's stale and already booked. Is that a bug? (No — search is eventual; the *booking* decrement is the source of truth. Make them say this.)
  4. The 10-min hold expires *while* the payment webhook is in flight. Who wins, and how do you avoid charging for a released room?
  5. Multi-room atomic booking across 5 hotels — 2PC or saga? Defend it. (saga with compensations; 2PC blocks and doesn't scale.)
  6. A single popular hotel on a holiday weekend is a hot row. How do you reduce contention without overselling? (careful — *bucketing risks oversell here*; discuss why the seat/room case is stricter than e-commerce SKUs.)
  7. Flight seats: seat *selection* vs seat *count*. Does your primitive change? (yes — specific seat is a unique resource, not a counter.)
  8. Reconciliation: your inventory count and the GDS/provider's drift apart. How do you detect and resolve?
  9. **Brutal:** The reservation DB is the bottleneck on a flash sale (concert hotel block). Walk me through it without overselling a single room.
  10. **Brutal:** A network partition splits your booking service from the inventory store mid-transaction. CP or AP here, and what does the user see?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Conditional atomic decrement; states the isolation level.
  - ✅ Hold-with-TTL + sweeper, idempotent.
  - ✅ Search/eventual vs book/strong boundary, reserve-at-checkout.
  - ✅ Saga (not 2PC) for multi-inventory atomic bookings.
  - ❌ "Lock the table" / coarse locking that kills search.
  - ❌ Reserving inventory at *search* or *add-to-cart* → phantom depletion.
  - ❌ Optimistic "retry on conflict" with no bound → livelock on the hot room, or worse, oversell under a sloppy retry.

- **Common traps:**
  - Importing e-commerce *bucketing* into hotel/flight without noticing finite-resource strictness (you can't backorder a seat).
  - Letting the TTL hold and the payment webhook race with no idempotent resolution.
  - Treating "specific seat 14C" like a fungible counter.

---

### E-commerce platform

- **Opening prompt:** "Design an e-commerce platform — browse catalog, add to cart, checkout, place order. Then I want to talk about a flash sale: 100K people buying the same 1,000-unit SKU in 30 seconds."

- **If they ask scope (curveballs to reveal):**
  - Read:write is *enormously* read-skewed (catalog/search browsing ≫ orders). Make them split it.
  - Flash sale is the real interview — most of the budget goes there.
  - **Mid-interview change #1:** "The flash-sale SKU has 1,000 units and a single inventory row. 100K concurrent buyers. The row is on fire." (hot-key contention.)
  - **Mid-interview change #2:** "We oversold by 12 units during the last sale. How did that happen and how do you make it impossible?"

- **The crux (what they MUST nail):**
  1. **Draw the strong/eventual boundary** — inventory + payment **strong**; catalog/cart/search/recs **eventual**. State it explicitly.
  2. **Reserve at checkout, not add-to-cart** — carts are abandoned constantly; reserving on add phantom-depletes stock.
  3. **The checkout saga with idempotency keys** — prevent double-charge / double-order; claim the key before side effects.
  4. **Flash-sale hot key** — recognize single-row serialization as the bottleneck; apply **inventory bucketing** (split 1,000 units into N buckets) **+ an in-memory atomic admission gate**, while keeping the durable DB as the **reconciled source of truth**. *This decouple move is the Staff signal.*

- **Follow-up probes (escalating):**
  1. Where do catalog reads come from at scale? (CDN/cache/read replicas — eventual, fine.)
  2. Add-to-cart: do you reserve stock? Why not? (abandonment → phantom depletion.)
  3. Checkout saga: enumerate the steps and the compensations.
  4. Idempotency key — what is it keyed on, and *when* is it claimed relative to the charge?
  5. Flash sale: the SKU is one row. 100K buyers. What breaks first? (row-level serialization throughput.)
  6. Now fix it without overselling. (bucketing + in-memory admission gate + DB as reconciled truth — push until they name all three.)
  7. With bucketing, one bucket empties while others have stock. How does a buyer find a non-empty bucket without scanning all of them?
  8. The in-memory gate and the durable DB disagree after a crash. Which wins, and how do you reconcile? (DB is truth; gate is rebuilt/reconciled.)
  9. **Brutal:** Explain *exactly* how the 12-unit oversell happened. (almost always: read-modify-write without atomicity, or gate trusted as truth without reconciliation, or a retry that re-decremented.)
  10. **Brutal:** Payment provider times out during the sale — you don't know if the customer was charged. What does the order show, and how do you converge? (idempotent re-query / reconciliation.)

- **Strong answer covers / Weak answer red flags:**
  - ✅ Explicit consistency boundary; reserve-at-checkout.
  - ✅ Recognizes hot-row serialization and applies bucketing + in-memory gate + DB-as-truth.
  - ✅ Idempotency key claimed before side effects; saga compensations.
  - ❌ "Just use a DB transaction" for the flash sale (the junior tell).
  - ❌ Reserves on add-to-cart.
  - ❌ Trusts the in-memory gate as the source of truth with no reconciliation → exactly the oversell bug.
  - ❌ BOTEC treats browse RPS and order RPS as the same number.

- **Common traps:**
  - In-memory admission gate as *truth* instead of *fast-path with DB reconciliation* — this is how the oversell happens.
  - Eventual catalog reads mistaken for an inventory-consistency problem (they're not — only the decrement is strong).
  - Bucketing without a story for finding a non-empty bucket.

---

### Payment system

> **The consistency-critical one. This is the hill.** If they relax consistency *anywhere* authoritative, they fail the problem. Volume is *low* (~1K TPS) — so throughput is never the excuse; correctness under failure is the entire game.

- **Opening prompt:** "Design a payment / wallet system: charge cards, hold balances, transfer between wallets, pay out. You orchestrate external providers (Stripe-like) that fail, time out, and lie about their state — while keeping an internally consistent ledger. Start by telling me your consistency stance."

- **If they ask scope (curveballs to reveal):**
  - Volume is *small* — ~1K transactions/sec. You can afford full ACID; the challenge is correctness, not throughput. Make them notice this and stop reaching for eventual-consistency tricks.
  - You call Stripe as a **gateway**; you do **not** outsource the ledger to it.
  - **Mid-interview change #1:** "You call Stripe to charge a card. The request times out. You don't know if the customer was charged. What does your system do?" (the single hardest case — idempotent re-query + reconciliation.)
  - **Mid-interview change #2:** "A retry storm replays the same charge 5 times. The customer must be charged exactly once. Show me the mechanism."

- **The crux (what they MUST nail):**
  1. **Double-entry ledger** — append-only, immutable entries; every transaction is ≥1 debit + ≥1 credit that **sum to zero**; balance is **derived (sum of entries), never mutated in place**. This is the heart of the answer.
  2. **Idempotency** — a unique idempotency key **claimed before any side effect**; replays return the *stored result*, never re-execute. No double-charge.
  3. **Saga, not 2PC** — the external provider can't join your transaction; orchestrate charge → ledger-write → confirm with **compensations** (void/refund).
  4. **Exactly-once-for-money is at-least-once + idempotency + reconciliation** — there is no magic exactly-once. The timeout-during-charge case is resolved by idempotent re-query against the provider plus end-of-day reconciliation.
  5. **Strong, serializable consistency everywhere authoritative — non-negotiable.** A stale balance read lets two concurrent withdrawals both succeed and overdraw. That's lost money and a regulator.

- **Follow-up probes (escalating):**
  1. Show me a single transfer of $10 from A to B as ledger rows. How many entries, and do they sum to zero?
  2. Why append-only? Why is balance *derived* and never an `UPDATE balance = balance - 10`? (lost-update / audit / no in-place mutation under concurrency.)
  3. Two concurrent withdrawals from the same wallet, balance exactly covers one. How does your model stop both? (serializable isolation on the balance check; the *derived* balance must be computed under the lock.)
  4. Idempotency key: what's it scoped to, *when* is it claimed, and what does a replay return?
  5. **The timeout case:** you charged Stripe, the call timed out, you don't know the outcome. Walk me through the next 5 steps. (don't re-charge blindly — re-query with the same idempotency key the *provider* honors; if unknown, reconcile, never double-charge.)
  6. Reconciliation: at end of day your ledger and Stripe's settlement file disagree. What's your process to converge, and what's authoritative for *which* fields?
  7. "Just use a queue with exactly-once delivery." React to that. (there is no exactly-once delivery; press them to say at-least-once + idempotency.)
  8. Multi-currency / fees: a charge has a fee and FX — how many ledger entries now, and do they still sum to zero?
  9. **Brutal:** Your ledger DB and your provider-call must both succeed or neither. They're different systems. No 2PC. How? (saga + outbox + idempotent compensation; the ledger write and the provider call are sequenced, not atomic — the saga + reconciliation closes the gap.)
  10. **Brutal:** Sharding the ledger for growth — a transfer crosses two shards (A on shard 1, B on shard 2). How do you keep it balanced and serializable? (per-account serialization + cross-shard saga, or co-locate by transaction; discuss the tradeoff.)

- **Strong answer covers / Weak answer red flags:**
  - ✅ Double-entry, append-only, balance derived — says this in the first two minutes.
  - ✅ Idempotency key claimed *before* the side effect; replay returns stored result.
  - ✅ Saga + compensations; explicitly rejects 2PC and explicitly rejects "exactly-once queue."
  - ✅ Timeout-during-charge: re-query with idempotency, reconcile, never blind-retry.
  - ✅ Serializable consistency stated as non-negotiable, with the overdraw example.
  - ❌ Mutates a `balance` column in place.
  - ❌ "Exactly-once via Kafka/SQS."
  - ❌ 2PC across the provider.
  - ❌ Relaxes consistency "for performance" — at 1K TPS there is no performance excuse; this is an instant red flag.
  - ❌ Blind retry on the charge timeout → double-charge.

- **Common traps:**
  - Treating the provider's response as ground truth (it lies / times out) instead of reconciling.
  - Forgetting that idempotency must be claimed *before* the side effect, not after.
  - Conflating "the ledger" with "the provider" — they outsource correctness to Stripe and skip building the ledger. Stripe is the gateway, not the ledger.
  - Reaching for sharding/eventual-consistency throughput tricks on a 1K-TPS workload — solving a problem they don't have while creating the one (correctness) that matters.

---

### Ad serving / RTB

- **Opening prompt:** "Design a real-time bidding ad exchange. A user loads a page; you run an auction across bidders and return a winning ad in **under ~100ms** end-to-end. Bidders have daily budgets that must not be overspent."

- **If they ask scope (curveballs to reveal):**
  - Massive QPS (millions of auctions/sec), hard **sub-100ms** latency budget (often <50ms for the auction itself).
  - Budgets are *global per advertiser* but auctions run on *distributed* servers — the pacing problem.
  - **Mid-interview change #1:** "An advertiser's budget is $10K/day. You have 200 bid servers. How do you not overspend across all of them in real time, without a global lock on every bid?"
  - **Mid-interview change #2:** "Budget pacing — don't blow the whole budget by 9am. Spend it smoothly across the day."

- **The crux (what they MUST nail):**
  1. **Sub-100ms auction path** — everything in-memory, candidate ad sets precomputed, no synchronous DB calls on the bid path, aggressive timeouts (a slow bidder is dropped, not waited on).
  2. **Budget control without a global lock** — distribute budget to servers in **leases/buckets**, reconcile asynchronously; accept small overspend as a bounded, reconciled error rather than serializing every bid.
  3. **Budget pacing** — spread spend across the day (probabilistic throttling / pacing rate), not first-come-first-spend.

- **Follow-up probes (escalating):**
  1. What's the latency budget breakdown across the auction? Where can you *not* afford a network hop?
  2. A bidder takes 200ms. Your budget is 100ms. What happens to that bid? (dropped on timeout — partial results are fine.)
  3. Budget across 200 servers without a global lock — how? (lease/sub-budget per server, periodic reconcile.)
  4. With leases, two servers' leases sum to more than remaining budget near the end of day → overspend. Is that acceptable? Bound it. (yes, bounded overspend; reconcile and refund/stop.)
  5. Pacing: how do you avoid spending the day's budget by 9am? (target spend curve, throttle bid participation probabilistically.)
  6. Frequency capping (don't show the same user the same ad 50×) — where's that state, and is it strong or eventual?
  7. Click/impression event volume dwarfs auctions. How do you count spend accurately enough for budgets but cheaply? (approximate real-time + exact batch reconciliation.)
  8. **Brutal:** Same auction, the latency budget is blown by the candidate-retrieval step. Cut it without tanking relevance.
  9. **Brutal:** Money correctness vs latency — billing must be *exact* but the bid path must be *fast*. Reconcile the two. (fast eventual pacing on the path; exact ledger reconciliation off-path — note the parallel to the Payment problem.)
  10. **Brutal:** A bidder is fraudulent / returns garbage. How does the auction stay sub-100ms and not get gamed?

- **Strong answer covers / Weak answer red flags:**
  - ✅ In-memory, no synchronous DB on the bid path, hard timeouts, partial-result auctions.
  - ✅ Budget leases + async reconcile; *names bounded overspend as an acceptable tradeoff*.
  - ✅ Pacing as a spend curve, not greedy.
  - ✅ Separates fast-eventual spend tracking from exact billing reconciliation.
  - ❌ A global budget lock / DB read on every bid (instantly blows the latency budget).
  - ❌ Waiting for all bidders (one slow bidder = missed SLA).
  - ❌ Claims *zero* overspend with no global coordination — physically impossible; the honest answer is bounded + reconciled.

- **Common traps:**
  - Demanding strict budget consistency on the bid path → either a global lock (too slow) or an impossible claim.
  - Synchronous candidate retrieval / feature lookup on the hot path.
  - Counting every impression exactly in real time instead of approximate-now + exact-later.

---

### Recommendation engine

- **Opening prompt:** "Design a recommendation system — given a user, return a ranked list of items (videos / products / posts) they're likely to engage with, at request time, low latency, over a catalog of hundreds of millions of items."

- **If they ask scope (curveballs to reveal):**
  - Catalog is huge (you can't score every item per request). Latency budget ~100–200ms for the list.
  - You have engagement logs; freshness of features matters (a just-watched video should influence the next request).
  - **Mid-interview change #1:** "You can't score 500M items per request. How do you get from 500M to ~500 candidates fast, then rank those?" (two-stage architecture.)
  - **Mid-interview change #2:** "The same feature (user's recent clicks) is needed at training time and at serving time, and they're computed differently → your model is wrong in production. Fix it." (feature store / train-serve skew.)

- **The crux (what they MUST nail):**
  1. **Two-stage retrieve-then-rank** — cheap **candidate generation** (ANN / embeddings / co-occurrence) narrows 500M → ~hundreds, then an **expensive ranking model** scores only those. You *cannot* run the heavy model over the full catalog.
  2. **Feature store** — consistent features across training and serving to avoid **train-serve skew**; online (low-latency) + offline (batch) stores fed from one definition.
  3. **Freshness path** — real-time signals (just-watched) update features fast enough to matter on the next request.

- **Follow-up probes (escalating):**
  1. 500M items, 150ms budget. Why can't you score them all? Walk me from 500M to 500. (ANN/embedding retrieval.)
  2. Candidate generation: embeddings + ANN vs collaborative-filtering co-occurrence — pick and defend.
  3. Ranking stage: what features, and where do they come from at serving time? (feature store online lookup.)
  4. Train-serve skew: the recent-clicks feature is computed in Spark for training and in code at serving — they diverge. How does a feature store fix it?
  5. Cold start (new user, new item) — what does each stage do?
  6. A user just watched a video; the next request should reflect it. What's the freshness latency of that feature, and how do you achieve it?
  7. How do you A/B test a new ranking model without shipping a regression to everyone?
  8. **Brutal:** Your ANN index is stale (new items not indexed for an hour). Is that acceptable? Where's the eventual-consistency line? (retrieval can be slightly stale; ranking features must be fresher.)
  9. **Brutal:** Ranking model latency p99 spikes. Degrade gracefully without an empty feed. (fallback to candidate-gen order / cached recs.)
  10. **Brutal:** How do you keep the *online* and *offline* feature stores from drifting? (single feature definition, shared computation, monitoring for skew.)

- **Strong answer covers / Weak answer red flags:**
  - ✅ Two-stage retrieve-then-rank, with the 500M→500 narrowing made explicit.
  - ✅ Feature store solving train-serve skew, online + offline from one definition.
  - ✅ A graceful-degradation fallback for ranker latency.
  - ❌ "Run the model on all items" / no candidate-generation stage.
  - ❌ No awareness of train-serve skew — features computed twice, two ways.
  - ❌ Treats it as a pure ML problem with no systems/latency/serving story.

- **Common traps:**
  - Single-stage scoring over the full catalog (latency-impossible).
  - Ignoring feature freshness and train-serve skew (the production-correctness bug).
  - No fallback when the ranker is slow → empty or timed-out feed.

---

### URL shortener

> Looks trivial; it's a *consistency-of-key-generation* and *cache-economics* problem in disguise. Use it as a warm-up that still has sharp edges.

- **Opening prompt:** "Design a URL shortener — POST a long URL, get a short code; GET the short code redirects to the long URL. Billions of URLs, read-heavy."

- **If they ask scope (curveballs to reveal):**
  - Read:write is *massively* read-skewed (redirects ≫ creations).
  - Codes must be unique, short, and ideally non-guessable / non-sequential depending on requirements.
  - **Mid-interview change #1:** "Two app servers generate a short code for two different URLs at the same instant. Guarantee no collision without a per-request round trip to a coordinator." (KGS — key generation service.)
  - **Mid-interview change #2:** "Should the redirect be 301 or 302? Defend it — there's a real business consequence."

- **The crux (what they MUST nail):**
  1. **301 vs 302** — **301 (permanent)** is cached by browsers/proxies → you *lose analytics and lose the ability to change/expire the mapping*; **302 (temporary)** forces a round-trip every time → you *keep analytics and control* at the cost of load. Naming the tradeoff (and that 301 sacrifices click tracking) is the signal.
  2. **Key generation without collisions** — **KGS (Key Generation Service)** pre-generates and hands out unique keys in batches, *or* base-62 encode a global counter / Snowflake-style ID. Avoid "hash the URL and hope" (collisions + need to handle them) unless they handle collisions correctly.
  3. **Cache the read path** — redirects are the hot path; the hot codes live in a cache/CDN, DB is the cold backstop.

- **Follow-up probes (escalating):**
  1. BOTEC: billions of URLs, read:write ratio? What's the actual redirect QPS vs create QPS? (catch them if they don't split it.)
  2. Short code: base-62 of a counter, hash, or random? Collision handling for each?
  3. KGS: how does it hand out unique keys to many servers without a per-request lock? (pre-allocated ranges / batches.)
  4. 301 vs 302 — pick one and tell me what you *give up*. (the analytics/control tradeoff.)
  5. Hot code (a viral link) — how does the read path stay fast? (cache/CDN; this code is in every cache.)
  6. Custom aliases (`/my-brand`) collide with generated codes. How do you keep both namespaces clean?
  7. Expiry / deletion — how, given 301s are cached at the edge you don't control?
  8. **Brutal:** The counter / KGS is a single point of failure. Make it HA without reintroducing collisions. (range pre-allocation per KGS node.)
  9. **Brutal:** Analytics — you want per-click counts at billions of redirects/day without slowing the redirect. (async fire-and-forget event, approximate counting.)
  10. **Brutal:** Storage: billions of mappings — sizing, and is this even a SQL problem? (KV store; size the working set.)

- **Strong answer covers / Weak answer red flags:**
  - ✅ States the read-skew and sizes redirect vs create QPS separately.
  - ✅ KGS or counter-based unique keys with correct collision story.
  - ✅ 301 vs 302 tradeoff *with the analytics consequence* named.
  - ✅ Cache/CDN on the redirect path; async analytics.
  - ❌ "Hash the URL" with no collision handling.
  - ❌ Per-request lock/coordinator for key generation.
  - ❌ Picks 301 "because it's permanent" with no awareness it kills analytics.
  - ❌ BOTEC: treats create and redirect QPS as equal.

- **Common traps:**
  - Choosing 301 reflexively and losing click analytics + control.
  - Single global counter as an unscaled SPOF.
  - Sequential codes when the requirement implies non-guessable.

---

### Rate limiter

- **Opening prompt:** "Design a distributed rate limiter — enforce 'N requests per user per window' across a fleet of API servers. It sits in front of everything, so it must be fast and correct under concurrency."

- **If they ask scope (curveballs to reveal):**
  - Distributed: many API servers share the *same* per-user limit; a per-server counter is wrong.
  - Algorithm matters: fixed window (boundary burst), sliding window, token bucket, leaky bucket.
  - **Mid-interview change #1:** "Two requests from the same user hit two servers at the same instant; both read the counter as N-1 and both allow → you let through N+1. Make the check-and-increment atomic." (Redis + Lua.)
  - **Mid-interview change #2:** "Your Redis (the limiter store) is *down*. Do you fail-open (allow all) or fail-closed (deny all)? Defend it." (depends on what's protected.)

- **The crux (what they MUST nail):**
  1. **Atomic check-and-increment** — read-then-write across servers races and over-admits. Solve with **Redis + a Lua script** (atomic on the single-threaded Redis), or `INCR` with expiry, so the limit check and the counter update are one atomic op.
  2. **Algorithm choice with tradeoffs** — fixed window is cheap but allows a 2× burst at the boundary; **sliding window** smooths it; **token bucket** allows bursts up to a cap then refills. Name the tradeoff, pick one.
  3. **Fail-open vs fail-closed** — a *deliberate* decision: fail-**open** for availability-critical user traffic (don't take the site down because the limiter is down), fail-**closed** for abuse/security-critical paths (don't let an attacker through).

- **Follow-up probes (escalating):**
  1. Per-server counters — why are they wrong for a shared limit? (each server only sees its share.)
  2. Fixed window vs sliding window — show me the boundary-burst bug in fixed window. (2× at the edges.)
  3. The check-and-increment race across two servers. Fix it. (Redis Lua atomicity — push until they say *atomic*.)
  4. Why Lua specifically, vs two round trips (GET then INCR)? (atomicity; no interleave.)
  5. Token bucket vs leaky bucket — pick one for an API with bursty-but-bounded clients.
  6. Redis is down. Fail-open or fail-closed? Now I tell you this limiter protects a *login* endpoint — does your answer change? (yes — security path leans fail-closed.)
  7. Redis is a hot shard for a celebrity user. How do you avoid the limiter itself becoming the bottleneck? (local pre-check + global, or sharded counters.)
  8. **Brutal:** Slight over-admission under a Redis failover (counter lost). Acceptable? Bound it.
  9. **Brutal:** Multi-region — a user's requests hit two regions. Is the limit global or per-region? What's the consistency cost of global? (global needs cross-region coordination → latency; often per-region with reconciliation.)
  10. **Brutal:** A two-tier limiter (local in-process + global Redis) — when does the local tier lie, and is that ok?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Atomic check-and-increment via Redis Lua (or atomic INCR+EXPIRE); explains *why* non-atomic over-admits.
  - ✅ Picks an algorithm and names its burst behavior.
  - ✅ Fail-open vs fail-closed as a *deliberate, path-dependent* choice.
  - ❌ GET-then-INCR with no atomicity (the race the whole problem is about).
  - ❌ Per-server counters for a shared limit.
  - ❌ No opinion on fail-open/closed, or the same answer regardless of what's protected.

- **Common traps:**
  - Non-atomic read-modify-write → over-admission (the central bug).
  - Fixed window without acknowledging the boundary burst.
  - Treating fail-open/closed as a config detail rather than a security/availability decision.

---

### Notification system

- **Opening prompt:** "Design a notification system — send push / SMS / email to users, triggered by events, at scale. Multi-channel, with user preferences and delivery guarantees."

- **If they ask scope (curveballs to reveal):**
  - Fan-out can be huge (one event → millions of recipients, e.g. a broadcast).
  - Channels (push/SMS/email) have different providers, latencies, and failure modes.
  - **Mid-interview change #1:** "A bug retries the send pipeline. Users must not get the same notification 5 times. How?" (idempotency / dedupe.)
  - **Mid-interview change #2:** "A broadcast to 50M users — how do you fan out without melting the system or the downstream providers?"

- **The crux (what they MUST nail):**
  1. **At-least-once delivery + dedupe = effectively-once** — there is no exactly-once; the pipeline retries, so dedupe on a notification key. (Same lesson as Payment/Food — catch "exactly-once via queue.")
  2. **Channel abstraction + per-provider rate limits / backpressure** — providers throttle you; the system must respect provider limits and degrade gracefully.
  3. **Fan-out strategy** — async, queued, batched; user preferences and quiet-hours filtering applied *before* the expensive send.

- **Follow-up probes (escalating):**
  1. One event → 1M recipients. Synchronous or async? Where's the queue?
  2. The send pipeline retries on a transient failure. How do users not get duplicates? (idempotency/dedupe key.)
  3. "Exactly-once delivery" — is that real? (no — at-least-once + dedupe; press them.)
  4. SMS provider rate-limits you to 1K/sec but you have 50M to send. What happens? (backpressure, queue, pacing.)
  5. User preferences / quiet hours — applied where in the pipeline, and why there? (before send, to avoid wasted provider calls.)
  6. Push token is stale / device uninstalled. How do you detect and stop wasting sends?
  7. Priority: a 2FA code and a marketing blast share the pipeline. How does the 2FA code not wait behind 50M marketing messages? (priority lanes.)
  8. **Brutal:** Provider returns "accepted" but never delivers. How do you know? (delivery receipts / reconciliation; honest about limits.)
  9. **Brutal:** Dedupe window — how long do you remember a notification key, and what's the storage cost at this scale?
  10. **Brutal:** A poison message (one recipient's send always fails) blocks the queue. Isolate it. (DLQ; don't let one bad message stall the batch.)

- **Strong answer covers / Weak answer red flags:**
  - ✅ At-least-once + dedupe key; *rejects* "exactly-once via queue."
  - ✅ Provider rate limits + backpressure; priority lanes for critical vs bulk.
  - ✅ Preference/quiet-hours filtering before the expensive send; DLQ for poison messages.
  - ❌ "Exactly-once delivery" claim.
  - ❌ Synchronous fan-out / no backpressure → melts the provider or self.
  - ❌ 2FA codes and marketing in one undifferentiated queue.

- **Common traps:**
  - The recurring "exactly-once via queue" myth.
  - No backpressure against rate-limited providers.
  - No priority isolation → critical messages starved behind a broadcast.

---

### Distributed cache

- **Opening prompt:** "Design a distributed cache (Memcached/Redis-class) — key/value, low latency, scales horizontally across nodes. Walk me through where a key lives and what happens when a node joins or dies."

- **If they ask scope (curveballs to reveal):**
  - Many nodes; keys must distribute evenly and *not* all remap when the fleet changes.
  - Read-heavy fronting a slower datastore.
  - **Mid-interview change #1:** "You add a node to scale out. With naive `hash(key) % N`, almost every key remaps and the whole cache misses at once. Fix it." (consistent hashing.)
  - **Mid-interview change #2:** "A single celebrity key gets 1M reads/sec and lands on one node — a hot key. That node is on fire." (hot-key replication.)

- **The crux (what they MUST nail):**
  1. **Consistent hashing (with virtual nodes)** — adding/removing a node remaps only ~1/N of keys, not all of them; vnodes smooth the distribution.
  2. **Hot-key handling** — replicate the hot key across nodes / client-side cache, since consistent hashing alone still pins one key to one node.
  3. **Eviction + invalidation** — LRU/LFU eviction; cache-aside vs write-through; and the **stampede** problem (a hot key expires → thundering herd to the DB).

- **Follow-up probes (escalating):**
  1. `hash(key) % N` — what's wrong when N changes? (mass remap → cache-miss storm.)
  2. Consistent hashing: how does adding a node remap only ~1/N keys? Why virtual nodes?
  3. Hot key: consistent hashing still pins it to one node. Now what? (replicate the key.)
  4. Eviction policy — LRU vs LFU, pick one for a read-heavy workload and defend it.
  5. Cache-aside vs write-through vs write-back — which, and what's the consistency/durability tradeoff?
  6. Stampede: a hot key expires, 100K requests miss simultaneously and hammer the DB. Prevent it. (lock/single-flight, probabilistic early refresh.)
  7. Stale data after a DB write — how does the cache get invalidated, and is there a race? (the read-then-write / invalidate race.)
  8. **Brutal:** A node dies. What do its keys do, and does the DB get a load spike? (its share misses → DB spike; mitigate with replication.)
  9. **Brutal:** Cache and DB disagree (cache has old value after a write). Walk me through the exact interleaving that causes it. (concurrent read populating cache with a stale value after the invalidate.)
  10. **Brutal:** Multi-region cache — do you replicate or keep per-region? Consistency cost of each?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Consistent hashing + vnodes; explains the ~1/N remap.
  - ✅ Hot-key replication (knows consistent hashing alone doesn't solve it).
  - ✅ Stampede prevention (single-flight / early refresh) and the invalidation race.
  - ❌ `mod N` hashing with no awareness of the remap storm.
  - ❌ Thinks consistent hashing solves hot keys.
  - ❌ No cache-invalidation consistency story / unaware of the stampede.

- **Common traps:**
  - Believing consistent hashing fixes hot keys (it doesn't — it pins one key to one node).
  - Ignoring the cache-stampede on hot-key expiry.
  - Hand-waving the cache/DB invalidation race.

---

### API gateway

- **Opening prompt:** "Design an API gateway — the single entry point in front of many backend services. It does auth, rate limiting, routing, and aggregation. Tell me what's on the request path and what's *not*."

- **If they ask scope (curveballs to reveal):**
  - Sits on the hot path for *every* request → its own latency and availability are critical.
  - Cross-cutting concerns: authn/z, rate limiting, routing, request transformation, observability.
  - **Mid-interview change #1:** "The gateway calls an auth service on every request synchronously. Auth is slow / down. The whole site is down. Fix the coupling." (token validation / caching, fail strategy.)
  - **Mid-interview change #2:** "One slow backend service makes the gateway's threads pile up and it stops serving *everyone*. Isolate it." (bulkheads / circuit breakers / timeouts.)

- **The crux (what they MUST nail):**
  1. **The gateway must not become a SPOF or a bottleneck** — stateless, horizontally scaled, behind an LB; keep heavy/synchronous calls *off* the hot path.
  2. **Resilience patterns** — timeouts, **circuit breakers**, **bulkheads** so one failing backend can't exhaust the gateway and take down unrelated routes.
  3. **Auth done right** — validate tokens (JWT) at the edge without a synchronous round trip per request where possible; decide the fail-open/closed posture.

- **Follow-up probes (escalating):**
  1. What's on the hot path per request, and what did you deliberately keep off it?
  2. Auth on every request — synchronous call to an auth service, or stateless token validation? Tradeoffs? (JWT validation at edge vs revocation freshness.)
  3. One backend is slow. Why do gateway threads pile up, and how do timeouts + bulkheads stop it? (resource isolation per route.)
  4. Circuit breaker: when does it open, what does the client see while open, and when does it close? (half-open probing.)
  5. Rate limiting at the gateway — global or per-instance? (ties back to the distributed rate-limiter problem.)
  6. Response aggregation (BFF-style fan-out) — one of three calls fails. Partial response or error? Defend it.
  7. Where does TLS terminate, and how do you avoid re-doing it per hop?
  8. **Brutal:** A bad config deploy to the gateway breaks routing for everyone. How do you make gateway changes safe? (canary, fast rollback, config validation.)
  9. **Brutal:** The gateway is stateless — but rate-limit counters and circuit-breaker state aren't. Where does *that* state live without making the gateway stateful? (external store / shared, with the consistency cost.)
  10. **Brutal:** Auth service is down. Fail-open (let requests through unauthenticated) or fail-closed? (almost always fail-closed for auth — contrast with the rate-limiter answer.)

- **Strong answer covers / Weak answer red flags:**
  - ✅ Stateless, horizontally scaled, not a SPOF; heavy calls kept off the hot path.
  - ✅ Timeouts + circuit breakers + bulkheads for blast-radius isolation.
  - ✅ Edge token validation with a clear revocation/freshness tradeoff; fail-closed on auth.
  - ❌ Synchronous auth call per request with no caching → auth is a SPOF for the whole site.
  - ❌ No isolation → one slow backend stalls all routes.
  - ❌ Stuffs business logic into the gateway (it becomes a distributed monolith).

- **Common traps:**
  - Synchronous per-request auth that couples the whole site's availability to the auth service.
  - No bulkheads → one bad backend exhausts shared threads/connections.
  - Putting real business logic in the gateway.

---

### Web crawler

- **Opening prompt:** "Design a web crawler — fetch billions of pages, follow links, store content for indexing. It must be polite to servers, avoid re-crawling the same URL, and not get stuck in traps."

- **If they ask scope (curveballs to reveal):**
  - Scale: billions of URLs, the URL frontier doesn't fit in memory.
  - Must respect `robots.txt` and not hammer a single host (politeness).
  - **Mid-interview change #1:** "You've crawled 2B URLs. Before fetching a new one, you must check 'have I seen this?' fast and with low memory. How?" (Bloom filter.)
  - **Mid-interview change #2:** "Your crawler is sending 1,000 req/sec to a single small website and DDoSing it. Make it polite." (per-host rate limiting / politeness queues.)

- **The crux (what they MUST nail):**
  1. **Seen-URL dedupe with a Bloom filter** — at billions of URLs you can't store every seen URL in a set; a **Bloom filter** gives O(1) membership with bounded memory and *no false negatives* (false positives just skip a real URL — acceptable tradeoff). Naming the false-positive tradeoff is the signal.
  2. **Politeness** — `robots.txt` compliance + **per-host rate limiting** so you don't hammer one server; the frontier must be organized *by host* with per-host delays, not a flat queue.
  3. **URL frontier at scale** — prioritized, partitioned, persisted (doesn't fit in memory); plus trap detection (infinite calendars, query-param explosions).

- **Follow-up probes (escalating):**
  1. "Have I seen this URL?" over 2B URLs with low memory. (Bloom filter.)
  2. Bloom filter false positives vs false negatives — which can you tolerate and why? (FP → skip a real URL, acceptable; FN never happens.)
  3. Politeness: how do you avoid hammering one host? Where does per-host delay live? (per-host frontier queues.)
  4. The frontier has billions of URLs and won't fit in memory. How is it structured? (partitioned/persisted priority queues.)
  5. Two crawler nodes both pull the same host's URLs and gang up on it. Coordinate. (host assigned to one worker.)
  6. Crawler traps — infinite URL spaces (calendars, faceted nav). Detect and escape. (depth/pattern limits.)
  7. Re-crawl freshness: how do you decide when to re-fetch a page you've seen? (change frequency / priority.)
  8. **Brutal:** The Bloom filter fills up and false-positive rate climbs — now you're skipping real pages. How do you handle growth? (scalable/partitioned Bloom filters, periodic rebuild.)
  9. **Brutal:** A node dies mid-crawl with URLs checked-out-but-not-done. How do you not lose or double-fetch them?
  10. **Brutal:** Politeness vs throughput — being polite to millions of hosts caps your global rate. How do you stay fast? (parallelism *across* hosts, not within.)

- **Strong answer covers / Weak answer red flags:**
  - ✅ Bloom filter for seen-set with the FP-vs-FN tradeoff stated correctly.
  - ✅ Politeness via per-host queues + robots.txt; parallelism across hosts.
  - ✅ Persisted/partitioned frontier; trap detection.
  - ❌ Stores every seen URL in a hash set / DB lookup per URL at billions of scale.
  - ❌ Flat global queue → DDoSes individual hosts.
  - ❌ Gets the Bloom filter tradeoff backwards (thinks false negatives happen).

- **Common traps:**
  - In-memory set of all seen URLs (memory-impossible at scale).
  - Flat queue ignoring politeness → effectively a DDoS tool.
  - No trap detection → stuck crawling an infinite calendar.

---

### Metrics / monitoring platform

- **Opening prompt:** "Design a metrics and monitoring platform — ingest time-series metrics from millions of hosts, store them, support queries and dashboards, and fire alerts. Walk me from a host emitting a counter to an alert paging someone."

- **If they ask scope (curveballs to reveal):**
  - Ingest is enormous and *write-heavy* (millions of series, high-frequency samples).
  - Queries are mostly recent + aggregated; old data is rarely queried at full resolution.
  - **Mid-interview change #1:** "You can't store every raw sample forever — it's petabytes. How do you keep recent data high-res and old data cheap?" (downsampling / rollups / retention tiers.)
  - **Mid-interview change #2:** "An alert must fire within seconds of a threshold breach, but your storage is optimized for batch queries. Reconcile the two paths." (streaming alerting vs query path.)

- **The crux (what they MUST nail):**
  1. **Time-series storage design** — write-optimized ingest (append-heavy), columnar/TSDB layout, **downsampling + retention tiers** (raw → 1m → 1h rollups; old data at lower resolution). You cannot store everything raw forever.
  2. **Cardinality is the killer** — high-cardinality labels (user_id, request_id) explode the series count; recognize and bound it. This is *the* Staff signal for this problem.
  3. **Separate alerting path from query path** — alerts evaluate on the streaming ingest path for low latency; dashboards query the stored rollups. Don't run alert rules as slow batch queries.

- **Follow-up probes (escalating):**
  1. BOTEC: millions of hosts × N metrics × sample rate = ? writes/sec. (force the ingest sizing.)
  2. Why is a general-purpose DB a bad fit? What does a TSDB do differently? (append-optimized, columnar, time-partitioned.)
  3. Storage: petabytes of raw samples. How do you make old data cheap without losing trends? (downsampling/rollups + retention tiers.)
  4. **Cardinality:** someone adds `user_id` as a metric label. What just happened to your series count? (explosion — bound/reject high-cardinality labels.)
  5. Alert latency: a threshold breach must page within seconds. Run alert rules as queries against storage? (no — evaluate on the ingest stream.)
  6. Query a 30-day dashboard at 1s resolution — does that even make sense? (no — serve from rollups.)
  7. Late-arriving / out-of-order samples (a host was partitioned). How do you handle them in a time-partitioned store?
  8. **Brutal:** An ingest spike (incident → everything alarms → metric storm) overwhelms ingestion exactly when you need it most. Survive it. (backpressure, sampling, prioritize critical metrics.)
  9. **Brutal:** Alert flapping (value oscillates around threshold) pages someone 100×. Fix it without missing real incidents. (hysteresis / for-duration / dedupe.)
  10. **Brutal:** Exactly-once vs at-least-once for metric ingest — does a dropped or duplicated sample matter? (mostly tolerant — aggregates absorb it; contrast sharply with the Payment problem to test judgment.)

- **Strong answer covers / Weak answer red flags:**
  - ✅ Write-optimized TSDB; downsampling + retention tiers.
  - ✅ Calls out **cardinality** as the dominant failure mode unprompted.
  - ✅ Alerting on the streaming path, dashboards on rollups — separate concerns.
  - ✅ Knows metric ingest is loss-tolerant (opposite of money) — *judgment about where consistency matters*.
  - ❌ Stores every raw sample forever / general SQL DB with no rollups.
  - ❌ No awareness of cardinality explosion.
  - ❌ Runs alerts as batch queries → seconds-to-minutes alert latency.
  - ❌ Over-engineers exactly-once for metrics (solving the wrong problem — wasted strictness).

- **Common traps:**
  - High-cardinality labels exploding series count (the classic outage).
  - Treating alerting as a query workload instead of a streaming one.
  - Applying money-grade consistency to loss-tolerant telemetry (mirror image of the Payment mistake — both are consistency-boundary misjudgments).

---

### Yelp (local search & reviews)

- **Opening prompt:** "Design Yelp. Users search for businesses near a location by name or category with filters, view a business, and leave star ratings and reviews."

- **If they ask scope (curveballs to reveal):**
  - ~20M businesses (static locations); ~100M MAU; ~10k search QPS at peak; ~3 review writes/s.
  - One review per user per business.
  - Average rating must be exact on the business page.
  - **Mid-interview change #1:** "Support 'restaurants in the Mission district'." (Polygon filter, point-in-polygon.)
  - **Mid-interview change #2:** "A coordinated campaign posts 5,000 one-star reviews on one restaurant in an hour." (Review-bombing.)

- **The crux (what they MUST nail):**
  1. **Static entities → no location firehose:** geo filter + text + structured filters in one search-engine query, ranked by relevance × distance decay × a rating signal.
  2. **Rating aggregates** updated atomically in the review transaction (sum + count), never `AVG()` on read.
  3. **One review per user** as a unique constraint.

- **Follow-up probes (escalating):**
  1. BOTEC: document size × count. Does it fit one cluster?
  2. Why not a QuadTree service like Uber? When would you build one?
  3. Write the query shape: what's a filter, what's a score?
  4. "Open now" across time zones: how, without scripts per query?
  5. Concurrent reviews on one business: how is the average kept correct?
  6. Should a place with one 5★ review outrank 4.6★ with 2,000 reviews? Fix it.
  7. Cache what, keyed how?
  8. **Brutal:** search shows the old rating for 10 minutes after a review. Acceptable? Where does it come from?
  9. **Brutal:** review-bombing: detect and contain without silencing real complaints.
  10. **Brutal:** move to a reindex with a new mapping with zero downtime.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Right-sizes (~40 GB) and uses one ES-style query; Postgres as source of truth via CDC.
  - ✅ Atomic `rating_sum`/`review_count`; Bayesian average for ranking.
  - ✅ Unique constraint for the one-review rule.
  - ❌ Custom geo-sharded index for 20M static rows.
  - ❌ Recomputes averages on read or in a nightly cron.
  - ❌ Sort purely by distance.

- **Common traps:** copying Uber; app-level duplicate checks; ignoring "open now" time zones.
- **Reference design:** [47 — Yelp](47-design-yelp.md).

---

### Online auction (eBay)

- **Opening prompt:** "Design an online auction like eBay. Sellers list items with an end time, buyers bid, the highest bid at the end wins, and watchers see the current price update live."

- **If they ask scope (curveballs to reveal):**
  - ~10M active auctions; ~1.2k bids/s average, hundreds per second on one hot auction in its final seconds.
  - Proxy bidding: users can enter a maximum.
  - No bid may be lost or accepted after close.
  - **Mid-interview change #1:** "Bots snipe in the last 200 ms." (Soft close, server receipt time.)
  - **Mid-interview change #2:** "The winner doesn't pay." (Second-chance offer from the bid log.)

- **The crux (what they MUST nail):**
  1. **Bid acceptance under contention:** conditional write by default; single writer per hot auction (Kafka partition by auction ID).
  2. **Closing correctly:** server receipt time decides validity; fenced, idempotent close.
  3. **Proxy bidding state** = top two maximum bids.

- **Follow-up probes (escalating):**
  1. Where is the consistency boundary?
  2. Two bids arrive at the same instant. Who wins and how is it enforced?
  3. Write the conditional update. What exactly is in the `WHERE`?
  4. 300 bids/s on one auction: what happens to the row lock, and what do you do?
  5. Explain proxy bidding's data model and the visible price.
  6. When does a bid count? Whose clock?
  7. The close job runs twice. What happens?
  8. **Brutal:** the scheduler fires 30 s late. Were late bids accepted?
  9. **Brutal:** a support ticket says "my bid didn't count." How do you answer it in one minute?
  10. **Brutal:** Buy It Now races a bid. Resolve it.

- **Strong answer covers / Weak answer red flags:**
  - ✅ Append-only bid log + derived state in one transaction.
  - ✅ OCC default, single-writer escalation for hot auctions.
  - ✅ Server timestamp rule, fenced close, soft-close as an explicit product choice.
  - ❌ Read-then-write in app code.
  - ❌ Client timestamps or "when the cron runs" as the cutoff.
  - ❌ Stores only the current price.

- **Common traps:** Redis `INCR` as a bid mechanism; forgetting idempotency on bid POSTs; no audit trail.
- **Reference design:** [48 — Online Auction](48-design-online-auction.md).

---

### Price tracking service (camelcamelcamel)

- **Opening prompt:** "Design a price tracker like camelcamelcamel. Show price history charts for Amazon products and let users set alerts that fire when a product drops below their target price."

- **If they ask scope (curveballs to reveal):**
  - ~20M tracked products, ~50M alerts, ~10M users.
  - **You only get ~2,000 price lookups per second** from the retailer's API (the hidden constraint).
  - Popular products' drops should alert within ~1 h; others within a day.
  - **Mid-interview change #1:** "A pricing glitch lists a TV at $1 for five minutes." (Confirmation fetch.)
  - **Mid-interview change #2:** "Add a browser extension that sees prices as users browse." (Untrusted observations as hints.)

- **The crux (what they MUST nail):**
  1. **The fetch budget is the constraint:** a priority scheduler (watchers × volatility × staleness) with tiered SLOs.
  2. **Alert matching as a range query** on a per-product index sorted by target price; fire on crossing with hysteresis.
  3. **Precision:** confirm large or wide-impact drops before notifying.

- **Follow-up probes (escalating):**
  1. Do the fetch-budget math. Can you check every product hourly?
  2. Design the scheduler. What's the priority function?
  3. Store price history how? Why not a row per check?
  4. A price changes: find the alerts to fire without scanning 50M.
  5. The price flickers between $79 and $80 around a $79.50 target. What does the user receive?
  6. 100k users watch one product that drops. How do notifications go out?
  7. Charts at ~600 QPS: how served?
  8. **Brutal:** the retailer throttles you to half the budget during Black Friday.
  9. **Brutal:** the $1 glitch: prevent a false alert to 50k users without delaying real drops much.
  10. **Brutal:** extension users report fake prices. What's your trust model?

- **Strong answer covers / Weak answer red flags:**
  - ✅ Names the fetch budget as the binding constraint early.
  - ✅ Change-only history, precomputed charts, last-checked shown.
  - ✅ Per-product sorted alert index, crossing + hysteresis, confirmation fetch.
  - ❌ "Cron every hour for every product."
  - ❌ Scans all alerts on every price change.
  - ❌ Trusts extension data directly.

- **Common traps:** sizing servers instead of the budget; alert spam from oscillation; ignoring terms of service.
- **Reference design:** [50 — Price Tracker](50-design-price-tracker.md).

---

## Scoring & cross-links

> **Use the shared 5-axis scoring rubric in [40a](40a-mock-chat-media.md)** (Problem framing / BOTEC · Consistency boundary · Scalability & bottlenecks · Failure handling · Communication & tradeoff ownership). Score each axis, write the *failure mode* not just the number, and note the highest probe rung the candidate cleared.

**The three reusable Staff signals across this entire bank:**

1. **Consistency-boundary judgment** — they draw strong vs eventual *deliberately* and per-field. Money/inventory = strong-serializable, non-negotiable. Telemetry/recs/catalog/feeds = eventual, by choice. Mis-drawing it *either way* (relaxing money, or over-strictening metrics) is the same failure: not knowing where correctness lives.
2. **Saga-not-2PC + idempotency + reconciliation** — for any cross-service / external-provider flow, they reach for sagas with compensations, idempotency keys claimed *before* side effects, and reconciliation as the safety net. They reject "exactly-once via a queue" on sight.
3. **Honest BOTEC** — peak factor, read/write split, working-set sizing. They never say "DAU = RPS." They notice when the firehose (telemetry, browse, ingest) dwarfs the consistency-critical path (matches, orders, charges) — and design the two paths *differently*.

See also: **[20 Staff Execution](20-staff-execution.md)** for what "Staff signal" means beyond the whiteboard, and **[26 Google SD Playbook](26-google-sd-playbook.md)** for the Google-EM-specific framing (scope negotiation, BOTEC discipline, leadership/influence woven into the design narrative). Reference designs: **[18c](18c-design-marketplace-money.md)** (marketplaces & money) and **[18d](18d-design-infra-primitives.md)** (infra primitives).
