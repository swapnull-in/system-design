# Deep Dives — The Cheat Sheet of Cheat Sheets

> The day-of-interview read: every technology's opening line, the numbers, the universal probes, and the themes that let you derive the rest. ~10 minutes. Each row links to its full note for the deep version.

---

## The seven themes (derive, don't memorize)

1. **Everything fast + durable is an append-only log** — Postgres WAL · Cassandra commit log · Kafka · Lucene segments · TSDB partitions · Flink checkpoints · MergeTree parts. *Append now, organize in the background, replay to recover.*
2. **The partition key is the design** — Kafka key · Cassandra PK · DynamoDB PK · Flink `keyBy` · ES routing · ClickHouse ORDER BY. Key by the entity that needs ordering/locality, nothing coarser; it sets ordering, parallelism ceiling, and hot-spot risk.
3. **Hot keys are the universal disease** — uniform hashing assumes uniform access; a viral entity breaks every system identically. Medicine: L1/local cache · salt/split the key (`k_0..9`, merge on read) · replicas for read-hot · sharded counters / two-stage aggregation for write-hot.
4. **At-least-once + idempotency is the honest story** — exactly-once ends at every system boundary. Upsert sinks, idempotency keys, commit-after-processing: *duplicates are recoverable, losses aren't.*
5. **Projections rebuild from truth** — ES, vector indexes, caches, rollups, ClickHouse: CDC-fed copies of a source of record. Dual-writes drift; the WAL/Streams is the one ordered truth; *if you can't rebuild it, you built it wrong.*
6. **State the contract with a number** — replica lag ms–s · cache staleness ≤ TTL · ES searchable ≤ ~1 s + pipeline lag · HLL ±0.8% · recall ≥ 0.95. "Eventually consistent" is hand-waving; a bound is engineering.
7. **Ladders + YAGNI** — Postgres until proven otherwise · poll before push · exact before sketch · batch before stream · shard last. Every specialized box is earned by a *measured number*; say which number would change your mind.

---

## Technologies — the opening line for every box

| Box | Say this when you draw it |
|---|---|
| [Redis](01-redis.md) | In-memory single-threaded structure server — every command atomic, ~100k ops/s/node; trades durability for speed, never the record |
| [Kafka](02-kafka.md) | Partitioned replicated append-only log — order per partition, consumers are offsets, time-based retention makes replay + fan-out free. *2026:* KRaft-only (ZooKeeper removed in 4.0), tiered storage GA (retention decouples from broker count), KIP-848 rebalances, KIP-932 share-group queues |
| [Postgres](03-postgresql.md) | MVCC (readers never block writers) + WAL + constraints — concurrency-correctness in one line; scale by the ladder, shard last |
| [Cassandra](04-cassandra.md) | Masterless ring + LSM — linear write scaling, per-request consistency (R+W>N); the game is query-first partition-key design |
| [DynamoDB](05-dynamodb.md) | Cassandra's model on Paxos leaders — contractual limits, conditional writes as the concurrency story, cost math is design input |
| [Elasticsearch](06-elasticsearch.md) | Distributed Lucene — immutable segments explain everything; CDC-fed read projection, searchable in seconds, never truth |
| [Flink](07-flink.md) | Stateful compute between two logs — checkpoints bind state to offsets so recovery = replay; event time + watermarks make windows honest |
| [ZooKeeper/etcd](08-zookeeper.md) | Consensus-replicated tiny tree + liveness — election, locks, membership; fencing tokens make locks actually safe |
| [API Gateway](09-api-gateway.md) | One sentence *is* the treatment: routing + authn + rate limiting at the door; LB picks which *copy*, gateway picks which *service* |
| [TSDB](10-time-series-databases.md) | Six workload assumptions cashed into 10–100× (Gorilla compression, drop-partition retention); cardinality is the detonator |
| [Vector DB](11-vector-databases.md) | ANN over embeddings — HNSW default, recall is an SLO; traps: filtered search + model upgrades (spaces don't mix) |
| [Sketches](12-big-data-structures.md) | Bloom/HLL/CMS/t-digest — constant memory, one-sided tunable error; *percentiles merge, they don't average* |
| [ClickHouse](20-clickhouse.md) | Columnar MergeTree — reads only queried columns × 10–30× compression × vectorized = 100× on aggregations; the analytics projection, batches only |
| [Temporal](37-temporal-durable-execution.md) | Event sourcing for control flow — workflow code replays from its history so it resumes where it crashed; workflows decide, activities do (idempotent, at-least-once); version before you deploy |
| [CDC](39-change-data-capture.md) | Read the database's own log into Kafka keyed by PK — never dual-write a derived store; version-guarded upserts; *CDC for copies, outbox for intent*; watch the replication slot |

## Patterns — the move in one line

| Pattern | The move |
|---|---|
| [Realtime Updates](13-realtime-updates.md) | Two hops: pub/sub routing + SSE-unless-client-streams-back; *the socket is a hint, the DB is the truth* |
| [Contention](14-pattern-contention.md) | Cheapest mechanism that holds the invariant: conditional write + constraint → row lock → OCC → serialize; *never read-then-decide in app memory* |
| [Multi-step](15-pattern-multi-step-processes.md) | Name where workflow state lives → orchestrated saga + compensation + idempotency keys + outbox; Temporal when flows are long-lived |
| [Scaling Reads](16-pattern-scaling-reads.md) | Do less work → more copies → don't hit the DB (index → replicas → cache → CDN); state the staleness per tier |
| [Scaling Writes](17-pattern-scaling-writes.md) | Right engine → batch → queue the burst + shed → shard by key → salt the whales; each rung changes the client contract |
| [Large Blobs](18-pattern-large-blobs.md) | Metadata in DB, bytes in S3, presigned URLs both ways; pending→ready via storage events; *servers move pointers, never bytes* |
| [Long-Running Tasks](19-pattern-long-running-tasks.md) | 202 + job id → queue → idempotent workers scaled on depth → DLQ with an owner; *the API promises acceptance, not completion* |

## Real systems — one sentence each ([DD 38](38-in-the-wild-case-studies.md))

| Case | Cite it when | The sentence |
|---|---|---|
| Shopify reservations | Inventory / last unit | One row per reservable unit claimed with `SKIP LOCKED`, in the same transaction as the ledger |
| Slack job queue | A queue fell over in a burst | Kafka in front of Redis: backlog on disk, working set in RAM, throttle at the drain |
| Figma multiplayer | Collaborative editing | Server-ordered per-property LWW, fractional indexes, one process per document |
| Spotify data lake | Lookups over analytics data | Bucket + sort + Bloom prune + row-level index instead of copying to a KV store |
| Meta ZGateway | Many clients on a shared DB | Pool, batch, coalesce, cache, admit — and name each trade |
| Discord ([DD 04 §10](04-cassandra.md#10-case-study--how-discord-stores-trillions-of-messages)) | Chat storage at scale | Route by channel + coalesce reads; GC-free engine second |
| GitLab ([Core 11](../Core%20Course/11-reliability.md)) | "How do you know backups work?" | Five unverified mechanisms = zero; automate restore-and-verify |

---

## The numbers

**Single-node throughput:** Redis **~100k ops/s** (Valkey/Redis 8 with I/O threads push toward ~1M) · Kafka **~1M msg/s** (at ~1 KB messages; falls with larger payloads) · Postgres **~5–20k TPS** · Cassandra **~10k writes/s** · connection servers **~100k–1M sockets**.

**Size limits (S3 + pointer above them):** DynamoDB item **400 KB** · Kafka message **<1 MB** · Cassandra partition **<100 MB** · ES shard **10–50 GB** · znode ≤1 MB (KBs in practice) · blob-in-DB threshold **~10 MB**.

**Ceilings & dials:** Dynamo **3k RCU / 1k WCU per key** · ES `from/size` dies **~10k** hits → `search_after` · request timeout **30–60 s** = the sync/async line · Postgres connections = hundreds → PgBouncer.

**Contracts:** AOF loses **~1 s** · replica lag **ms–s** · ES refresh **1 s** · quorum **R+W>N** (RF=3 → 2+2) · HLL **12 KB @ ±0.8%** · Bloom **~10 bits/elem @1% FP** · Gorilla ≈ **1-bit timestamps / 1.4-byte floats** · ClickHouse compression **10–30×**, batches **10k+ rows**.

---

## The four universal probes (ask them of every box you draw)

1. **"What if it dies?"** — the recovery story: what's replayed (WAL/offsets/checkpoint), what promotes (replica/ISR/ensemble), and which *acknowledged* data can vanish (async replication, AOF window, pub/sub).
2. **"How does it scale?"** — the key design + the ladder rung you're on, with a BOTEC showing the requirement inside the envelope.
3. **"Two actors race?"** — atomicity story: single-threaded command / conditional write / row lock / quorum / fencing token.
4. **"One key goes viral?"** — the hot-key medicine (theme 3), picked for read-hot vs write-hot.

---

## Say-verbatim greatest hits

- *"The app check is UX; the constraint is correctness."*
- *"I commit offsets after processing — duplicates are recoverable, losses aren't."*
- *"Redis locks are an efficiency tool that occasionally fails; election without fencing is a race with extra steps."*
- *"Scaling this store is a data-modeling decision — key layout — not a provisioning one."*
- *"Every box I don't add is a failure mode I don't own."*
- *"It's a rebuildable projection; the database and the model/source version are the truth."*
- *"Recall/staleness/error is a dial with an SLO, not a caveat."*
- *"Batch unless someone acts on the result in seconds."*
- *"The socket is a latency optimization; the message table is the product."*
- *"Exact until the memory math breaks — sketches are earned by scale."*

---

## Ten minutes before the interview

1. Re-read this page.
2. Open the **"In the Room"** section of the 2–3 notes matching the likely problem domain (chat → 01/02/04/13 · analytics → 02/07/10/12/20 · marketplace → 03/14/15 · media → 18/19/02).
3. Say the relevant one-liners and verbatim lines **out loud** once — they're written for the mouth, not the eye.
