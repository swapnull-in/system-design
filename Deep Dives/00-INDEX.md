# Deep Dives — Technology-Specific Mastery

> Companion track to the main course. Each deep dive takes **one technology** (Redis, Kafka, Postgres, Cassandra, DynamoDB, Elasticsearch…) and goes to the depth an interviewer probes when you name-drop it: internals, failure modes, the numbers, and exactly how to wield it across design problems.

## Why this track exists

The main modules teach *concepts* (caching, messaging, storage). But in the room, the moment you say **"I'll use Redis here"**, the follow-up is *"OK — what happens when the primary dies mid-write?"* Deep dives make sure that when you commit to a technology, you can defend it three probes deep.

## How to use

- Read the matching **concept module first** (each deep dive lists its prerequisites), then the deep dive.
- Each deep dive ends with an **interview cheat-sheet** — that's your pre-interview 10-minute refresher.
- Sources (videos/articles) are linked at the top of each file — watch/read once, then rely on the note.

## Topics

> Roadmap = Hello Interview's **[Deep Dives playlist](https://www.youtube.com/playlist?list=PL5q3E8eRUieUHnsz0rh0W6AzwdVJBwEK6)** (+ their written guides), distilled into the course format.

| # | Deep Dive | Pairs with module | Source | Status |
|---|-----------|-------------------|--------|--------|
| 01 | [Redis](01-redis.md) | [03 — Caching](../Core%20Course/03-caching.md) · [09 — Real-Time](../Core%20Course/09-realtime-systems.md) | [Video](https://youtu.be/fmT5nlEkl3U) | ✅ Done |
| 02 | [Kafka](02-kafka.md) | [04 — Messaging](../Core%20Course/04-messaging-event-driven.md) | [Video](https://www.youtube.com/watch?v=DU8o-OTeoCc) | ✅ Done |
| 03 | [PostgreSQL](03-postgresql.md) | [06 — Databases](../Core%20Course/06-databases.md) | [Written guide](https://www.hellointerview.com/learn/system-design/deep-dives/postgres) (no public video) | ✅ Done |
| 04 | [Cassandra](04-cassandra.md) | [06 — Databases](../Core%20Course/06-databases.md) | [Video](https://www.youtube.com/watch?v=TD3-INhm60Q) | ✅ Done |
| 05 | [DynamoDB](05-dynamodb.md) | [06 — Databases](../Core%20Course/06-databases.md) | [Video](https://www.youtube.com/watch?v=2X2SO3Y-af8) | ✅ Done |
| 06 | [Elasticsearch](06-elasticsearch.md) | [07 — Search](../Core%20Course/07-search.md) | [Video](https://www.youtube.com/watch?v=PuZvF2EyfBM) | ✅ Done |
| 07 | [Flink](07-flink.md) | [16 — Data Processing](../Core%20Course/16-data-processing.md) | [Written guide](https://www.hellointerview.com/learn/system-design/deep-dives/flink) (no public video) | ✅ Done |
| 08 | [ZooKeeper & etcd](08-zookeeper.md) | [02 — Distributed Systems](../Core%20Course/02-distributed-systems.md) | [Written guide](https://www.hellointerview.com/learn/system-design/deep-dives/zookeeper) (no public video) | ✅ Done |
| 09 | [API Gateway](09-api-gateway.md) | [05 — Load Balancing](../Core%20Course/05-load-balancing.md) · [10 — Microservices](../Core%20Course/10-microservices.md) | [Written guide](https://www.hellointerview.com/learn/system-design/deep-dives/api-gateway) | ✅ Done |
| 10 | [Time Series Databases](10-time-series-databases.md) | [12 — Observability](../Core%20Course/12-observability.md) · [39 — IoT](../Core%20Course/39-design-iot-ingestion.md) | [Written guide](https://www.hellointerview.com/learn/system-design/deep-dives/time-series-databases) | ✅ Done |
| 11 | [Vector Databases](11-vector-databases.md) | [17 — AI Systems](../Core%20Course/17-ai-systems.md) | [Written guide](https://www.hellointerview.com/learn/system-design/deep-dives/vector-databases) (premium) | ✅ Done |
| 12 | [Big-Data Structures (Bloom/HLL/CMS/t-digest)](12-big-data-structures.md) | [34 — Real-Time Analytics](../Core%20Course/34-design-realtime-analytics.md) | [Written guide](https://www.hellointerview.com/learn/system-design/deep-dives/data-structures-for-big-data) (premium) | ✅ Done |
| 20 | [ClickHouse & Columnar OLAP](20-clickhouse.md) | [34 — Real-Time Analytics](../Core%20Course/34-design-realtime-analytics.md) · [16 — Data Processing](../Core%20Course/16-data-processing.md) | Beyond the HI catalog (standard internals) | ✅ Done |
| 34 | [NewSQL / Distributed SQL (Spanner, CockroachDB)](34-newsql-distributed-sql.md) | [06 — Databases](../Core%20Course/06-databases.md) · [02 — Distributed Systems](../Core%20Course/02-distributed-systems.md) · [22 — Google Papers](../Core%20Course/22-google-papers.md) | Spanner (OSDI'12) + CockroachDB docs; beyond the HI catalog | ✅ Done |
| 37 | [Temporal & Durable Execution](37-temporal-durable-execution.md) | [Pattern 15 — Multi-step](15-pattern-multi-step-processes.md) · [35 — Scheduler](../Core%20Course/35-design-scheduler.md) | Temporal docs + Cadence lineage; HI Key Technologies | ✅ Done |
| 38 | [In the Wild — Real-System Case Studies](38-in-the-wild-case-studies.md) (Shopify, Slack, Figma, Spotify, Meta ZGateway) | Patterns 14–19 · [11 — Reliability](../Core%20Course/11-reliability.md) | Company engineering posts, cross-checked with HI "In the Wild" | ✅ Done |
| 39 | [Change Data Capture (CDC)](39-change-data-capture.md) | [07 — Search](../Core%20Course/07-search.md) · [32 — Migrations](../Core%20Course/32-migrations.md) · [Pattern 16](16-pattern-scaling-reads.md) | [Written guide](https://www.hellointerview.com/learn/system-design/deep-dives/change-data-capture) (premium) + Debezium / Postgres / MySQL docs | ✅ Done |

> Proximity search from the playlist is already covered in depth by [29 — Geospatial Systems](../Core%20Course/29-geospatial-systems.md) and [DD-01 §5.5](01-redis.md#55-proximity-search-geospatial). Numbers 13–19 are the patterns section below.
>
> **⚡ [CHEATSHEET.md](CHEATSHEET.md)** — the cheat sheet of cheat sheets: every one-liner, the numbers, the four universal probes, and the seven themes. The 10-minute day-of-interview read.

## Patterns — cross-cutting design moves

> Hello Interview's **patterns series**: not technologies but the recurring *shapes* interview problems take. Tighter notes — each is a decision ladder that leans on the tech deep dives above.

| # | Pattern | The one-line move | Source | Status |
|---|---------|-------------------|--------|--------|
| 13 | [Realtime Updates](13-realtime-updates.md) | Two hops: pub/sub routing + SSE-unless-client-streams-back | [Guide](https://www.hellointerview.com/learn/system-design/deep-dives/realtime-updates) | ✅ Done |
| 14 | [Dealing with Contention](14-pattern-contention.md) | Cheapest mechanism that holds the invariant: conditional write → lock → OCC → serialize | [Guide](https://www.hellointerview.com/learn/system-design/patterns/dealing-with-contention) | ✅ Done |
| 15 | [Multi-step Processes](15-pattern-multi-step-processes.md) | Name where workflow state lives: saga + compensation + idempotency, Temporal at scale | [Guide](https://www.hellointerview.com/learn/system-design/patterns/multi-step-processes) | ✅ Done |
| 16 | [Scaling Reads](16-pattern-scaling-reads.md) | Do less work → more copies → don't hit the DB: index → replicas → cache → CDN | [Guide](https://www.hellointerview.com/learn/system-design/patterns/scaling-reads) | ✅ Done |
| 17 | [Scaling Writes](17-pattern-scaling-writes.md) | Right engine → batch → queue the burst → shard by key → salt the whales | [Guide](https://www.hellointerview.com/learn/system-design/patterns/scaling-writes) | ✅ Done |
| 18 | [Handling Large Blobs](18-pattern-large-blobs.md) | Metadata in DB, bytes in S3, presigned URLs both ways — servers move pointers | [Guide](https://www.hellointerview.com/learn/system-design/patterns/large-blobs) | ✅ Done |
| 19 | [Long-Running Tasks](19-pattern-long-running-tasks.md) | 202 + job id → queue → idempotent workers on depth → DLQ | [Guide](https://www.hellointerview.com/learn/system-design/patterns/long-running-tasks) | ✅ Done |
| 21 | [Pattern Selection](21-pattern-selection.md) | The meta-pattern: 4-question triage → cue table → name the crux, dive there | [Guide](https://www.hellointerview.com/learn/system-design/patterns/pattern-selection) | ✅ Done |
| 22 | [Rate Limiting & Admission Control](22-rate-limiting.md) | Token bucket by default; atomic-in-Redis for exact, local+borrowing for fast; fail-open reads, fail-closed money | Stripe/Cloudflare posts + Alex Xu ch 4 | ✅ Done |

## Applied case studies — money & analytics paths

> The patterns above, composed into the questions loops actually ask. Client-side twins live in the [Mobile track](../Mobile%20System%20Design/00-INDEX.md) (M-29, M-30). The full worked flash-sale package (15 docs, YouTube-view-count style) lives in Supplementary/E-Commerce Flash Sale.

| # | Case study | The one-line move | Source | Status |
|---|-----------|-------------------|--------|--------|
| 23 | [Exactly-Once Purchase: Backend Idempotency](23-exactly-once-purchase.md) | At-least-once delivery is a law: atomic key claim + order + response snapshot in one txn; replay the receipt | Stripe idempotency posts | ✅ Done |
| 24 | [Banking Ledger & Money Transfer](24-banking-ledger.md) | Double-entry append-only; one boundary = ACID txn, two = saga + suspense; debit-first; serialize debits, credits commute | Ledger practice + DDIA | ✅ Done |
| 25 | [Limited Stock: Inventory Contention & Flash Sales](25-flash-sale-inventory.md) | Atomic conditional decrement; TTL holds (never undersell either); hot-row ladder → Redis gate → single-writer → tokens; cart ≠ reservation; authorize-allocate-capture | Flash-sale architectures | ✅ Done |
| 26 | ["Design an E-commerce Platform," at Staff/EM](26-ecommerce-platform-interview.md) | The capstone: scope to the money path, name the three hard problems (dedup/serialize/saga), the 45-min shape, the Senior-vs-Staff-vs-EM altitude table, failure drills to volunteer | [Senior mock video](https://www.youtube.com/watch?v=RuGY_1pap74), one level up | ✅ Done |
| 27 | ["Top-K Most-Listened Songs," at Staff/EM](27-topk-music-analytics.md) | The mock elevated: name the class (top-K over a counting cube), rollup-never-recount, per-user split, dedup + event-time spine, freshness as a priced menu, privacy/integrity unprompted | [Google India mock video](https://www.youtube.com/watch?v=S1DvEdR0iUo), one level up | ✅ Done |
| 28 | [Distributed Leaderboard & Ranking](28-distributed-leaderboard.md) | Three queries, three costs: top-N cached, my-rank = prefix sum (one ZSET until ~100M — know when NOT to distribute), friends = point reads; histogram-approximate + snapshot-exact past that; projection-not-truth; payouts read frozen snapshots | Redis ZSET practice + HI leaderboard family | ✅ Done |
| 29 | [Typeahead / Search Autocomplete](29-typeahead-autocomplete.md) | Top-K per prefix, all computation at build time — read path is a point lookup; client is tier zero (debounce/cancel/cache); daily base + minutes trending overlay; k-distinct gate (privacy + anti-gaming); corpus = publishing surface with a takedown SLA | Expands [Core 07 §4.2](../Core%20Course/07-search.md); classic Google question | ✅ Done |
| 30 | [The Money Path, End to End: Journeys](30-money-path-journeys.md) | CS-23/24/25 re-cut by *time*: user journey ↔ engineering workflow stage by stage (browse → claim → pay → branches → aftermath; transfer intent → gate → debit-first → rails → resolution), with challenges + the trade-off taken at each stage — the narration layer for mocks | Companion walkthrough to CS-23/24/25 | ✅ Done |
| 31 | [Quick Commerce: The Dark-Store Backend](31-quick-commerce-dark-store.md) | The dark store is the shard — resolve user→store once and everything is small; serviceability = admission control (geofence + capacity defends the 10-min SLA at the front door); hold at proceed-to-checkout, not add-to-cart; "never oversell" splits: DB = WHERE clause, shelf = buffer + compensation (the picker is a second writer) | Blinkit/Zepto-style prompt, composed on CS-23/25 + Core 29/18c | ✅ Done |
| 32 | [Design a Distributed Message Queue (Build It, Don't Name It)](32-design-message-queue.md) | Build the log, not the mailbox — per-message state collapses to a cursor; durability = replication + high-watermark, not fsync; speed = "nothing random, ever" (sequential IO, page cache, zero-copy); pull turns backpressure into observable lag; consensus on metadata only | Grokking ch. 17 elevated; composed on Core 04/21 + DD-02 | ✅ Done |
| 33 | [Messaging Queues: The Mastery Bar](33-messaging-mastery-bar.md) | The self-test: floor → five Staff questions → internals → judgment → operations → EM bar, each probe with "the bar" a strong answer must contain, a 10-question final gate, and a study map per miss — pass = cold, out loud | Self-assessment companion to Core 04 + DD-02 + CS-32 | ✅ Done |
