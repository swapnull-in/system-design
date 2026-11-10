# Pattern 21 — Pattern Selection

> The meta-pattern: hearing a prompt and recognizing **which** of the seven patterns it actually is — in the first two minutes, out loud. Every pattern file (13–19) tells you how to execute a move; this one tells you which move the prompt is asking for. Misdiagnose here and everything downstream is well-executed waste.

---

## The 60-second triage — four questions

Run these against any prompt, in order. Each answer eliminates patterns.

1. **What breaks first at 10× — reads, writes, or one row?**
   Reads → [16](16-pattern-scaling-reads.md). Writes everywhere → [17](17-pattern-scaling-writes.md). Everyone fighting over the *same* row/seat/item → [14 Contention](14-pattern-contention.md). This is the most important fork: **hot everywhere is a scaling problem; hot in one place is a contention problem** — their fixes are opposites (spread vs serialize).

2. **Does the invariant live inside one database or across systems?**
   One DB ("stock ≥ 0") → contention primitives: conditional write, lock, OCC ([14](14-pattern-contention.md)). Across services/providers ("payment charged ⇔ order exists") → saga + compensation + idempotency ([15](15-pattern-multi-step-processes.md)). Never answer a cross-system invariant with a lock.

3. **Does the user wait for the result, or come back for it?**
   Comes back / result takes seconds-to-hours (transcode, report, bulk import) → [19 Long-Running Tasks](19-pattern-long-running-tasks.md): 202 + job id + queue. Must *see changes as they happen* (chat, live comments, dashboards) → [13 Realtime Updates](13-realtime-updates.md). These get confused constantly — see misdiagnoses.

4. **Is the data big in bytes or hot in operations?**
   Big bytes (photos, video, files) → [18 Large Blobs](18-pattern-large-blobs.md): metadata in DB, bytes in S3, presigned URLs. Hot ops → back to question 1. "Where **near me**…" → geospatial, which is its own index problem: [Module 29](../Core%20Course/29-geospatial-systems.md).

## The cue table — phrase → pattern → first move

| When the prompt says… | Pattern | Your file | Say this first |
|---|---|---|---|
| "feed/page is slow", "millions of DAU browsing", read:write ≫ 10:1 | Scaling Reads | [16](16-pattern-scaling-reads.md) | "Read-dominated — index and shape the query first, then replicas, then cache. In that order." |
| "ingest clicks/telemetry/locations", "100k events/sec" | Scaling Writes | [17](17-pattern-scaling-writes.md) | "Write-dominated — right engine, batch, buffer the burst in a queue, then shard by a key that spreads load." |
| "last item", "same seat", "flash sale", "auction", "double-booking" | Contention | [14](14-pattern-contention.md) | "The crux is one contended invariant — cheapest mechanism that holds it: conditional write → lock → OCC → serialize through a queue." |
| "checkout", "booking flow", "money moves", "order → payment → shipping" | Multi-Step Processes | [15](15-pattern-multi-step-processes.md) | "Multiple systems must agree — saga with compensation, idempotency keys everywhere, reconciliation as the backstop." |
| "see it live", "typing indicator", "live scores/comments/presence" | Realtime Updates | [13](13-realtime-updates.md) | "Two hops: how updates route between servers (pub/sub), and server→client transport — SSE unless the client streams back." |
| "upload video/photos/documents", "process the file" | Large Blobs | [18](18-pattern-large-blobs.md) | "Bytes never touch my servers — presigned upload to blob storage, metadata row in the DB, CDN out." |
| "generate report", "transcode", "send a million emails", "import CSV" | Long-Running Tasks | [19](19-pattern-long-running-tasks.md) | "Accept fast, work async: 202 + job id, queue, idempotent workers, status endpoint, DLQ." |
| "nearby drivers/restaurants/friends", "within 5 km" | Proximity | [Module 29](../Core%20Course/29-geospatial-systems.md) | "Geo index choice is the crux — geohash/S2/quadtree — and whether the points move." |

## The five classic misdiagnoses

1. **"Slow page" that's actually one hot row.** You reach for replicas/caches ([16](16-pattern-scaling-reads.md)) but the symptom is lock waits on a single record — it's [14 Contention](14-pattern-contention.md). Tell: does slowness correlate with *specific items* (the drop, the celebrity) rather than overall traffic?
2. **"Millions of writes" that are really one hot counter.** Sharding the table ([17](17-pattern-scaling-writes.md)) doesn't help when every write hits the same likes-counter row — that's contention's hot-key corner: shard the *counter*, batch, or go async. Tell: write volume concentrated on few keys vs spread.
3. **WebSockets for job status.** "User uploads video and sees progress" is *not* realtime updates — it's a long-running task ([19](19-pattern-long-running-tasks.md)); polling a status endpoint every few seconds is the right answer, and proposing WebSocket infra for it reads as poor judgment. Reserve [13](13-realtime-updates.md) for many-to-many, second-by-second change.
4. **A lock across services.** "Reserve inventory while payment processes" tempts a distributed lock — wrong layer; it's a [15 saga](15-pattern-multi-step-processes.md) with a reservation + TTL (a *state*, not a lock). Locks live inside one database; across systems you have compensations.
5. **Blob bytes through the API tier.** Any design that streams file bytes through your app servers fails [18](18-pattern-large-blobs.md) — the tell is upload/download endpoints doing anything but minting presigned URLs.

## Real prompts are 2–4 patterns — name the crux

Every famous prompt composes patterns. Say the decomposition out loud in minute two, then spend your depth on the **crux** (the one the interviewer picked the prompt *for*):

| Prompt | Patterns present | The crux |
|---|---|---|
| Ticketmaster | contention + scaling reads + realtime seat map | **Contention** (14) |
| Uber | proximity + scaling writes (locations) + realtime + multi-step (trip) | **Proximity** (29) |
| Instagram / Twitter feed | scaling reads + blobs + realtime-ish | **Scaling reads via fan-out** (16) |
| YouTube | blobs + long-running (transcode) + scaling reads | **Blobs + pipeline** (18 + 19) |
| Chat (WhatsApp/Slack) | realtime + scaling writes + inbox reads | **Realtime delivery** (13) |
| E-commerce checkout | multi-step + contention (inventory) + scaling reads (catalog) | **Multi-step + contention** (15 + 14) |
| Metrics/analytics (Datadog-ish) | scaling writes + long-running (rollups) + scaling reads | **Scaling writes** (17) |
| Google Docs | realtime + contention (concurrent edits → OT/CRDT) | **Realtime co-editing** (13 + 14) |

The senior-cap failure isn't picking the wrong pattern — it's executing all four shallowly. The Staff move: "There are four problems here; three are standard — *[name them and their one-line moves]* — the interesting one is X, let's go deep there."

## Day-of-interview compression

Four questions: **reads, writes, or one row? · invariant in one DB or across systems? · wait or come back? · big bytes or hot ops?** — then name the crux pattern, state its one-line move from the [cheat sheet](CHEATSHEET.md), and dive.

**Source:** [Hello Interview — Choosing the Right Pattern](https://www.hellointerview.com/learn/system-design/patterns/pattern-selection) · Companion to patterns [13](13-realtime-updates.md)–[19](19-pattern-long-running-tasks.md)
