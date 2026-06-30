# Knowledge Base — URL Shortener Study Guide

---

## 1. L7 Rate Limiting (NGINX)

**L7 = Layer 7 = Application Layer** of the OSI networking model.

The OSI model has 7 layers:

| Layer | Name         | What it sees                              |
|-------|--------------|-------------------------------------------|
| 1     | Physical     | raw bits/cables                           |
| 2     | Data Link    | MAC addresses, frames                     |
| 3     | Network      | IP addresses                              |
| 4     | Transport    | TCP/UDP ports                             |
| 5–6   | Session/Pres | (rarely distinct in practice)             |
| **7** | **Application** | **HTTP methods, URLs, headers, cookies** |

**L4 rate limiting** (IP + port) is blunt: it only knows _who_ is connecting.  
**L7 rate limiting** is smart: NGINX can inspect the HTTP request and apply different limits per URL path, per method, per header value, etc.

In this project, NGINX applies **different rate limits per endpoint**:
- `/api/v1/data/shorten` → 10 req/s (write, expensive)
- `/*` redirect → 100 req/s (read, cheap)

That differentiation is only possible at L7 — at L4 all those requests look the same (same IP, same port 80/443).

**Why it matters:** Rate limiting in NGINX happens before the request reaches Node.js. Rejected requests (`429`) never consume an event loop tick. L7 lets us protect the expensive write path more aggressively than the read path.

**References:**
- [NGINX `limit_req` module docs](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- [CloudFlare: What is the OSI Model?](https://www.cloudflare.com/learning/ddos/glossary/open-systems-interconnection-model-osi/)
- [NGINX blog: Rate Limiting with NGINX](https://www.nginx.com/blog/rate-limiting-nginx/)

---

## 2. p99 Latency — Deep Dive

### What a percentile actually is

Sort all latency measurements from fastest to slowest. The **Nth percentile** is the value at position N% in that sorted list.

```
10 requests, sorted: [1, 1, 2, 2, 3, 3, 4, 5, 8, 9000] ms

p50 = value at position 50% → 5th value  = 3 ms   (half are faster, half slower)
p90 = value at position 90% → 9th value  = 8 ms
p99 = value at position 99% → last 1%    = 9000 ms
avg = (1+1+2+2+3+3+4+5+8+9000) / 10     = 902.9 ms  ← completely useless
```

The average is 902 ms but 9 out of 10 users experienced ≤ 8 ms. The single 9-second outlier obliterates the average. **Averages lie. Percentiles tell the truth.**

---

### Why the average is structurally broken for latency

The mean works well for symmetric distributions (like height of people). Latency is **not symmetric** — it has a hard floor (can't be negative) and a long tail on the right (a request can take arbitrarily long due to GC pause, lock contention, cold disk read, etc.).

```
Symmetric (mean works):          Latency (mean broken):

     ▐█▌                             ▐█▌
    ▐███▌                           ▐███▌
   ▐█████▌                         ▐█████▌─────────── long tail ──────►
  ◄──────────►                    ◄──────────────────────────────────►
     mean ≈ median                  mean >> median
```

This distribution shape is called **right-skewed**. In skewed distributions, the mean is pulled toward the tail. One request that takes 30 seconds can raise the average of 10,000 requests from 2 ms to 5 ms — invisibly hiding the catastrophic outlier.

---

### The percentile ladder

| Percentile | Plain English                             | Who experiences it                          |
|------------|-------------------------------------------|---------------------------------------------|
| p50        | Median — the "typical" user               | Half of all users                           |
| p75        | 3 out of 4 users are at least this fast   | Three-quarters of users                     |
| p90        | 9 out of 10 users are at least this fast  | Near-typical                                |
| p95        | 19 out of 20 users                        | What most SLAs start at                     |
| p99        | 99 out of 100 users                       | Where most production SLAs are defined      |
| p999       | 999 out of 1000 users                     | 1 in 1000 users hit this — at 10k req/s, that's 10 users/s |
| p9999      | 9999 out of 10000 users                   | Rare but real; GC pauses often show here    |

**Choosing which percentile to care about** depends on traffic volume:

- At 1 req/s: p99 means 1 bad request every 100 seconds → not critical
- At 10,000 req/s: p99 means **100 bad requests per second** → engineers are paged
- At 10,000 req/s: p999 means **10 bad requests per second** → still real users affected

---

### The "tail at scale" problem

This is where percentiles get genuinely surprising. Imagine a single backend call has p99 = 1 ms. Good. Now you build a feature that makes **100 parallel backend calls** and waits for all of them.

**What's the p99 of the combined response?**

The probability that at least one of 100 calls hits the p99 tail:

```
P(at least one slow) = 1 - P(all fast)
                     = 1 - (0.99)^100
                     = 1 - 0.366
                     = 63.4%
```

**63% of user requests will be slow** even though each individual call is only slow 1% of the time. The more dependencies you fan out to, the worse this gets.

| Calls in parallel | Probability at least one hits p99 tail |
|-------------------|-----------------------------------------|
| 1                 | 1%                                      |
| 10                | 9.6%                                    |
| 100               | 63.4%                                   |
| 1000              | 99.996%                                 |

This is why Google's "Tail at Scale" paper (2013) is foundational — it explains why **large distributed systems almost always feel slow to the end user** even when each individual service looks healthy in isolation.

**Mitigation strategies:**
- **Hedged requests**: send the same request to two replicas, use whichever responds first, cancel the other
- **Timeout budgets**: each downstream call gets a time budget; if it exceeds it, use a cached/default value
- **Eliminate fan-out**: design APIs to avoid requiring many parallel calls per user request

---

### How percentiles are computed in practice

**Exact method:** sort all values, index into the array. Requires storing every measurement. For 1 million requests/day = 1M numbers in memory. Impractical at scale.

**Histogram approximation (what Prometheus uses):**

Prometheus `http_duration_seconds` in this project is a **histogram**. Instead of storing each value, it counts how many observations fell into pre-defined buckets:

```
bucket[0, 0.005]   = 9500   (requests taking 0–5 ms)
bucket[0, 0.01]    = 9850   (requests taking 0–10 ms, cumulative)
bucket[0, 0.025]   = 9980
bucket[0, 0.05]    = 9999
bucket[0, +Inf]    = 10000  (total)
```

To estimate p99: find the smallest bucket upper bound where cumulative count ≥ 99% of total. Interpolate linearly within that bucket.

**The tradeoff:** Prometheus histograms are cheap (fixed memory regardless of traffic) but only as accurate as the bucket boundaries you define. If your actual p99 is between two bucket boundaries, you get a linear interpolation estimate, not an exact value.

**HDR Histogram** (High Dynamic Range) is an alternative that uses logarithmic buckets to maintain accuracy across a wide range (microseconds to seconds) with very low memory. Used in tools like wrk2, HdrHistogram.js.

---

### Prometheus histogram — how this project exposes latency

In `src/observability/metrics.ts`, `http_duration_seconds` is defined as a histogram with default Prometheus bucket boundaries. Grafana queries it with:

```
histogram_quantile(0.99, rate(http_duration_seconds_bucket[5m]))
```

This computes the p99 over a rolling 5-minute window. The `rate()` gives per-second rates of each bucket, and `histogram_quantile` does the interpolation.

**What the Grafana dashboard shows:**
- `histogram_quantile(0.5, ...)` → p50 (median latency)
- `histogram_quantile(0.95, ...)` → p95
- `histogram_quantile(0.99, ...)` → p99 ← this is the SLO target

---

### Why p99 < 10 ms when cache is warm

The tail at p99 is dominated by whatever happens to the slowest 1% of requests. When Redis is warm:

- ~99%+ of redirects: Redis hit → ~0.5–2 ms → never reach MongoDB
- ~1% of redirects: cache miss or Redis hiccup → MongoDB → ~5–20 ms

The p99 captures the worst of the cache-hit path plus the best of the fallback path. With a warm cache, even the p99 stays within the in-process + Redis round-trip budget.

When Redis is cold (just restarted), every request hits MongoDB. The p99 jumps to MongoDB's tail (~20–50 ms). This is the "degrades gracefully" scenario from section 3 — latency degrades, correctness doesn't.

---

**References:**
- [The Tail at Scale — Google (ACM, 2013)](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext)
- [Percentile latency — Brendan Gregg](https://www.brendangregg.com/FrequencyTrails/modes.html)
- [Google SRE Book: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Prometheus: Histograms and Summaries](https://prometheus.io/docs/practices/histograms/)
- [HdrHistogram — Gil Tene](http://hdrhistogram.org/)
- [Cloudflare: How we think about percentiles](https://blog.cloudflare.com/the-problem-with-averages/)

---

## 3. Circuit Breaker & Graceful Degradation — Deep Dive

### The core problem: cascading failures

Without a circuit breaker, a failing dependency can take down your entire system — not through direct failure, but through **cascading failure**:

```
Redis goes down
  → every cache call blocks waiting for timeout (e.g. 5s)
  → redirect requests pile up, each holding a thread/connection
  → connection pool exhausts
  → new requests immediately error with "no connections available"
  → your healthy service is now returning 500s because of a cache
```

The timeout is the killer. If Redis times out after 5 seconds and you have 1000 concurrent users, you've accumulated 5000 seconds of blocked work. The circuit breaker solves this by **failing fast** — instead of waiting 5 seconds per call, you return `null` in microseconds once the circuit is open.

---

### The three-state machine

A circuit breaker is literally a finite state machine with three states:

```
                   ┌──────────────────────────────────┐
                   │                                  │
              threshold                         probe succeeds
              exceeded                               │
                   │                                  │
                   ▼                                  │
┌──────────┐    ┌──────┐    timeout elapsed    ┌────────────┐
│  CLOSED  │───►│ OPEN │───────────────────────► HALF-OPEN  │
│(normal)  │    │(fail │                       │(testing)   │
│          │◄───│ fast)│◄──────────────────────│            │
└──────────┘    └──────┘    probe fails        └────────────┘
     ▲
     │  errors below threshold → stay closed
     └──────────────────────────────────────
```

**CLOSED (normal operation)**
- All calls pass through to the dependency
- Errors are counted in a sliding window
- When error count/rate exceeds threshold → transition to OPEN

**OPEN (failing fast)**
- All calls are immediately rejected without touching the dependency
- Returns a fallback value (null, cached response, default) or throws immediately
- After a configured timeout (e.g. 30s) → transition to HALF-OPEN

**HALF-OPEN (probing recovery)**
- A limited number of "probe" requests are allowed through
- If they succeed → transition to CLOSED (service recovered)
- If they fail → transition back to OPEN (restart the timer)

---

### Two trigger strategies

**Count-based window:** open after N consecutive failures.
```
errors: [ok, ok, FAIL, FAIL, FAIL, FAIL, FAIL] → threshold=5 → OPEN
```
Simple but susceptible to noise. One bad batch of 5 requests trips the breaker even if overall error rate is 0.1%.

**Rate-based sliding window (preferred):** track error rate over the last N seconds or N calls.
```
last 100 calls: 95 ok, 5 fail → error rate = 5%
threshold = 10% → stay CLOSED

last 100 calls: 85 ok, 15 fail → error rate = 15%
threshold = 10% → OPEN
```
More stable. Netflix Hystrix and Resilience4j both use this. Avoids tripping on short bursts.

**Minimum call volume guard:** even with rate-based, don't open the circuit after 1/1 failures (100% error rate but only 1 call). Require a minimum volume before the rate matters:
```
if (total_calls < 20) → stay CLOSED regardless of error rate
```

---

### How this project implements it (simplified variant)

This project's circuit breaker is deliberately minimal — it uses ioredis's own connection lifecycle events instead of tracking error rates manually:

```typescript
// UrlCache constructor
redis.on('error', () => {
  this.available = false;  // circuit OPENS immediately on any error
});
redis.on('ready', () => {
  this.available = true;   // circuit CLOSES when ioredis confirms reconnect
});
```

Every operation checks the flag before touching Redis:
```typescript
async get(shortCode: string): Promise<string | null> {
  if (!this.available) return null;  // fail fast — microseconds
  try {
    return await this.redis.get(`url:${shortCode}`);
  } catch {
    return null;  // defensive: shouldn't happen when available=true, but safe
  }
}
```

**Why this works here:** ioredis already manages reconnection with exponential backoff. It emits `error` when a connection attempt fails and `ready` when it succeeds. Delegating state tracking to ioredis avoids re-implementing reconnection logic. The circuit is effectively OPEN while ioredis is reconnecting and CLOSED once it's ready.

**What this skips vs. a full circuit breaker:**
- No HALF-OPEN state — ioredis probes internally; `available` goes straight to `true` on `ready`
- No rate-based threshold — any error opens (appropriate here: Redis is binary, not degraded)
- No per-operation timeout — ioredis timeout is configured at the client level

---

### When to apply a circuit breaker

Apply when **all three** are true:

1. **The dependency is external** (network call: database, API, cache, queue). Pure in-process failures don't need it.
2. **Failure mode is slow** (timeouts, not instant errors). Fast failures don't cascade — slow ones do.
3. **A fallback exists** (cached data, default value, degraded mode, queue for later). Without a fallback, the circuit breaker just turns a slow failure into a fast one — still a failure.

**Good candidates:**
- Redis / Memcached (cache — fallback: DB)
- Payment processor (fallback: queue the attempt, show "processing" to user)
- Email service (fallback: queue for retry)
- Third-party analytics API (fallback: drop the event, log it)
- Downstream microservice (fallback: return cached/default response)

**Bad candidates (don't need it):**
- Your own in-process code — no network, no timeout risk
- Database as source of truth with no fallback — opening the circuit just returns errors anyway
- Short-lived batch jobs — circuit breaker state doesn't persist between runs

---

### Tradeoffs

| Concern | Without circuit breaker | With circuit breaker |
|---------|------------------------|----------------------|
| Failure mode | Slow (timeouts accumulate) | Fast (immediate rejection) |
| Recovery | Automatic when dependency recovers | Requires HALF-OPEN probe cycle (adds a delay) |
| Fallback required | No (but you'll return errors either way) | Yes — must define what to return when open |
| Complexity | Low | Moderate (state machine, thresholds, timers) |
| False positives | N/A | Can open on transient spikes, rejecting valid requests |
| Observability | Easy (just check errors) | Must instrument state transitions |

**The false positive problem:** if your error threshold is too aggressive, a brief network hiccup opens the circuit and you start rejecting healthy requests unnecessarily. Solutions:
- Raise the minimum call volume before tripping
- Use a longer sliding window
- Use HALF-OPEN with gradual traffic increase (canary probe) rather than binary pass/fail

---

### Circuit breaker vs. related patterns

These are often confused. They solve different problems and are frequently used together:

| Pattern | What it does | When to use |
|---------|--------------|-------------|
| **Circuit breaker** | Stops calling a failing dependency; fails fast | Dependency is slow/down |
| **Retry with backoff** | Tries again after a failure | Transient errors (momentary blip) |
| **Timeout** | Abandons a call that takes too long | Unbounded wait prevention |
| **Bulkhead** | Isolates thread/connection pools per dependency | One slow dep shouldn't block others |
| **Fallback** | Returns alternative data when primary fails | Any failure that needs a response |

**Combining them correctly:**

```
Request
  → Timeout (don't wait forever)
      → Circuit breaker (don't try if already failing)
          → Retry with backoff (try a few times for transient errors)
              → Bulkhead (cap concurrent calls to this dep)
                  → Actual call
                      → Fallback (return default if all else fails)
```

Retry without a circuit breaker is dangerous: 100 clients each retrying 3 times = 300× amplified load on an already-struggling service.

---

### Production implementations

| Library | Language | Notes |
|---------|----------|-------|
| [opossum](https://nodeshift.dev/opossum/) | Node.js | Most popular; full state machine, metrics, events |
| [Resilience4j](https://resilience4j.readme.io/docs/circuitbreaker) | Java | Successor to Hystrix; count + rate-based windows |
| [Polly](https://github.com/App-vNext/Polly) | .NET | Policy composition: retry + CB + timeout + bulkhead |
| [Hystrix](https://github.com/Netflix/Hystrix) | Java | Netflix original; in maintenance mode, use Resilience4j |
| [pybreaker](https://github.com/danielfm/pybreaker) | Python | Simple, production-used |

**opossum example** (what a full circuit breaker looks like in Node.js):
```typescript
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(redisGet, {
  timeout: 3000,           // 3s → timeout = OPEN
  errorThresholdPercentage: 50,  // 50% error rate → OPEN
  resetTimeout: 30000,     // 30s in OPEN before HALF-OPEN probe
  volumeThreshold: 10,     // minimum 10 calls before rate is evaluated
});

breaker.fallback(() => null);   // return null when OPEN (same as cache miss)

breaker.on('open',     () => logger.warn('circuit opened'));
breaker.on('halfOpen', () => logger.info('circuit probing'));
breaker.on('close',    () => logger.info('circuit closed'));
```

This project uses a manual implementation instead — simpler and sufficient for a binary dependency like Redis.

---

### Observability for circuit breakers

A circuit breaker that opens silently is dangerous — you might not know you're running without your cache for hours. Instrument:

1. **State transitions** → log + counter on OPEN/HALF-OPEN/CLOSE
2. **Rejection rate** → how many calls were short-circuited per second
3. **Fallback invocation rate** → what percentage of requests used the fallback
4. **Error rate by dependency** → per-service breakdown to isolate which dep is failing

In this project, `redis_errors_total` in Prometheus partially covers this. A full implementation would also export a `cache_circuit_state` gauge (0=closed, 1=open, 2=half-open).

---

**References:**
- [Martin Fowler: Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Release It! — Michael Nygard (book)](https://pragprog.com/titles/mnee2/release-it-second-edition/) — coined the term in software
- [AWS: Circuit Breaker pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/circuit-breaker.html)
- [Netflix Tech Blog: Making Netflix API More Resilient](https://netflixtechblog.com/making-the-netflix-api-more-resilient-a8ec62159c2d)
- [Resilience4j: Circuit Breaker docs](https://resilience4j.readme.io/docs/circuitbreaker)
- [opossum: Node.js circuit breaker](https://nodeshift.dev/opossum/)

---

## 4. MongoDB Write Concern — `w:majority` vs `w:1`

MongoDB can run as a **replica set**: multiple servers that all hold copies of the data (1 primary + N secondaries). Writes go to the primary; secondaries replicate asynchronously.

**Write concern** controls when MongoDB acknowledges a write back to the client:

| Write Concern | Acknowledged when…                                           | Risk if primary crashes |
|---------------|--------------------------------------------------------------|------------------------|
| `w:1`         | Primary has written to memory (not yet replicated)           | Data may be lost (primary crashed before secondaries caught up) |
| `w:majority`  | A majority of replica set members have written               | Data is durable — even if primary crashes, secondaries have it |

**In this project:**

- `mainClient` uses `w:majority` — URL records and the ID counter must be durable. Losing the counter would cause ID collisions on the next sequence reset.
- `analyticsClient` uses `w:1` — click events are best-effort. Losing a click record on a crash is acceptable. The lower write concern = lower latency for the analytics write path.

**Simple analogy:** `w:majority` is like a notary who sends a copy to three offices before signing. `w:1` is like the notary signing immediately and sending copies later.

**References:**
- [MongoDB: Write Concern docs](https://www.mongodb.com/docs/manual/reference/write-concern/)
- [MongoDB: Replica Set Write Concern](https://www.mongodb.com/docs/manual/core/replica-set-write-concern/)

---

## 5. LFU Eviction in Redis

When Redis runs out of memory, it must evict (delete) some keys to make room for new ones. The **eviction policy** decides _which_ keys to delete.

**LFU = Least Frequently Used.** Redis tracks how often each key has been accessed. When memory is full, it evicts keys with the _lowest access frequency_ — keys that have been used least often.

**Other eviction policies for comparison:**

| Policy         | Evicts…                                                      |
|----------------|--------------------------------------------------------------|
| `noeviction`   | Nothing — returns error on new writes when full              |
| `allkeys-lru`  | Least Recently Used — the key idle the longest               |
| `allkeys-lfu`  | Least Frequently Used — the key accessed least often (used here) |
| `allkeys-random` | A random key                                               |
| `volatile-*`   | Same as above but only among keys with a TTL set             |

**Why LFU over LRU for this use case?**

URL access follows a **power law**: ~20% of URLs get ~80% of traffic (think: a viral tweet vs. a one-off link). With LRU, a popular URL that wasn't accessed in the last hour could be evicted even if it's requested millions of times per day. LFU keeps high-frequency keys even during quiet periods.

Example:
- Key A: accessed 10,000 times this week, not accessed in the last 30 min
- Key B: accessed 1 time, 5 minutes ago

LRU evicts A (older last-access). LFU evicts B (lower total frequency). LFU is correct here.

**Redis LFU internals:** Redis doesn't keep exact counts. It uses a probabilistic counter called a **Morris counter** (logarithmic approximation) that fits in 8 bits. This keeps overhead tiny.

**References:**
- [Redis: Using Redis as an LFU cache](https://redis.io/docs/manual/eviction/#using-redis-as-an-lfu-cache)
- [Redis eviction policies — full list](https://redis.io/docs/manual/eviction/)
- [Wikipedia: Least Frequently Used](https://en.wikipedia.org/wiki/Least_frequently_used)

---

## 6. Exponential Backoff & Thundering Herd

### Exponential Backoff

When a connection fails, the client waits before retrying. **Exponential backoff** means the wait time doubles with each failed attempt:

```
attempt 1 → wait 100ms
attempt 2 → wait 200ms
attempt 3 → wait 400ms
attempt 4 → wait 800ms
...
attempt N → wait min(100ms × 2^N, 30s)
```

The `min(..., 30s)` cap prevents the wait from growing to infinity. After some point, it's pointless to wait longer than 30 seconds.

**Why not just retry immediately?** If something is down, hammering it with retries every millisecond makes the problem worse. Rapid retries:
1. Waste CPU/network on both sides
2. Can prevent the failing service from recovering (it's busy handling retry storms)

### Thundering Herd Problem

Imagine 500 app instances all connected to Redis. Redis crashes and restarts 10 seconds later.

**Without backoff:** All 500 instances detect the reconnect at nearly the same moment and all fire their connection requests simultaneously. Redis, just starting up, receives 500 connections at once — potentially crashing again or becoming very slow.

**With exponential backoff + jitter:** Each instance waited a different amount of time before retrying (because the backoff is staggered and jitter adds randomness). Reconnections arrive spread across several seconds — Redis handles them comfortably.

The backoff formula in this project:
```
delay = min(100ms × 2^attempts, 30_000ms)
```

**Jitter** (random variance added to the delay) is a common enhancement that further spreads reconnection attempts across the cluster. ioredis adds a small amount automatically.

**References:**
- [AWS: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Wikipedia: Thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)
- [ioredis retry strategy docs](https://github.com/redis/ioredis#auto-reconnect)

---

## 7. Fire-and-Forget — Not Awaiting a Promise (Deep Dive)

### The mechanics: what actually happens at runtime

In JavaScript/TypeScript, async operations return **Promises**. There are two ways to call them:

**Awaited:**
```typescript
const result = await doSomething(); // current function suspends here
res.json({ ok: true });             // runs AFTER doSomething() resolves
```

**Not awaited (fire-and-forget):**
```typescript
doSomething();                      // Promise created, I/O submitted to event loop
res.json({ ok: true });             // runs IMMEDIATELY on next line — no wait
// doSomething() settles later, on a future event loop tick
```

The key: `await` does **not** block the thread. It suspends the current `async` function and yields control back to the event loop, which runs other callbacks. When the Promise resolves, the function resumes. Not awaiting skips the suspend entirely — the current function never pauses.

**Node.js event loop sketch:**

```
Tick 1:  Handle incoming HTTP request
           → call redirect controller
           → send HTTP response  ✓
           → submit MongoDB insert to I/O queue (not awaited)
           → controller returns

Tick 2–N: Other requests handled

Tick N+1: MongoDB insert I/O completes
           → .catch() callback runs if error
           → done
```

The response is already gone by tick N+1. The insert and the response are **concurrent**, not sequential.

---

### This project's implementation

```typescript
// redirect.controller.ts
res.redirect(config.redirectCode, longUrl); // HTTP 302 sent to client NOW

clickRepo
  .insert({ shortUrl, timestamp, ip, userAgent, referrer })
  .catch((err) => {
    req.log.warn({ err }, "click write failed");
    clickWriteErrors.inc();  // Prometheus counter — makes silence visible
  });
// controller returns. insert is still running in background.
```

Three things happening here:
1. Response sent — user's browser starts following the redirect
2. MongoDB insert starts — async, non-blocking
3. `.catch()` registered — errors surface in logs and metrics, never swallowed

---

### Why `.catch()` is not optional

An unawaited Promise with no error handler that rejects will emit `unhandledRejection` on the Node.js process. In Node 15+, this **terminates the process by default**. In earlier versions it's a warning, but the default will eventually become a crash everywhere.

```typescript
// WRONG — if insert() throws, process crashes or warns
clickRepo.insert({ ... });

// CORRECT — error handled, process stays up
clickRepo.insert({ ... }).catch((err) => logger.warn({ err }, "click failed"));

// ALSO CORRECT — explicit void signals intentional fire-and-forget to linters
void clickRepo.insert({ ... }).catch((err) => logger.warn({ err }, "click failed"));
```

TypeScript with `@typescript-eslint/no-floating-promises` will flag unawaited Promises without `void` — the `void` operator explicitly signals "I know this is unawaited and I'm OK with that."

---

### The Node.js event loop — what makes fire-and-forget work

Node.js is single-threaded but non-blocking. All I/O (network, disk, timers) is handled by **libuv** — a C library that manages a thread pool and OS-level async I/O under the hood. Node's JS thread never blocks on I/O; it just submits work and registers callbacks.

**Event loop phases (simplified):**
```
┌─────────────────────────────────────────────┐
│              Event Loop Tick                │
│                                             │
│  1. timers       (setTimeout, setInterval)  │
│  2. pending I/O  (completed I/O callbacks)  │
│  3. idle/prepare (internal)                 │
│  4. poll         (wait for new I/O events)  │
│  5. check        (setImmediate callbacks)   │
│  6. close        (socket close events)      │
└─────────────────────────────────────────────┘
         ↑                          ↓
         └──────────── repeat ──────┘
```

When you call `clickRepo.insert(...)`, Node submits a TCP write to the MongoDB socket (handled by libuv) and immediately returns a Promise. The JS thread moves on. When MongoDB acknowledges the write, libuv puts the callback in the "pending I/O" queue. On the next event loop tick, the callback runs — which either resolves or rejects the Promise.

**This is why fire-and-forget works in Node but is dangerous in Go/Java threads:** In multi-threaded runtimes, "background work" means spawning a thread. Unbounded thread creation can exhaust resources. In Node, the event loop is the scheduler — there's no thread cost for adding another pending I/O callback.

---

### When to use fire-and-forget

Use it when **all three** conditions hold:

1. **The result is not needed by the caller.** Nothing you return to the user depends on this operation completing.
2. **Failure is acceptable (best-effort semantics).** Losing the operation has no correctness impact — just observability impact.
3. **The work is bounded.** You're not queuing infinite tasks that accumulate unboundedly if a dependency is slow.

**Good use cases:**

| Use case | Why F&F fits |
|----------|--------------|
| Click/view analytics | User doesn't need to wait; losing a click is acceptable |
| Audit log writes | Observability, not correctness; don't block user flow |
| Cache warming after a DB read | If it fails, next request just misses cache |
| Sending a welcome email | Email delivery is async anyway; failure → retry queue |
| Updating a "last seen" timestamp | Slightly stale data is fine |
| Prometheus counter increment | In-memory; can't fail in the traditional sense |
| Webhook delivery to third-party | Send and move on; delivery is their problem |

---

### When NOT to use fire-and-forget

**Never fire-and-forget when:**

1. **Correctness requires the write.** If the data must exist before you respond, you must await.
   ```typescript
   // WRONG: user gets shortUrl but DB write might not have happened yet
   urlRepo.insert(urlDoc); // not awaited!
   return res.json({ shortUrl });
   ```

2. **The caller needs the result.** Any `await result =` pattern means you need the return value.

3. **You need transactional consistency.** If A and B must both succeed or both fail, fire-and-forget breaks atomicity.

4. **Resource exhaustion is possible.** If the background work is slow and requests pile up, you can queue thousands of pending inserts in memory with no backpressure.
   ```typescript
   // HIGH TRAFFIC: 10k req/s, MongoDB slow → 10k pending inserts in event loop queue
   // Memory grows; no backpressure; eventual OOM
   heavyWork(); // not awaited, repeated at high rate
   ```

5. **Graceful shutdown requires draining.** If your process receives SIGTERM while 500 fire-and-forget inserts are in-flight, they all die silently.

---

### The hidden dangers

#### 1. Silent data loss on shutdown

```typescript
// SIGTERM arrives while 200 inserts are pending
process.on('SIGTERM', async () => {
  await server.close();
  await mongo.close();
  process.exit(0);  // ← the 200 in-flight inserts are GONE
});
```

Solutions:
- **Drain counter:** increment a counter before each F&F, decrement in `.finally()`. On shutdown, wait until counter reaches 0.
- **Message queue:** instead of writing directly, push to a queue (Redis, RabbitMQ, SQS). The queue persists across crashes. A worker reads and inserts.
- **Accept the loss:** explicitly document and monitor with `clickWriteErrors` — this project's approach. Correct for analytics; wrong for financial data.

#### 2. Uncontrolled concurrency / backpressure

Every not-awaited Promise is a task in the event loop with no limit. If your downstream is slow:
```
10,000 requests/s → 10,000 concurrent DB inserts
MongoDB handles 1,000 inserts/s → queue grows at 9,000/s
After 1 minute: 540,000 pending inserts in memory
→ OOM crash
```

Solution: use a semaphore or a bounded queue.
```typescript
import pLimit from 'p-limit';
const limit = pLimit(100); // max 100 concurrent

// instead of bare fire-and-forget:
limit(() => clickRepo.insert(data)).catch(handleErr);
```

#### 3. Context/scope leakage

Variables captured by the F&F closure stay in memory until the Promise settles. If the closure captures a large request object or database connection, that memory can't be GC'd.

```typescript
// req is captured in closure — stays alive until insert settles
const { ip, userAgent, referrer } = req; // extract primitives instead
clickRepo.insert({ ip, userAgent, referrer }).catch(logger.warn);
```

Always extract primitive values from request objects before firing; don't close over the whole `req`.

---

### Alternatives to raw fire-and-forget

| Alternative | When to use | Tradeoff |
|-------------|-------------|----------|
| **Await** | Result needed or failure is unacceptable | Adds latency to the response |
| **Message queue** (Redis Streams, SQS, RabbitMQ) | Durability required, high volume, need retry | Adds infrastructure; survives crashes |
| **`Promise.allSettled()`** | Want to run multiple tasks and respond after all settle, regardless of failures | Still awaited — adds full duration to response |
| **`setImmediate()`** | Defer sync CPU work to after current I/O, not for async work | Doesn't help with async I/O |
| **Worker threads** | Heavy CPU-bound work | Separate thread, memory copy overhead |
| **p-limit / bottleneck** | F&F but with concurrency cap (backpressure) | Small overhead; prevents OOM under load |

**Message queue pattern** (the production-grade version of this project's analytics):
```typescript
// Instead of:
clickRepo.insert(data).catch(logger.warn);

// Durable version:
await redisStream.xadd('clicks', '*', data);
// A separate worker process reads the stream and inserts to MongoDB
// Survives crashes, has retry logic, naturally backpressures
```

---

### Decision tree: should I await this?

```
Is the result needed to construct the response?
  YES → await

Does failure mean incorrect data (not just missing data)?
  YES → await

Is this financial, security, or compliance data?
  YES → await (and consider a queue for durability)

Could this operation be slow and called at high rate?
  YES → await or use bounded queue (p-limit / message queue)

Does graceful shutdown need this to complete?
  YES → await or use a drain counter

Everything above is NO?
  → fire-and-forget with .catch() + metrics
```

---

### This project vs. production analytics

| Aspect | This project | Production |
|--------|-------------|------------|
| Mechanism | F&F MongoDB insert | Emit to message queue (Kafka/SQS) |
| Durability | Loses in-flight on crash | Queue persists; replayed on restart |
| Backpressure | None — unbounded | Queue depth + consumer rate |
| Retry | None — single attempt | Consumer retries with backoff |
| Observability | `clickWriteErrors` counter | Dead-letter queue + consumer lag metrics |
| Correctness | Acceptable: analytics | Required: billing, audit logs |

The F&F pattern here is correct for a learning project / small-scale deployment. At scale, you'd replace the direct MongoDB insert with a `redis.xadd()` to a Redis Stream (or Kafka topic), and run a separate click consumer service.

---

**References:**
- [MDN: Using Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)
- [MDN: async/await](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Promises#async_and_await)
- [Node.js Event Loop — official guide](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick)
- [Node.js: unhandledRejection](https://nodejs.org/api/process.html#event-unhandledrejection)
- [ESLint: no-floating-promises](https://typescript-eslint.io/rules/no-floating-promises/)
- [p-limit — concurrency control](https://github.com/sindresorhus/p-limit)
- [Redis Streams as message queue](https://redis.io/docs/data-types/streams/)
- [Libuv — the async I/O library under Node](https://libuv.org/)

---

## 8. Alerting & the SRE Signal Pipeline

### Monitoring is not incident response

A dashboard full of metrics tells you what's wrong **once you're already looking**. The
metric that matters at 3am is the one that *wakes someone up*. Without alerts, your
**MTTD** (Mean Time To Detect) is "however long until a human happens to glance at Grafana"
— which in practice means "until a customer complains."

```
metrics only:     incident happens ──────────?─────────► someone notices ──► fix
                                    (minutes to hours of silent failure)

with alerting:    incident happens ──► rule fires ──► page ──► fix
                                    (seconds to detect)
```

This project originally shipped dashboards but no alert rules — "production-visible,
not production-ready." Sections below describe the pipeline added to close that gap.

### The Prometheus alerting pipeline

```
┌────────────┐  evaluates   ┌──────────────────┐  fires    ┌──────────────┐  routes  ┌──────────┐
│ Prometheus │─────────────►│ alerting rules   │──────────►│ Alertmanager │─────────►│ receiver │
│  (scrapes) │  every 15s   │ (expr + for:)    │  HTTP push│ (dedup/group)│          │(Slack/PD)│
└────────────┘              └──────────────────┘           └──────────────┘          └──────────┘
```

- **Prometheus** evaluates each rule's PromQL `expr` every `evaluation_interval` (15s here).
- **Alertmanager** is a *separate* process. It deduplicates (10 app replicas firing the same
  alert = one notification), groups, silences, and routes to receivers. Prometheus only
  decides *whether* an alert fires; Alertmanager decides *who hears about it and how often*.
- A **receiver** is the delivery target (Slack, PagerDuty, Opsgenie, email, webhook).

### `for:` — pending vs firing (debouncing)

A rule with `for: 5m` does not fire the instant its `expr` is true. It goes **pending**
first, and only transitions to **firing** if the condition stays true for the whole 5
minutes. This filters out transient blips — a single slow scrape or a 10-second CPU spike
shouldn't page anyone.

```
expr true? :  no  no  YES YES YES YES YES no ...
state      :  inactive──► pending(starts timer)──► (cleared before 5m → back to inactive)

expr true? :  YES YES YES YES YES YES (≥ 5m) ...
state      :  pending ─────────────────────────► FIRING ──► Alertmanager
```

### Symptom-based vs cause-based alerting

| Style | Alerts on… | Example | Pro / Con |
|-------|-----------|---------|-----------|
| **Symptom** | User-visible pain | `HighErrorRate`, `HighLatencyP99` | Always actionable; few false pages. Preferred. |
| **Cause** | An internal condition | `RedisDown`, `TargetDown` | Faster root-cause, but can page for things users never feel. |

Google SRE guidance: **page on symptoms, diagnose with causes.** Too many cause-based pages
cause alert fatigue. This project keeps cause-based alerts (`RedisDown`) at lower severity
where the cache degrades gracefully (DB still serves), and treats `HighErrorRate` /
`HighLatencyP99` as the real signals.

### SLO burn-rate alerting (the next level)

A mature setup doesn't alert on "error rate > 5% right now." It alerts on **how fast you're
burning your error budget**. If your SLO is 99.9% success (0.1% budget/month), a burn-rate
alert fires when you're consuming that budget fast enough to exhaust it before the window
ends — fast burn pages immediately, slow burn opens a ticket. This avoids both flapping and
slow leaks going unnoticed. (Not implemented here — `HighErrorRate` is a simple threshold —
but it's the natural evolution.)

### Mechanics — what Prometheus actually does each cycle

Prometheus is a single process running a loop:

```
every scrape_interval (15s):   GET http://app1:3000/metrics  → append samples to local TSDB
every evaluation_interval(15s):for each rule: run the PromQL expr against the TSDB
                                 ├─ expr returns rows? → those label-sets are "active"
                                 ├─ active < for: duration  → state = PENDING (not sent anywhere)
                                 └─ active ≥ for: duration   → state = FIRING  → push to Alertmanager
```

Key points:
- An alert is **per result row**, not per rule. If the expr returns 3 series, you get 3
  alert instances with different labels.
- Prometheus pushes alerts to Alertmanager's `/api/v2/alerts` **and keeps re-sending every
  evaluation** while firing (so a restarted Alertmanager re-learns state). When the expr
  stops returning the row, Prometheus sends a `resolved`.
- **Only FIRING is sent.** `pending` lives entirely inside Prometheus — that's why a pending
  alert never shows in the Alertmanager UI.

### Mechanics — what Alertmanager does with a firing alert

Alertmanager is a *separate* process whose whole job is turning a stream of firing alerts
into the right number of useful notifications:

```
firing alert in ─► [ route tree ] ─► [ group_by ] ─► [ wait/dedup/inhibit/silence ] ─► receiver
```

- **Routing** — a tree matches alert labels to a receiver (e.g. `severity=critical` →
  PagerDuty, everything else → Slack).
- **Grouping** (`group_by`) — 10 app replicas all firing `RedisDown` collapse into **one**
  notification, not ten. `group_wait` (10s) holds the first notification briefly so related
  alerts batch together; `group_interval` controls follow-ups; `repeat_interval` (1h here)
  is how often it re-notifies if still firing.
- **Dedup** — identical alerts from multiple Prometheis (HA pairs) become one.
- **Silences** — a human mutes matching alerts for a window (deploys, maintenance).
- **Inhibition** — a higher-level alert suppresses noise (e.g. `TargetDown` inhibits
  `HighLatencyP99` for the same target — no point paging on latency when it's down).

In this project the receiver is `null` (no real paging), so firing alerts show up in the
Alertmanager **UI** but go nowhere else — enough to prove the pipeline.

### Pitfall — `for:` must be shorter than the data's life in the rate window

This trips everyone (and is why a quick `npm run gen:errors` burst won't fire
`HighErrorRate`). The rule is:

```
HighErrorRate:  rate(http_requests_total{5xx}[5m]) / rate(...[5m]) > 0.05   for: 5m
```

A 5-second error burst lands in the `[5m]` window, so the ratio spikes and the alert goes
**pending**. But the burst's samples only *stay* in a `[5m]` window for 5 minutes — they age
out exactly as the `for: 5m` timer completes, the expr flips false, and the alert resolves
**without ever firing**.

```
t=0    50 errors fired (5s burst), Mongo unpaused
t=0    expr true  → PENDING (for: timer starts)
t=5m   for: timer would complete... but the 50 errors just left the rate[5m] window
       expr → false → back to inactive. NEVER FIRES.
```

**Rule of thumb:** to fire an alert with `for: D` and `rate[W]`, the condition must hold for
**at least D**, which means the underlying events must keep happening for ≳ D (not just W).
To actually trigger `HighErrorRate`: sustain errors > 5 min
(`node scripts/gen-errors.js 2000 5` ≈ 6.7 min of paused Mongo), or lower `for:`. By
contrast `TargetDown`/`RedisDown` fire reliably in `test-alerts.sh` because their condition
(`up==0`, `redis_circuit_open==1`) is *level-triggered* — it stays true the whole time the
dependency is down, not a decaying rate.

### How this project does it

`observability/prometheus.rules.yml` defines six alerts; `observability/prometheus.yml`
wires `rule_files` + an `alertmanagers` target; `observability/alertmanager.yml` defines a
no-op `null` receiver (the challenge has no real paging integration — it demonstrates the
full pipeline). The `alertmanager` service runs in `docker-compose.yml` (loopback `:9093`).

| Alert | Type | Expr (essence) | `for:` |
|-------|------|----------------|--------|
| `TargetDown` | cause | `up{job="url-shortener"} == 0` | 1m |
| `HighErrorRate` | symptom | 5xx ratio > 5% | 5m |
| `RedisDown` | cause | `redis_circuit_open == 1` | 2m |
| `RedirectCacheHitRatioLow` | cause | url hit ratio < 80% | 10m |
| `ClickWriteErrors` | cause | `rate(click_write_errors_total) > 0` | 5m |
| `HighLatencyP99` | symptom | p99 > 0.5s | 5m |

`scripts/test-alerts.sh` proves the pipeline end-to-end: it stops `app1` / `redis`, polls
Prometheus `/api/v1/alerts` until the alert is `firing`, confirms it propagated to
Alertmanager `/api/v2/alerts`, then restores and confirms it clears.

**References:**
- [Prometheus: Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Prometheus: Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Google SRE Workbook: Alerting on SLOs (burn rate)](https://sre.google/workbook/alerting-on-slos/)
- [My Philosophy on Alerting — Rob Ewaschuk](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/)

---

## 9. RED & USE — Two Methods for Picking Metrics

You can't graph everything. Two complementary mental models tell you *which* signals matter.

### RED — for request-driven services (the app)

| Letter | Metric | This project |
|--------|--------|--------------|
| **R**ate | Requests per second | `rate(http_requests_total[1m])` (read vs write panels) |
| **E**rrors | Failed requests per second | `rate(http_requests_total{status=~"5.."}[1m])` |
| **D**uration | Latency distribution | `histogram_quantile(…, http_duration_seconds_bucket)` |

RED answers: *"Are my users getting fast, correct responses?"* It's symptom-oriented — the
same three signals that the best alerts (§8) fire on.

### USE — for resources (CPU, memory, disk, pools, the event loop)

| Letter | Meaning | This project |
|--------|---------|--------------|
| **U**tilization | % of time the resource is busy | CPU via `collectDefaultMetrics`, container `cpus` limit |
| **S**aturation | Queued/waiting work the resource can't service yet | **event-loop lag** (`nodejs_eventloop_lag_*`), RSS vs `mem_limit` |
| **E**rrors | Error events from the resource | `redis_errors_total`, `click_write_errors_total` |

USE answers: *"Is any resource the bottleneck?"*

### Why event-loop lag is *the* Node saturation signal

Node is single-threaded. If a synchronous handler hogs the CPU, the event loop can't service
pending I/O callbacks — they queue up. **Event-loop lag** measures exactly that delay: the
gap between when a timer *should* fire and when it *actually* fires. Rising lag means the
process is saturated even if CPU% looks moderate. It's the classic Node 3am page, which is
why a panel for `nodejs_eventloop_lag_p99_seconds` was added (it ships free with
`collectDefaultMetrics`, it just wasn't graphed).

### How this project does it

`src/observability/metrics.ts` registers RED metrics explicitly; `collectDefaultMetrics`
provides USE signals (event-loop lag, RSS, GC, CPU). The Grafana dashboard
(`observability/grafana/dashboards/url-shortener.json`) now has panels for event-loop lag
p99 and process RSS alongside the existing RED panels.

**References:**
- [The RED Method — Tom Wilkie (Grafana/Weave)](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
- [The USE Method — Brendan Gregg](https://www.brendangregg.com/usemethod.html)
- [Google SRE Book: The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/#xref_monitoring_golden-signals)
- [Node.js perf_hooks: monitorEventLoopDelay](https://nodejs.org/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions)

---

## 10. Liveness vs Readiness Probes

These two health checks answer **different questions**, and conflating them causes outages.

| Probe | Question | If it fails, the orchestrator… | Checks dependencies? |
|-------|----------|-------------------------------|----------------------|
| **Liveness** | "Is this process alive / not deadlocked?" | **Restarts the container** | **No** |
| **Readiness** | "Can this instance serve traffic right now?" | **Removes it from the load balancer** (no restart) | **Yes** |

### The restart-storm trap

Suppose you use **one** `/health` endpoint that pings MongoDB, and wire it to the container's
liveness check. Mongo has a 30-second blip:

```
Mongo blips
  → /health returns 503 on every app instance
  → liveness probe fails everywhere
  → orchestrator KILLS and restarts every app container simultaneously
  → cold caches, reconnection storms, dropped in-flight work
  → the blip became a full outage
```

The app was **fine** — only its dependency hiccuped. Restarting it fixed nothing and made
everything worse. Liveness must depend on **nothing external**. Readiness is where dependency
checks belong: a 503 there just stops routing new traffic to that instance until Mongo
recovers — no restart, no storm.

### How this project does it

`src/server.ts`:
- `GET /livez` → returns `{status:'ok'}` with **no dependency checks**. The Docker
  healthcheck in `docker-compose.yml` points here, so a Mongo blip never restarts a healthy app.
- `GET /health` → pings Mongo, returns 503 if unreachable. This is the **readiness** probe —
  intended for the load balancer (NGINX) to decide routing.

```
docker-compose healthcheck:  curl -f http://localhost:3000/livez   ← restart decision (no deps)
load balancer routing:       GET /health                            ← traffic decision (deps)
```

**Kubernetes mapping:** `livenessProbe` → `/livez`, `readinessProbe` → `/health`. (A third,
`startupProbe`, guards slow-starting apps so liveness doesn't kill them during boot — the
compose `start_period: 15s` is the equivalent here.)

**References:**
- [Kubernetes: Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Google SRE: Health checking](https://sre.google/sre-book/load-balancing-datacenter/)
- [Liveness probes are dangerous — Colin Breck](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-how-to-avoid-shooting-yourself-in-the-foot/)

---

## 11. Metric Label Cardinality

**Cardinality** = the number of distinct time series a metric produces. Prometheus stores
**one time series per unique label-value combination**, in memory. This is the single
easiest way to take Prometheus down.

```
http_requests_total{method, route, status}

methods: ~4   ×   routes: ~5   ×   statuses: ~10   =   ~200 series   ✅ fine
```

Now imagine `route` held the **raw URL path** instead of a route *pattern*:

```
http_requests_total{route="/aB3xK"}      ← every short code is a new series
http_requests_total{route="/9zQ1p"}
http_requests_total{route="/scan-attempt-47281"}   ← every scanner probe too
...millions of series → Prometheus OOMs
```

This is a **cardinality bomb**. A bot scanning random paths could mint unbounded series and
crash your monitoring — turning an attack on your app into an attack on your observability.

### The rules

1. **Labels must be bounded.** Never put raw URLs, user IDs, emails, request IDs, timestamps,
   or full error messages in a label.
2. **Use patterns, not values.** `route="/:shortUrl"` (one series) not `route="/aB3xK"` (∞).
3. **Bucket high-cardinality dimensions.** `status_class="2xx"` (4 values) instead of, or
   alongside, raw `status` only where the raw value is genuinely needed.
4. **High-cardinality data belongs in logs/traces,** not metrics. (See §12 — that's what
   `X-Request-Id` in Loki is for.)

### How this project does it

`src/middleware/metrics.ts`:
- `routeLabel()` returns `req.route?.path` (always a **pattern** like `/:shortUrl`) and
  collapses the unmatched catch-all to a single fixed `'unmatched'` label — so scanner
  traffic hitting random paths can never expand cardinality.
- `statusClass()` maps the numeric status to `2xx/3xx/4xx/5xx` for the **duration histogram**
  label, keeping latency splittable-by-outcome without an unbounded status dimension. The raw
  `status` stays on the *counter* only (where ~10 codes is acceptable).

`src/middleware/metrics.test.ts` asserts the guard: it fires matched + scanner traffic and
verifies raw path segments **never** appear as label values.

**References:**
- [Prometheus: Naming and labels best practices](https://prometheus.io/docs/practices/naming/)
- [Prometheus: Cardinality is key](https://www.robustperception.io/cardinality-is-key/)
- [Grafana: Avoiding high cardinality](https://grafana.com/blog/2022/02/15/what-are-cardinality-spikes-and-why-do-they-matter/)

---

## 12. Correlating Logs, Metrics & Traces

The "three pillars of observability" answer different questions:

| Pillar | Answers | Cardinality | This project |
|--------|---------|-------------|--------------|
| **Metrics** | "Is something wrong, and how much?" | Low (bounded labels) | Prometheus |
| **Logs** | "What exactly happened on this request?" | High (free-form) | pino → Loki |
| **Traces** | "Where did the time go across services?" | High (per-span) | (not implemented — single service) |

A metric tells you p99 latency spiked. It can't tell you *which* requests were slow. A log
line has the detail but you need a way to **pivot** from the spike to the lines. That bridge
is a shared **request ID**.

```
client reports "the redirect was slow at 14:32"
        │
        ▼
response carried  X-Request-Id: 7f3a…           ← the bridge
        │
        ▼
Loki:  {app="app1"} |= "7f3a…"                  ← exact log lines for that request
```

For a **single service**, an end-to-end request ID is the cheap 80% of distributed tracing.
Full tracing (OpenTelemetry, propagating a trace context across service hops) matters once
you fan out to multiple services — it's the natural next step, not needed here.

### How this project does it

- `src/middleware/logger.ts` — `pino-http`'s `genReqId` assigns a UUID `req.id` to every
  request; every log line for that request carries it.
- `src/server.ts` — a middleware sets `res.setHeader('X-Request-Id', req.id)` so the id is
  visible to clients and in network traces. It runs **after** the logger (which creates the
  id) and before route handlers.
- The id flows into Loki via Promtail, so a client-reported request is greppable end-to-end.

**References:**
- [Three pillars of observability — Honeycomb](https://www.honeycomb.io/blog/observability-101-terminology-and-concepts)
- [OpenTelemetry: Context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [Grafana Loki: LogQL](https://grafana.com/docs/loki/latest/query/)
- [pino-http: request id](https://github.com/pinojs/pino-http#pinohttpopts-stream)

---

## 13. Container Hardening — Blast Radius & Isolation

Most of these are one-line changes that turn a small incident into a contained one instead
of a host-wide one.

### Non-root containers

By default a container's process runs as **root inside the container**. If an attacker gets
RCE (or a dependency is compromised), root-in-container is a much larger blast radius — it
eases container-escape exploits and lets the process tamper with mounted files. Running as an
unprivileged user is defense-in-depth.

```dockerfile
RUN chown -R node:node /app
USER node          # node:alpine ships this user; the app never needs root
```

### Resource limits — the noisy-neighbor / OOM problem

Containers on one host share the kernel and physical RAM. With **no limits**, a memory leak
(or Redis filling to its `--maxmemory`) can consume all host RAM and the OOM killer starts
killing *other* containers — your DB dies because your app leaked.

```yaml
mem_limit: 512m      # app capped — a leak kills only the app, host stays up
cpus: 1.0
# redis mem_limit set ABOVE its --maxmemory 2gb so it evicts (LFU, §5) before OOM-kill
```

Limits convert "unbounded host failure" into "one container restarts."

### Timing-safe secret comparison

A naive `token !== expected` returns as soon as the first differing byte is found. The
**time taken leaks how many leading bytes matched** — an attacker can recover a secret byte
by byte (a timing side-channel). Use a constant-time compare:

```ts
function tokensMatch(provided: string, expected: string): boolean {
  const a = Buffer.from(provided), b = Buffer.from(expected)
  return a.length === b.length && timingSafeEqual(a, b)  // always compares all bytes
}
```

(Length is compared first because `timingSafeEqual` *requires* equal-length buffers; the
length itself is low-value to leak.)

### Loopback-only port binding

`ports: ["27017:27017"]` binds to `0.0.0.0` — the datastore is reachable from **any** network
interface, including public ones, often with no auth. Binding to `127.0.0.1` keeps the port
available for local debugging (`mongosh`, `redis-cli`) while making it unreachable from the
network. Inter-container traffic is unaffected — it uses Docker's internal DNS, not the
published host port.

```yaml
ports:
  - "127.0.0.1:27017:27017"   # localhost only, not the world
```

### How this project does it

- `Dockerfile`: `chown` + `USER node` in the runtime stage (non-root).
- `docker-compose.yml`: `mem_limit`/`cpus` on every service; Mongo, Redis, Prometheus,
  Alertmanager and the exporters bound to `127.0.0.1` (only NGINX `:80` and Grafana `:3001`
  are world-facing); Grafana admin password no longer defaults to `admin`.
- `src/server.ts`: `tokensMatch()` uses `crypto.timingSafeEqual` for the `/metrics` token.

**References:**
- [Docker: Run containers as a non-root user](https://docs.docker.com/build/building/best-practices/#user)
- [Docker Compose: resource constraints](https://docs.docker.com/compose/compose-file/compose-file-v3/#resources)
- [Node.js crypto.timingSafeEqual](https://nodejs.org/api/crypto.html#cryptotimingsafeequala-b)
- [OWASP: Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

---

## 14. Metrics Data Flow — Pull Model, Source vs View

A common confusion: *"if Grafana already shows the data, isn't the app's `/metrics`
endpoint a pointless bypass?"* No — it's the **source**. The dependency runs the other way.

```
┌──────────────┐  scrape every 15s   ┌──────────────┐   PromQL query   ┌──────────┐
│ app /metrics │────────────────────►│  Prometheus  │─────────────────►│ Grafana  │
│   SOURCE     │   (HTTP GET, pull)  │   STORAGE    │  (on dashboard   │   VIEW    │
│ live snapshot│                     │ 30d history  │     load)        │  (draws) │
└──────────────┘                     └──────────────┘                  └──────────┘
```

| Layer | Role | Has history? | Generates data? |
|-------|------|--------------|-----------------|
| app `/metrics` | **produces** the numbers (in-process counters/gauges) | No — instantaneous | Yes (the origin) |
| Prometheus | **scrapes & stores** | Yes (TSDB) | No — just records |
| Grafana | **queries & draws** | No (reads Prometheus) | **No** |

Grafana renders *nothing on its own*. Kill `/metrics` → Prometheus scrapes empty →
Grafana goes blank. "Grafana shows the same data" is the whole point: it's *that* data,
stored and graphed. Hitting `/metrics` directly is only for debugging ("is the app even
exposing counter X?") without the scrape→store→render lag.

### Pull vs push

Prometheus **pulls** (scrapes an HTTP endpoint) rather than the app **pushing**. Benefits:
the app stays dumb (just exposes current values, no network egress to a metrics backend);
scrape failures are themselves a signal (`up == 0` → the `TargetDown` alert); and any tool
can read `/metrics` independently. The trade-off — short-lived jobs that die between scrapes
need a Pushgateway — doesn't apply to a long-running web server.

### Two families on `/metrics`

| Family | Examples | Method | Why |
|--------|----------|--------|-----|
| **App metrics** (explicit) | `http_requests_total`, `cache_hits_total`, `redis_circuit_open` | defined in `src/observability/metrics.ts` | RED + domain signals |
| **Default metrics** (auto) | `process_resident_memory_bytes`, `nodejs_eventloop_lag_p99_seconds`, `nodejs_gc_duration_seconds`, `process_open_fds` | `collectDefaultMetrics()` | USE/saturation — *why* the process is slow |

The `process_*` / `nodejs_*` block is **not noise** — it's the resource (USE, §9) view that
app metrics can't provide: event-loop lag (blocked loop), RSS (leak), GC pauses (latency),
fd count (connection leak). The dashboard's event-loop-lag and RSS panels read exactly these.

### How this project does it

`src/observability/metrics.ts` builds one `prom-client` registry (`collectDefaultMetrics`
for the USE family + explicit RED/domain metrics). `src/server.ts` serves it at `/metrics`
behind a bearer token. `observability/prometheus.yml` scrapes `app1:3000` every 15s;
`observability/grafana/provisioning/datasources` points Grafana at Prometheus.

**References:**
- [Prometheus: Overview & data model (pull)](https://prometheus.io/docs/introduction/overview/)
- [Prometheus: Why pull over push](https://prometheus.io/docs/introduction/faq/#why-do-you-pull-rather-than-push)
- [prom-client: default metrics](https://github.com/siimon/prom-client#default-metrics)
- [Grafana: Prometheus data source](https://grafana.com/docs/grafana/latest/datasources/prometheus/)

---

## 15. WHATWG URL Parsing & Input Normalization

### What is the WHATWG URL Standard

The **WHATWG URL Standard** (`https://url.spec.whatwg.org/`) is the living spec that
browsers and Node.js follow when parsing URLs. It replaced the older RFC 3986 approach
for the web platform and is what `new URL(string)` uses in JavaScript.

Key design goal: **be permissive in what you accept, but produce a canonical output**.
This means the parser silently fixes many inputs rather than rejecting them:

| Input | `new URL(input).href` | What happened |
|-------|----------------------|----------------|
| `"https://EXAMPLE.COM/Path"` | `"https://example.com/Path"` | host lowercased |
| `"https://example.com/a%20b"` | `"https://example.com/a%20b"` | percent-encoding preserved |
| `"https://example.com/a b"` | `"https://example.com/a%20b"` | space in path encoded |
| `"  https://example.com  "` | `"https://example.com/"` | **leading/trailing whitespace stripped** |
| `"https://example.com/./a/../b"` | `"https://example.com/b"` | path normalized |

The last two rows are the critical ones. The parser strips surrounding whitespace and
resolves dot-segments before it even starts interpreting the URL components.

### Zod `.url()` validates but does not normalize

Zod's `.url()` validator calls `new URL(input)` internally — but only to **check** if
parsing succeeds. It returns the **original string** unchanged:

```ts
z.string().url().parse("  https://example.com  ")
// returns: "  https://example.com  "   ← original, with spaces
```

This is correct behavior for a validator: Zod's job is to say yes/no, not rewrite your
data — unless you tell it to with a transform.

### The dedup bug this causes

This project deduplicates by hashing `longUrl` with SHA-256:

```
"https://example.com" → SHA-256 → "abc123..." → stored as longUrlHash index
```

If the raw (unstripped) string is hashed:

```
"https://example.com"   → SHA-256 → "abc123..." → shortCode: "xK3p"
"https://example.com "  → SHA-256 → "def456..." → shortCode: "zQ9r"   ← DIFFERENT
```

Two distinct short codes now redirect to the **same destination** — a dedup miss. The
WHATWG parser treats the two URLs as identical (whitespace is stripped before parsing),
but the SHA-256 hash sees two different byte sequences.

### The fix: `.trim()` before `.url()`

```ts
export const longUrlSchema = z
  .string()
  .trim()   // ← normalize BEFORE validation
  .url()
  .max(2048)
  ...
```

`.trim()` is a Zod transform — it rewrites the value, so `.url()` (and everything after)
sees the cleaned string. Now both `"https://example.com"` and `"https://example.com "`
hash to the same value and hit the dedup index.

### The general rule: normalize inputs at the boundary

```
           BEFORE normalization: "https://x.com/hello "
                                           │
        ┌──────────────────────────────────▼───────┐
        │         Validation boundary              │
        │  1. trim()  → "https://x.com/hello"     │  ← canonical form
        │  2. url()   → passes                    │
        │  3. max()   → passes                    │
        └──────────────────────────────────────────┘
                                           │
                  stored / hashed: "https://x.com/hello"  ✅
```

The WHATWG parser's silent normalization is useful in a browser (lenient UX), but in a
server storing a hash of the input it creates a mismatch between "what the parser sees"
and "what you store". Always normalize to the canonical form **before** hashing or storing.

### When to go further

`.trim()` handles whitespace. For complete URL canonicalization (fragment removal, scheme
lowercasing, trailing-slash normalization, query-string sort order), you could do:

```ts
.transform(u => new URL(u).href)   // full WHATWG normalization
```

This project stops at `.trim()` — it's enough for the dedup case and avoids rewriting
URLs in unexpected ways (e.g., normalizing `%2F` in paths).

### How this project does it

`src/utils/validators.ts` — `.trim()` added before `.url()` in `longUrlSchema`.
`src/utils/validators.test.ts` — test asserts that a URL with surrounding whitespace
produces the same `.data` value as the clean URL, confirming dedup consistency.

**References:**
- [WHATWG URL Standard](https://url.spec.whatwg.org/)
- [MDN: URL() constructor](https://developer.mozilla.org/en-US/docs/Web/API/URL/URL)
- [Zod: string validations & transforms](https://zod.dev/?id=strings)
- [Node.js: URL module (WHATWG implementation)](https://nodejs.org/api/url.html#the-whatwg-url-api)

---

## 16. Prometheus Histogram Design — Buckets, Labels, and Tail Coverage

### Why histograms for latency (not gauges, not counters)

Three metric types can record a duration:

| Type | What it stores | Can compute percentiles? | Memory per metric |
|------|---------------|--------------------------|-------------------|
| **Gauge** | one current value | No | 1 series |
| **Counter** | cumulative total | No | 1 series |
| **Histogram** | count of observations per bucket | Yes (approximate) | 1 series per bucket |
| **Summary** | pre-computed quantiles in-process | Yes (exact, fixed window) | 1 series per quantile |

Histograms win for latency because:
- Percentiles computed **server-side** in PromQL — you can compute them across instances
  and arbitrary time windows after the fact.
- Summaries compute quantiles in the **app process** — can't aggregate across replicas
  and the quantile window is fixed at instrument time.
- The trade-off: histogram percentiles are **approximate** (interpolated within a bucket).
  Accuracy improves with finer buckets around the values you care about.

### How buckets work

A histogram creates one counter per bucket boundary. Each counter tracks how many
observations fell **at or below** that boundary (cumulative):

```
observe(0.003s):
  http_duration_seconds_bucket{le="0.005"} += 1
  http_duration_seconds_bucket{le="0.01"}  += 1
  http_duration_seconds_bucket{le="0.025"} += 1
  ... all larger buckets also increment (cumulative)

observe(0.150s):
  http_duration_seconds_bucket{le="0.25"} += 1
  http_duration_seconds_bucket{le="0.5"}  += 1
  ... larger buckets only
```

`histogram_quantile(0.99, rate(http_duration_seconds_bucket[5m]))` interpolates the p99
value from these cumulative counts. If 99% of observations fall in the `(0.1, 0.25]`
bucket, Prometheus assumes a uniform distribution within that range and interpolates.
**Accuracy degrades if the bucket is too wide.**

### Bucket design: cover your SLO, extend the tail

The default `prom-client` buckets stop at **10 seconds**. For a URL shortener with a
sub-10ms p99 target, the interesting range is `[0.005, 0.5]`. But you still need tail
buckets — not for normal traffic, but because:

1. **Alerts on p99** need a finite bucket to interpolate into. Without a `10` bucket,
   requests that take 7 seconds would all pile into `+Inf` and your HighLatencyP99
   alert's PromQL expression can't distinguish "slightly slow" from "completely stuck".
2. **Incident investigation**: when something is wrong, you want to know *how* wrong —
   "p99 is 3s" vs "p99 is 8s" leads to different diagnoses.

```
Default prom-client buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1]
                                                                           ↑
                                                         stops here — no tail

This project's buckets:      [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
                                                                            └───────────┘
                                                                             tail coverage
```

The rule: **your slowest SLO threshold should sit inside a bucket**, not past the end.

### The `status_class` label — outcome segmentation without cardinality explosion

Adding `status` (raw HTTP code) to a histogram label is tempting — you'd like separate
latency percentiles for successful vs failed requests. The problem:

```
Without status_class:
  http_duration_seconds_bucket{method, route, le}
  ~4 methods × ~5 routes × 11 buckets = 220 series ✅

With raw status:
  http_duration_seconds_bucket{method, route, status, le}
  ~4 × ~5 × ~10 statuses × 11 buckets = 2200 series — 10× more
  + scanners can probe unusual status paths → unbounded
```

`status_class` collapses all status codes to four values (`2xx`, `3xx`, `4xx`, `5xx`):

```
http_duration_seconds_bucket{method, route, status_class, le}
~4 × ~5 × 4 × 11 = 880 series — bounded and useful
```

Now you can write:

```promql
histogram_quantile(0.99,
  sum by (le) (
    rate(http_duration_seconds_bucket{status_class="2xx"}[5m])
  )
)
```

— p99 latency for **successful requests only**, filtering out error paths that are always
slow (e.g., database-down paths) from inflating the SLO metric.

### How `status_class` enables the HighErrorRate alert

The `HighErrorRate` alert uses `http_requests_total` (the counter, which keeps raw
`status`). The histogram's `status_class` is separate — it's there for the latency
**SLO panel**, not the error rate alert. The two metrics serve different purposes:

| Metric | Label used | Purpose |
|--------|-----------|---------|
| `http_requests_total` | raw `status` | error rate alert — `rate(...{status=~"5.."}[5m])` |
| `http_duration_seconds` | `status_class` | latency SLO — `histogram_quantile(0.99, ...)` |

### Grafana panel design choices

Three new panels added in this session:

| Panel | Metric | Why |
|-------|--------|-----|
| Redis circuit state | `redis_circuit_open` | binary gauge: 1=broken, 0=ok — instantly visible degradation |
| Event-loop lag p99 | `nodejs_eventloop_lag_p99_seconds` | saturation signal — blocked loop → slow response even with fast DB |
| Process RSS | `process_resident_memory_bytes` | leak detection — RSS growing over hours = memory leak |

The error-log Loki panel filter was changed from `{level="50"}` (error only) to
`{level=~"40|50"}` (warn + error). pino outputs numeric levels: `30`=info, `40`=warn,
`50`=error. Click-write failures log at `warn` (40) — they'd be invisible with the old filter.

### How this project does it

`src/observability/metrics.ts` — `httpDuration` Histogram defined with the extended
bucket array and `status_class` in `labelNames`.
`src/middleware/metrics.ts` — `statusClass(code)` helper maps `Math.floor(code/100)` to
`"2xx"` string. `routeLabel(req)` returns `req.route?.path ?? 'unmatched'`.
`observability/grafana/dashboards/url-shortener.json` — panels 11–13 for circuit state,
event-loop lag, and RSS; panel 10 filter updated for warn+error.

**References:**
- [Prometheus: Histogram vs Summary](https://prometheus.io/docs/practices/histograms/)
- [Prometheus: histogram_quantile](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile)
- [prom-client: Histogram](https://github.com/siimon/prom-client#histogram)
- [Google SRE Book: Latency SLOs](https://sre.google/sre-book/service-level-objectives/)
- [Robust Perception: Cardinality is key](https://www.robustperception.io/cardinality-is-key/)

---

## 17. Monolith vs Microservices vs Event-Driven Architecture

These are three separate axes, not three mutually exclusive boxes:

- **Monolith vs Microservices** = how you draw *deployment* boundaries (one process vs many).
- **Event-driven vs Request/Response (RPC/REST)** = how components *communicate* across whatever
  boundaries you drew.

You can have a request/response monolith (this project), an event-driven monolith (a single
process that talks to itself via an in-memory event bus), request/response microservices (services
calling each other's REST/gRPC APIs synchronously), or event-driven microservices (services
publishing to Kafka/SQS/RabbitMQ, decoupled). Conflating the two axes is the most common mistake
when this topic comes up in interviews.

### Monolith — when it's the right call

A monolith is **one deployable unit** containing all the business logic, even if internally
organized into modules/domains (like this project's `url/`, `click/`, `counter/` folders).

**Use a monolith when:**
- Team is small (roughly < 8-10 engineers) — microservices' main cost is coordination overhead,
  which only pays off once a single team can't agree on a single codebase/release cadence anymore.
- Domain boundaries are still unclear or the product is pre-product-market-fit. Splitting services
  along the wrong seams is far more expensive to undo than splitting a monolith's modules later —
  modules live in one repo, one transaction, one deploy; undoing a network boundary means rewriting
  contracts, migrating data, and coordinating two teams' release schedules.
- You need transactional consistency across entities (e.g., "decrement inventory AND create order"
  in one ACID transaction). Distributed transactions across services are hard (two-phase commit,
  sagas) and a monolith gets this for free via the DB transaction.
- Operational simplicity matters more than independent scaling — one thing to deploy, one thing to
  monitor, one log stream, no service mesh, no distributed tracing required to debug a request.
- This project is exactly this case: a single Express process. `url/`, `click/`, `counter/` are
  domains within one deploy unit. The README's `npm run build && node dist/main.js` is the entire
  deploy story. No inter-service network calls, no partial-failure-across-services class of bugs.

**Cost of staying monolith too long:** a single team's changes start colliding (merge conflicts,
shared test suite getting slow), unrelated features must scale together (can't scale the read-heavy
redirect path independently from the write-heavy shorten path without scaling the whole process),
and a bug in one module can crash the whole process for unrelated traffic.

### Microservices — when it's the right call

Microservices split the system into **independently deployable** services, each owning its own
data store, usually communicating over the network (REST/gRPC, sync) or a broker (async).

**Use microservices when:**
- Independent scaling is a real, measured need. Concrete example from this project's own
  shape: the redirect path (`GET /:shortCode`) gets ~10x the traffic of the shorten path
  (NGINX config here gives `/*` 100 req/s vs `/shorten` 10 req/s) — at large enough scale you'd
  split "redirect service" from "shorten service" so you can scale the cheap, hot read path
  without paying for extra capacity on the write path, and vice versa.
- Different parts of the system have genuinely different scaling/availability/tech requirements
  (e.g., a write-heavy service that's fine being eventually consistent vs. a billing service that
  must be ACID).
- Multiple teams need to ship independently without blocking on each other's release trains — the
  org chart is the real driver here (Conway's Law: system shape mirrors communication structure).
- You need fault isolation: one service crashing shouldn't take down unrelated functionality. In a
  monolith, an unhandled exception or memory leak in the analytics path can starve the redirect
  path on the same event loop; as separate services, the analytics service falling over doesn't
  touch redirects.
- Polyglot needs: e.g., a hot path written in Go for raw throughput while the rest stays in
  Node/TS — only possible across a process boundary.

**Cost:** network calls replace function calls (latency, partial failure, retries, idempotency),
each service needs its own CI/CD, monitoring, on-call story; debugging a single user request now
means correlating logs/traces across services (this is *why* distributed tracing — section 12 —
exists: it's the tool that makes microservices debuggable at all). Data consistency across services
requires sagas/outbox patterns instead of a DB transaction.

### Event-Driven Architecture — when it's the right call

Event-driven = components communicate by **publishing facts about what happened** (events) to a
broker, and other components **react** asynchronously, instead of one component directly calling
another and waiting for a response.

**Use event-driven when:**
- Producer shouldn't block on, or even know about, every consumer. Example from this very project:
  the redirect controller does fire-and-forget click inserts (section 7) — conceptually this is
  already "event-driven-lite": "a redirect happened" is a fact the click-tracking logic reacts to
  without the redirect response waiting on it. At larger scale this fire-and-forget `.catch()` call
  becomes a real event (`UrlClicked`) published to Kafka/SQS, and you could add new consumers
  (fraud detection, real-time analytics dashboards, billing) without ever touching the redirect
  controller again — that's the core win: **new consumers, zero changes to the producer.**
- You need to decouple services that have different uptime/throughput profiles. A queue/broker
  absorbs bursts — if the analytics DB is down or slow, events queue up instead of the redirect
  path failing or blocking (this project's `analyticsClient` with `w:1` writes + fire-and-forget is
  the in-process version of this same idea — section "Two MongoDB clients" in CLAUDE.md).
- Workflows naturally span multiple steps with retries/long-running state (order placed → payment
  charged → inventory reserved → shipped) — event-driven plus a saga/state-machine fits better than
  a long synchronous call chain that has to stay open across all of it.
- Audit/replay matters: an event log is a durable history of "everything that happened," which you
  can replay to rebuild state, debug an incident, or feed a new service that didn't even exist when
  the events were originally produced.

**Cost:** eventual consistency (consumer might process the event seconds later — fine for "send a
welcome email," not fine for "confirm payment before shipping"), debugging requires tracing an event
through N async consumers instead of reading a linear call stack, you need a broker (Kafka/SQS/
RabbitMQ) as new infra to run/monitor, and message ordering/dedup/at-least-once-delivery semantics
become real problems you must design for (idempotent consumers, dedup keys).

### Decision shortcut

| Question | Leans toward |
|---|---|
| Small team, unclear domain boundaries, need transactions? | Monolith |
| Need independent scaling/deploys per well-understood domain, multiple teams? | Microservices |
| Producer shouldn't wait for or know about consumers; need fan-out, buffering, replay? | Event-driven |
| None of the above pains exist yet | Monolith (default) — split when a *specific, measured* pain shows up, not speculatively |

The strongest real-world pattern: **start monolith (request/response), extract microservices only
along seams where you've actually felt the pain** (a specific module scaling differently, a
specific team blocked on releases), and **introduce events only where decoupling specifically pays
off** (fan-out to multiple unknown future consumers, absorbing load spikes, audit/replay needs) —
not as a default communication style everywhere.

### How this project fits

Monolith + mostly request/response, with one fire-and-forget async edge (click inserts) that is
architecturally the seed of an event-driven boundary. If this project needed to scale further, the
two most likely splits, following the "split along measured pain" rule above, would be:
1. Separate the redirect service from the shorten service (different traffic profiles, see NGINX
   rate limits).
2. Turn the fire-and-forget click insert into a real published event (`UrlClicked`) so click
   analytics could be consumed by multiple future services without redirect-path changes.

**References:**
- [Martin Fowler: Microservices](https://martinfowler.com/articles/microservices.html)
- [Martin Fowler: MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
- [Martin Fowler: What do you mean by "Event-Driven"?](https://martinfowler.com/articles/201701-event-driven.html)
- [AWS: Monolithic vs Microservices Architecture](https://aws.amazon.com/microservices/)
- [Confluent: Event-Driven Architecture](https://www.confluent.io/learn/event-driven-architecture/)
- [Conway's Law](https://www.melconway.com/Home/Conways_Law.html)
