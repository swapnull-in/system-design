# Deep Dive 38 — In the Wild: Real-System Case Studies

> **Sources:** each case lists the original engineering post. Summaries were cross-checked against Hello Interview's "In the Wild" write-ups of the same posts. Where a figure comes from the company's post it's quoted as reported; where a point is analysis rather than the company's claim, it says so.
> **Companions elsewhere in the repo:** [Discord's message store](04-cassandra.md#10-case-study--how-discord-stores-trillions-of-messages) (DD-04 §10) · [GitLab's 2017 database loss](../Core%20Course/11-reliability.md) (Core 11 §7).

## Why this file exists

Interviewers love "have you seen this solved in practice?" A real case study does three things a hypothetical can't: it shows a **constraint that forced the design**, a **failure that taught the lesson**, and a **trade-off someone actually paid**. Each case below ends with the one sentence to say in the room and the pattern it illustrates, so it slots into a design rather than becoming a tangent.

| # | Case | The lesson in one line | Pattern |
|---|------|------------------------|---------|
| 1 | [Shopify — inventory reservations](#1-shopify--inventory-reservations-from-redis-to-mysql) | Put the hold and the ledger in one transaction, and spread a hot row into many | [P14 Contention](14-pattern-contention.md) |
| 2 | [Slack — job queue](#2-slack--kafka-in-front-of-redis-for-the-job-queue) | Separate the burst buffer from the working set | [P17 Writes](17-pattern-scaling-writes.md) · [P19](19-pattern-long-running-tasks.md) |
| 3 | [Figma — multiplayer](#3-figma--multiplayer-editing-with-a-simplified-crdt) | A central server lets you delete most of CRDT's complexity | [38 Collaborative whiteboard](../Core%20Course/38-design-collaborative-whiteboard.md) |
| 4 | [Spotify — point queries on the data lake](#4-spotify--point-queries-on-the-data-lake) | Index the lake instead of copying it into a KV store | [P16 Reads](16-pattern-scaling-reads.md) |
| 5 | [Meta — ZGateway for ZippyDB](#5-meta--zgateway-in-front-of-zippydb) | A proxy that pools, batches, coalesces, caches and admits | [P16](16-pattern-scaling-reads.md) · [09 API gateway](09-api-gateway.md) |

---

## 1. Shopify — Inventory Reservations from Redis to MySQL

> **Source:** Shopify Engineering, *"Scaling inventory reservations"* (2026).

**Context.** Checkout reserves units for a few minutes while payment runs, then either deducts them permanently or releases them. Reservations lived in **Redis counters**; the permanent inventory ledger lived in **MySQL**.

**What broke.** **Split-brain across two stores:** a crash between the Redis write and the MySQL write either hid stock that existed or oversold stock that didn't. Reordering the writes just moved the failure; Redis can't take part in a two-phase commit; an outbox or reconciliation job fixes it *eventually*, which is too late during a flash sale.

**First attempt that failed:** one MySQL row per item with a quantity (`UPDATE … SET available = available - 1 WHERE available >= 1`). Correct, but every buyer of a hot item serialized on **one row lock**, so a flash sale became sequential.

**The design that worked:**
- **One row per reservable unit** in a `reservation_units` pool. Reserving N units = `SELECT … FOR UPDATE SKIP LOCKED LIMIT N`, delete those rows, insert one hold row, all **in one transaction with the ledger**. Concurrent buyers lock *different* rows and commit in parallel.
- **Bounded pool:** at most ~1,000 rows per item, refilled by a background replenisher; if a rush drains it, the request refills inline under a lock so capacity isn't added twice.
- **Lock hygiene:** primary key `(shop_id, inventory_item_id, inventory_group_id, id)` so the query locks rows directly (one lock per unit instead of index entry + row); **READ COMMITTED** for these transactions, because REPEATABLE READ's **gap locks** on an empty pool deadlocked with replenishment inserts; and **one global lock order** (delete from the pool before inserting the hold) on every code path.
- **Result (reported):** held up through Black Friday 2025, with merchant sales peaking at about **$5.1M per minute**.

**Analysis (not from Shopify's post) — expiring holds.** In MySQL each hold carries `expires_at`, and availability excludes expired holds in an indexed query, so nothing has to fire at expiry; cleanup is garbage collection, not correctness. In Redis, a key TTL can't give units back to a shared counter, and keyspace notifications can be lost, which is part of why the counter design was fragile.

> **Say in the room:** *"When the hold and the ledger must commit together, they belong in one database. Shopify moved reservations from Redis into MySQL for exactly that, and avoided the hot-row lock by making each reservable unit its own row, claimed with `SKIP LOCKED`, so buyers lock different rows in parallel."*

---

## 2. Slack — Kafka in Front of Redis for the Job Queue

> **Source:** Slack Engineering, *"Scaling Slack's job queue"* (2017).

**Context.** Slack's async job queue (message posting side effects, push notifications, URL unfurls, reminders, billing) ran entirely on **Redis**: over **1.4 billion jobs** on peak days, about **33,000 jobs/s** at peak.

**What broke.** A database slowdown made jobs pile up until Redis hit **maxmemory**. Enqueues failed, *and dequeues failed too*, because moving a job to the in-flight list also needs memory. The queue stayed wedged after the database recovered and needed manual repair. **Root cause: the burst buffer and the dispatch workspace shared one bounded pool of RAM**, so overload ended in lockup instead of lag. Secondary problems: every client connected to every Redis instance, workers couldn't scale without adding polling load, and dequeue cost grew with queue length.

**The design — the minimum viable change:**
- **Kafka in front of Redis**, not instead of it. Kafka holds the backlog on disk (bounded by retention, not RAM); Redis keeps only the short dispatch window and the existing queue semantics, so workers and the enqueue API didn't change.
- **Kafkagate** (stateless Go service): HTTP `POST /enqueue` → Kafka. Persistent broker connections, AZ-local preference, and **leader-only acks** (lower latency, a small risk of losing a job if a broker dies before replicating — a deliberate trade).
- **JQRelay** (stateless Go service): one instance per topic (via **Consul locks**) drains Kafka into the matching Redis cluster, **committing offsets only after a successful Redis write**, retrying indefinitely if Redis is down while Kafka absorbs new work. Its **rate limit is read live from Consul**, so operators throttle the drain during an incident while enqueues continue at full speed.
- **Config (reported):** 16 brokers, 32 partitions per topic, RF 3, 2-day retention, rack/AZ-aware replicas; per-partition **heartbeat canaries** every minute across 1,600 partitions.
- **Rollout:** double writes → shadow mode (relay reads and discards) → hop-by-hop counts → internal dogfooding → customers one job type at a time. The snag: JSON re-encoding differed between **Go and PHP** (Go escapes `<`, `>`, `&`; PHP escapes `/`), so the same job produced different bytes.

> **Say in the room:** *"Never let the backlog and the working set share one bounded resource. Slack's Redis queue locked up when a burst filled memory, so they put Kafka in front: the backlog lives on disk, Redis holds only in-flight work, and a throttle on the relay gives operators a control point."*

---

## 3. Figma — Multiplayer Editing with a Simplified CRDT

> **Source:** Evan Wallace, *"How Figma's multiplayer technology works"*, Figma Engineering blog (2019). Multiplayer launched in 2016.

**Context.** Many editors on one design file: instant local feedback, hours of offline editing merged on reconnect, and a **tree** of objects where two users can do things like move one object into two different frames at once.

**The key insight.** Figma rejected **OT** (too many edge cases) and noticed that most **CRDT** machinery exists for *decentralized* systems. Since every edit already goes through a server, **the server can define the order**, and most of that complexity disappears.

**The design:**
- **Document = `Map<ObjectID, Map<Property, Value>>`**; **each property is a last-writer-wins register**, and the winner is simply whichever write **arrives at the server last**. No timestamps. The whole property value is atomic, which suits visual properties (merging "red" and "blue" means nothing).
- **Clients apply edits optimistically** and **ignore server updates that conflict with their own unacknowledged writes**, which stops flicker.
- **IDs embed a client ID**, so objects can be created offline without collisions. Deletes remove the entry; a property write can't resurrect a deleted object.
- **Reparenting = writing the child's `parent` property**, so identity survives and one value means one parent. **Cycles** (A under B while B goes under A) can't be prevented per property, so the **server rejects** a parent write that would create one.
- **Sibling order uses fractional positions** (insert between 0.3 and 0.4 → 0.35), so inserting never renumbers siblings; `{parentID, position}` is written as one atomic property.
- **One server process per open document**: every editor of a file connects to it, which is what makes "last to arrive" well defined. Reconnect = re-download the document, replay unacknowledged local edits, reopen the WebSocket, the same path for 4 seconds or 4 hours offline.

**The trade-off.** Algorithmic complexity was swapped for **infrastructure complexity**: routing clients to the right document process and failing over without two processes accepting writes for one file. Free text isn't merged character by character; that needs a real text CRDT (Yjs is the common choice). The pattern became mainstream: per-property LWW with a server ordering optimistic mutations, and "one process per document" as a platform primitive (Cloudflare Durable Objects).

> **Say in the room:** *"If every edit goes through a server anyway, you don't need a full CRDT. Figma uses per-property last-writer-wins ordered by arrival at one process per document, fractional indexes for sibling order, and a server-side cycle check for reparenting. They traded algorithmic complexity for routing and failover, which is easier to operate."*

---

## 4. Spotify — Point Queries on the Data Lake

> **Source:** Spotify Engineering, *"Indexing the Data Lake for Online Point Queries"* (2026). Note: the post reports mechanisms and size ratios but no production end-to-end latency or cost savings.

**Context.** Exabytes of events in object storage as **Parquet**: great for scans ("average listening time by country"), bad for point lookups ("what did user X listen to?"), which touch a few scattered rows but must read whole pages through a distributed query engine. The usual fix, copying hot data into Bigtable/DynamoDB, **duplicates data and costs per GB**, so teams ration what's online.

**The design — make the lake itself point-queryable:**
- **Prune:** a 90-day lookup spans ~90,000 files (~1,000 per day). **Bucketing by `hash(user_id) % 1000`** cuts that to ~1 file per day (~90). **Per-file Bloom filters** on the user-ID column, cached in a metadata store, rule out most of the rest (~12 files).
- **RAP (Random Access Parquet) index:** an external, append-only index mapping key → (file, row numbers). Built incrementally from Parquet footers and key columns, **~1/1000th the data size**, sharded by key hash. Unlike Bloom filters, which only rule files *out*, the index **locates** rows, which become **byte ranges** read **in parallel**.
- **Write-side layout:** sort by key and **start a new Parquet page whenever the key changes**, so a lookup reads a page holding only that user's rows (a few KB instead of a ~4 MB mixed page). ~20 bytes of header overhead per boundary; files remain standard Parquet.
- **Covering entries:** small values or precomputed aggregates stored in the index answer some lookups with no data read at all. **Secondary indexes** are just more overlays over the same files (less efficient, since data is laid out by the primary key).

> **Say in the room:** *"Before copying lake data into a serving store, check whether the lake can serve the lookup. Spotify bucketed and sorted Parquet by user, added Bloom filters to prune and an external row-level index to locate, so a point query becomes an index lookup plus parallel ranged reads. Bloom filters prune; only an index locates."*

---

## 5. Meta — ZGateway in Front of ZippyDB

> **Source:** Meta Engineering, *"ZGateway"* (2026). The post doesn't give batch sizes, timers or queue limits.

**Context.** ZippyDB is Meta's shared key-value store used by hundreds of teams and **over a million client machines**. Problems: **connection overhead** (tens of thousands of connections per database host, each costing memory and TLS/auth CPU), **reconnect storms** when fleets restart together, **repeated identical reads** of hot keys, and **noisy neighbours** filling shared queues.

**The design — a reverse proxy with five techniques**, each with its price:

| Technique | What it does | What it costs |
|-----------|--------------|----------------|
| **Connection pooling** | Clients connect to a few gateways; gateways share a small set of backend connections | One more hop |
| **Batching** | Requests for keys 73, 81, 92 go to the backend as one request (still 3 lookups, but one round of serialization/network overhead); flush on timer or size; cap in-flight batches | Latency for early callers in a batch |
| **Request coalescing** | Callers asking for a key already in flight attach to that lookup: fewer actual reads, best for sudden hot keys (per gateway only) | Memory and bookkeeping for waiters; needs limits and timeouts |
| **Read-through cache** | Local in-memory cache with a **per-key fill lock** (stampede protection) and **CDC-driven invalidation** | Staleness within an agreed maximum lag |
| **Admission control** | Bounded queues per use case and priority, served in turns; a full queue rejects only that team's excess | Some callers get errors and must back off |

> **Say in the room:** *"When a shared store has a million clients, put a gateway in front and name what each technique trades: pooling for connections, batching trades latency for overhead, coalescing trades memory for fewer reads, caching trades freshness, and admission control trades availability for isolation. That's Meta's ZGateway in front of ZippyDB."*

---

## Interview Cheat-Sheet

| Prompt you hear | Case to cite | The sentence |
|-----------------|--------------|--------------|
| Flash sale / inventory / "last unit" | **Shopify** | One row per reservable unit + `SKIP LOCKED`, in the same transaction as the ledger |
| Queue fell over under a burst | **Slack** | Kafka in front of Redis: backlog on disk, working set in RAM, throttle at the drain |
| Collaborative editing | **Figma** | Server-ordered per-property LWW, fractional indexes, one process per document |
| Serve lookups from analytics data | **Spotify** | Bucket + sort + Bloom prune + row-level index; don't copy the lake |
| Many clients hammering a shared DB | **Meta ZGateway** | Pool, batch, coalesce, cache, admit — and name each trade |
| Chat storage at trillions | **Discord** ([DD-04 §10](04-cassandra.md#10-case-study--how-discord-stores-trillions-of-messages)) | Route by channel + coalesce; GC-free engine second |
| "How do you know backups work?" | **GitLab** ([Core 11](../Core%20Course/11-reliability.md)) | Five unverified mechanisms = zero; automate restore-and-verify |
