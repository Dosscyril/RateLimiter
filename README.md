# Rate Limiter — Two Theta Take-Home

A per-client rate limiter built with Java 17 + Spring Boot, enforcing **N requests per T
seconds** using the **sliding window counter** algorithm.

## Quick start

```bash
mvn spring-boot:run
```

Then, in another terminal:

```bash
# allowed
curl -i http://localhost:8080/api/ping -H "X-Client-Id: alice"

# fire the sample scenario from the assignment (8 requests, wait, 2 more)
./demo.sh
```

Every response under `/api/**` carries `X-RateLimit-Limit` and `X-RateLimit-Remaining`
headers. A blocked request gets `HTTP 429` plus a `Retry-After` header (seconds) and a
small JSON body.

Client identity is resolved in this order: `X-Client-Id` header → `clientId` query
param → remote IP. Configure the limit in `src/main/resources/application.properties`:

```properties
ratelimiter.max-requests=5
ratelimiter.window-seconds=10
```

Run the tests:

```bash
mvn test
```

## Algorithm: sliding window counter

I chose the **sliding window counter** (as opposed to fixed window, sliding window log,
token bucket, or leaky bucket) because it's the best fit for what an intern take-home is
actually testing: correctness, per-client isolation, and clear reasoning about trade-offs,
without either the naive bugs of a fixed window or the extra machinery of a full log.

How it works: time is split into fixed-size buckets of length `T`. For each client we keep
only two numbers — the count in the *current* bucket and the count in the *previous*
bucket. To decide "how many requests has this client made in the last `T` seconds,
counting from right now", we don't need the exact timestamps of every past request; we
approximate by assuming the previous bucket's requests were spread evenly across it:

```
estimate = previousBucketCount * overlapFraction + currentBucketCount
overlapFraction = 1 - (elapsedTimeInCurrentBucket / T)
```

A request is allowed if `estimate < N`.

**Why this over the alternatives:**

| Algorithm | Problem it has that sliding window counter avoids |
|---|---|
| Fixed window | Lets a client burst up to `2N` requests around a window boundary (N at the end of one window, N at the start of the next) — the exact "burstiness at edges" problem the prompt calls out. |
| Sliding window log | Fully accurate (stores every request timestamp), but memory scales with request volume per client, and every check requires pruning old entries. Overkill for this scope. |
| Token bucket / leaky bucket | Excellent for smoothing traffic and allowing controlled bursts, but that's a different product decision (deliberately allowing burst capacity) than what the assignment describes (a hard N-per-T cap). |

**Trade-offs I accepted:**
- **Approximation, not exact.** The "requests spread evenly across the previous bucket"
  assumption isn't always true — see edge case 1 below. In practice this is the standard
  trade-off (used by, e.g., Cloudflare and Kong's default limiters) because it's close
  enough to exact while using O(1) memory per client.
- **O(1) memory per client** — just two integers and a timestamp, regardless of request
  volume. This matters at scale far more than log-based exactness.
- **Slightly conservative `Retry-After`.** I return the time left in the current bucket
  as the retry hint. It's a safe *upper bound* — the client could sometimes retry a bit
  sooner — but never tells a client to retry before it's actually likely to succeed.

## Assumptions

- **Client identity** is a string key handed to `allow(clientId)` — the demo resolves it
  from a header/query param/IP, but the core `SlidingWindowCounterRateLimiter` class is
  identity-agnostic and would work identically for user IDs, API keys, or IP addresses.
- **In-memory, single-instance.** State lives in a `ConcurrentHashMap` inside the JVM.
  This is explicitly out of scope per the assignment's time-box, but see "with more time"
  below for how I'd extend it to multiple instances.
- **No client eviction/TTL implemented.** Clients that stop sending requests keep a small
  entry in the map forever. Fine for a demo; not fine for a long-running production
  service with millions of distinct (e.g., IP-based) clients — see below.
- **The limit check and the increment are atomic together** (guarded by a per-client
  lock) — I treat "check and consume" as one indivisible operation rather than two steps,
  which is what makes the concurrency guarantee below hold.

## Edge cases considered

1. **Burst straddling a window boundary.** A client sends 4 requests at t=9.0s (just
   before a 10s boundary) then more right at t=10.0s. A fixed window would see this as
   "bucket resets to 0" and allow a fresh 5 immediately — permitting up to 9 requests in
   about 1 second. The sliding window counter instead keeps most of the previous bucket's
   weight (`overlapFraction ≈ 1.0` right at the boundary) so only one more request gets
   through before blocking. Covered by
   `edgeCase_burstStraddlingTwoWindowsIsThrottledByWeightedEstimate`.

2. **Concurrent requests for the same client at (almost) the same instant.** See
   "Concurrency" below — covered by `concurrentRequestsForSameClientNeverExceedTheLimit`,
   which fires 50 threads at a single client simultaneously and asserts exactly 5 get
   through, not 6+ from a race.

3. **A brand-new client's first request.** `currentBucketStart` starts at `-1` as a
   sentinel so the very first request for any client is always evaluated against an empty
   window rather than crashing or misreading a stale timestamp.

4. **Two clients never share quota.** One client hammering the limiter must not consume
   or corrupt another client's counters — enforced by keying the whole state per client in
   the `ConcurrentHashMap` and locking per-client, not globally.

## Concurrency

Each client gets its own `ClientWindow` object holding a `ReentrantLock`. The read
(compute weighted estimate), decision (compare to limit), and write (increment the
counter) happen inside a single `lock()/unlock()` block, so two threads racing for the
same client's last remaining slot can't both read "4 out of 5 used" and both proceed —
one wins the lock, consumes the slot, and the second sees the updated count and is
correctly blocked.

Locking is **per-client**, not global, so this only serializes requests that are actually
racing for the same client's quota — traffic from different clients still proceeds fully
in parallel (`ConcurrentHashMap.computeIfAbsent` handles safe creation of a client's first
entry without a global lock either).

I verified this isn't just "probably fine" by writing a test that launches 50 threads at
once against a single client with a limit of 5, using a `CountDownLatch` to release them
simultaneously, and asserting the allowed count is exactly 5 — not more (a race would
show up as 6+ admitted) — see `concurrentRequestsForSameClientNeverExceedTheLimit`.

## What I'd do differently with more time

- **Distributed state.** Move the counters into Redis (`INCR` + `EXPIRE`, or a Lua script
  for atomicity) so the limiter works correctly behind multiple app instances/load
  balancers instead of only within one JVM's memory.
- **Client map eviction.** Add a TTL/LRU eviction policy (e.g., Caffeine cache) so
  long-running processes with many transient clients (especially IP-keyed anonymous
  traffic) don't grow the map unboundedly.
- **Per-route or per-tier limits.** Right now the whole `/api/**` surface shares one
  limit; a real API usually wants different limits per endpoint or per subscription tier.
- **A tighter `Retry-After`.** Currently conservative (time left in the current bucket).
  With more time I'd compute the actual time at which the weighted estimate is projected
  to drop below the limit, rather than always waiting out the full bucket.
- **Integration test via `MockMvc`** hitting the real `/api/ping` endpoint through the
  filter, in addition to the unit tests against the core algorithm, to catch any
  header/status wiring bugs the unit tests wouldn't see.

## Sample scenario walkthrough (5 req / 10s)

Client `alice` sends 8 requests in the first 3 seconds, waits 10s, sends 2 more:

- Requests 1–5 (all within the first bucket): **allowed**, remaining counts down 4, 3, 2,
  1, 0.
- Requests 6–8 (same bucket, estimate already at the limit): **blocked**, each with a
  `Retry-After` telling alice how long is left in the current 10s bucket.
- After the 10s wait, alice is now well into the next bucket and the previous bucket's
  weight has decayed substantially (or fully, depending on exact timing) — both of the
  final 2 requests are **allowed**.

`demo.sh` reproduces this against the live app and prints the allow/block decision and
headers for each call.
