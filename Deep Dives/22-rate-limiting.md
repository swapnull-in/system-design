# Pattern 22 — Rate Limiting & Admission Control

> The missing deep dive: both a **component you add to every design** ("gateway does authN + rate limiting") and a **standalone prompt** ("design a distributed rate limiter"). Interviewers probe three things: which algorithm and why, how it stays correct when distributed, and what happens when the limiter itself fails. This file covers all three, plus the follow-up minefield.

**Pairs with:** [09 — API Gateway](09-api-gateway.md) · [01 — Redis](01-redis.md) · [11 — Reliability](../Core%20Course/11-reliability.md) · [28 — Designing for Failure](../Core%20Course/28-designing-for-failure.md)

---

## 1. What it's for (say this, not "prevent abuse")

Four distinct jobs — name the one your design needs, because they pick different limits and keys:

1. **Protect your capacity** — keep aggregate load under what you've provisioned (load shedding's polite cousin).
2. **Fairness / noisy neighbor** — one tenant can't starve the rest (per-tenant limits in every multi-tenant system).
3. **Abuse & security** — credential stuffing, scraping, spam (per-IP/per-account, often at the edge/WAF).
4. **Cost control & product tiers** — free = 100 req/min, pro = 10k (per-API-key, billing-adjacent).

> **Rate limit ≠ concurrency limit.** Rate = requests per window (protects *arrival* rate). Concurrency = in-flight requests (protects *slow* downstreams — a low rate of expensive queries can still kill you). Mature systems need both; saying this unprompted is a Staff signal.

## 2. The algorithms — know all five, default to token bucket

| Algorithm | How | Pro | Con |
|---|---|---|---|
| **Fixed window** | counter per window (`user:12:04`), reset each minute | trivial, one counter | **2× burst at boundary** (100 at :59 + 100 at :01) |
| **Sliding window log** | store every request timestamp, count last N sec | exact | memory = O(requests) — dies at scale |
| **Sliding window counter** | current + previous window, weighted overlap | near-exact, 2 counters | approximation (assumes even spread) |
| **Token bucket** | bucket of capacity B, refills r tokens/sec; request takes a token | **allows bursts up to B, enforces sustained rate r** — matches real traffic; O(1) memory (2 numbers: tokens, last-refill) | burst B needs choosing (and defending) |
| **Leaky bucket** | queue drained at fixed rate | perfectly smooth output | queuing adds latency; burst-hostile |

**The decision:** **token bucket** is the industry default (Stripe, AWS API Gateway) — real clients are bursty, and it forgives bursts while holding the average. Use **sliding window counter** when the contract is a hard cap per window (billing/quotas). Mention **GCRA** ("leaky bucket as a timestamp — one stored value, no background refill") as the refined form; it's what Redis-Cell implements.

**Token bucket in four lines** (lazy refill — no timers):

```
on request(key):
  tokens = min(B, tokens + (now - last_ts) * r)   # refill since last seen
  if tokens >= 1: tokens -= 1; allow
  else: deny (retry_after = (1 - tokens) / r)
```

## 3. Distributed enforcement — the real interview

One node is trivial; the probes start when limits must hold **across a fleet**.

**Option A — Global store (Redis), atomic via Lua.** Every request runs the token-bucket math in a Lua script (or `INCR`+`EXPIRE` for windows) — atomicity matters because *read-compute-write from the app races*: two gateways read tokens=1 simultaneously, both allow. Cost: +0.5–1 ms per request, and Redis becomes a dependency on your hot path. Scale: one Redis node ≈ 100k+ ops/sec → **shard by rate-limit key** (natural — each key lives on one shard). Watch the **hot key**: one huge API key hammers one shard; split its bucket or dedicate capacity.

**Option B — Local buckets, divided quota.** Each of N gateways enforces limit/N locally. Zero latency, no dependency — but uneven load distribution makes it inaccurate (LB skew → false throttling), and N changes on every deploy/scale event.

**Option C — Local + async sync (the production answer).** Enforce locally; reconcile with the global store in the background — or **batch-borrow**: each node takes 100 tokens from Redis at once, hands them out locally, returns for more. One Redis trip per 100 requests instead of per request. Accuracy is now *eventual* — brief over-admission during bursts.

> **The trade to state unprompted:** "Exact global limits cost a synchronous hop on every request; local enforcement is free but approximate. I'd go local-with-borrowing and accept ~5% overshoot — rate limits protect capacity, they're not billing invariants." (If it *is* billing/quota → exact, Option A.) Multi-region: rate-limit **per region** against regional quotas — a globally synchronous limiter imports cross-region latency into every request for accuracy nobody needs.

## 4. The response contract — where retry storms are born

- Reject with **429** (or **503 + Retry-After** for "server overloaded" semantics), include **`Retry-After`** and quota headers (`X-RateLimit-Limit/Remaining/Reset` — Stripe/GitHub style) so well-behaved clients self-regulate.
- **The trap:** naive clients retry immediately on 429 → the limiter *amplifies* load instead of shedding it. Answer: clients retry with **exponential backoff + jitter**; your own SDKs must do this; at the edge, egregious offenders get dropped (no response) rather than politely 429'd.
- **Shed early, shed cheap:** rate-limit at the outermost viable layer (WAF/CDN → gateway → service). A request rejected at the edge costs ~nothing; the same rejection inside the service already spent connection, TLS, auth, and a thread.
- Degrade by priority under global pressure: shed anonymous/read traffic before authenticated/write traffic (admission control, [SD 28](../Core%20Course/28-designing-for-failure.md)).

## 5. Keying & rules

Limits stack — a request passes **all** applicable buckets: per-IP (edge, abuse) → per-API-key/user (gateway, tiers) → per-tenant (fairness) → per-endpoint (expensive routes get tighter caps) → global (capacity floor). Rules live in config (key pattern → algorithm, rate, burst), hot-reloadable, owned by the gateway/sidecar — services shouldn't hand-roll their own.

## 6. Failure modes — the follow-up list

| Probe | The answer |
|---|---|
| "Redis is down — now what?" | **Fail-open vs fail-closed is per-endpoint policy**: fail open for user-facing reads (availability > enforcement), fail closed for auth/login/payment endpoints (enforcement IS the security). Local fallback limits (each node enforces a conservative cap) cover the gap. Never let the limiter be a single point of failure for the whole API. |
| "Two gateways race on the same key" | Atomicity in the store (Lua/CAS), not in the app — or accept approximation via borrowing. |
| "One API key is 50% of traffic" | Hot shard: split that key's bucket into sub-buckets (sum on check), cache a local allowance, or move the whale to dedicated capacity. Same move as any hot partition ([17](17-pattern-scaling-writes.md)). |
| "Clock skew between nodes?" | Use the *store's* clock (Redis `TIME` inside the Lua script), never per-node wall clocks. |
| "Thundering herd when a window resets" | That's fixed-window's boundary burst — token bucket/GCRA has no reset instant. If stuck with windows, jitter the reset per key. |
| "How do users know their quota?" | Headers + a quota endpoint + docs; 429 with `Retry-After` is part of the API contract, not an error to hide. |

## 7. "Design a rate limiter" — the 5-minute skeleton

1. **Requirements:** rules (per-key, per-tier), scale (say 1M rps across 50 gateways), latency budget (<1 ms added), accuracy tolerance (protective ≈ ±5% OK; quota = exact), multi-region?
2. **Placement:** library/sidecar in the gateway (not a separate synchronous service — that just adds a hop that can fail).
3. **Algorithm:** token bucket, lazy refill, per-key state = 2 numbers.
4. **State:** Redis cluster sharded by key; Lua for atomicity; local borrowing for the hot path (state the accuracy trade).
5. **Deep dives they'll pick:** hot key · Redis failure policy (open/closed per endpoint) · retry storm behavior · multi-region (regional quotas).
6. **Numbers to anchor:** Redis op sub-ms · ~100k ops/sec/node · token-bucket state ~40 bytes/key → 100M active keys ≈ 4 GB — memory is not the problem; the synchronous hop is.

## Interview cheat-sheet

- Four jobs: capacity, fairness, abuse, tiers — name yours; rate ≠ concurrency limit.
- **Token bucket by default** (burst B, sustained r, 2 numbers, lazy refill); sliding-window counter for hard quotas; GCRA = the elegant form.
- Distributed: **atomic in Redis via Lua** for exact; **local + batch-borrowing** for fast; state the ±% you accept.
- 429 + Retry-After + quota headers; backoff-with-jitter or the limiter *creates* the storm; shed at the edge, not in the service.
- Redis down → fail-open for reads, fail-closed for auth/money, local fallback caps either way.
- Hot API key = hot partition — split, cache, or isolate the whale.

**Sources:** Stripe rate-limiting engineering post · Cloudflare "How we built rate limiting" (sliding window counter) · Redis-Cell/GCRA docs · Alex Xu Vol 1 ch 4 — all consistent with the above; the note is self-sufficient.
