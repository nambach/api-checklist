
# API & Service Build — Teaching Edition (Deep Dive) — **Part 3**
**Entries 28–38: Reliability/Resilience & Performance/Scalability**

---

## 28) SLOs & Error Budgets (targets that guide change)
Service Level Objectives translate product promises into measurable reliability targets and create **error budgets**—the allowed failure time in a window. For example, 99.9% monthly availability permits ~43.8 minutes of unavailability; consuming that budget should slow the rate of risky changes until stability improves. Define SLOs per critical endpoint (availability and P95/P99 latency), measure them from the user’s perspective, and alert on **burn rate** (how fast you’re spending budget) rather than single spikes. This turns reliability into a deliberate trade‑off, not a vague aspiration.  
**Example:** `GET /refunds/{id}` SLO: 99.95% success, P95 < 250 ms. Alert if 2‑hour burn rate exceeds 14× and 24‑hour burn exceeds 6× the budget.

## 29) Timeouts, Retries with Jitter, Circuit Breakers
Distributed systems fail in partial, messy ways. Set **timeouts** on every call so threads don’t hang indefinitely; the client timeout should be shorter than the server’s to avoid retries that race completions. Use **exponential backoff with jitter** so retry storms don’t synchronize; retry only **idempotent** operations. Add **circuit breakers** to stop hammering a failing dependency: after enough errors, the breaker opens and immediately rejects calls; later it half‑opens to probe if recovery has occurred. These patterns prevent cascading failures and keep latency predictable.  
**Example (pseudocode):**
```
for attempt in 0..5:
  try request(deadline=200ms)
  except transient:
    sleep random(0, 2**attempt * 100ms)
breaker.trip_on(error_rate>50% over 10s); half_open_after(30s)
```

## 30) Bulkheads & Isolation (contain blast radius)
**Bulkheads** partition resources so one hot path can’t starve others. Use separate connection pools and thread pools per dependency; cap queue lengths per tenant or request class; and reserve capacity (or higher priority lanes) for critical traffic like health checks, payments, or admin actions. Combine with admission control so low‑priority requests shed first during overload. This containment keeps the rest of the ship afloat when one compartment floods.  
**Example:** Database writes have their own pool; background exports cannot exceed 20% of worker threads; health checks bypass user queues entirely.

## 31) Load Shedding & Overload Protection
When demand exceeds safe throughput, it’s better to quickly refuse some work than to slow everyone into timeouts. Implement **load shedding**: reject early with `503 Service Unavailable` and `Retry-After`, or use adaptive concurrency limits that cap in‑flight requests based on observed latency. Prefer to shed low‑priority traffic, large queries, and expensive filters first. Combine with clear client guidance to back off, and verify that shedding leaves enough capacity for recovery.  
**Example:** Your API tracks queue depth and P95 latency; if either crosses a threshold, it returns 503 for non‑critical endpoints until the system recovers.

## 32) Startup Readiness & Graceful Shutdown
On startup, an instance should not receive traffic until it is **ready** (dependencies reachable, caches warm, migrations finished). On shutdown (e.g., a deploy), it should **drain**: stop accepting new connections, complete in‑flight requests, and release resources before exit. In orchestrators like Kubernetes, implement separate **liveness** (is the process healthy?) and **readiness** (can it serve?) probes; handle SIGTERM by closing listeners and waiting for requests to finish. This prevents black‑hole errors and reduces user‑visible failures during deploys.  
**Example:** A pod reports `/ready` only after it successfully connects to DB and cache; on SIGTERM it closes the listener, waits 30s, then exits.

## 33) Chaos & Fault Injection (practice failure safely)
Don’t wait for production to teach you failure modes. **Chaos experiments** deliberately inject latency, errors, and dependency outages in staging and controlled canaries to validate that timeouts, retries, breakers, and fallbacks work as designed. Start small and scoped, define a hypothesis (“payment retries mask a transient PSP outage”), run, and measure. Feed learnings back into code and runbooks. Practiced failure turns incidents into routine operations rather than emergencies.  
**Example:** In staging, inject 300 ms latency into the payment gateway and confirm user checkout still meets P95 targets via retry with jitter and circuit breaker behavior.

## 34) Performance Budgets (P50/P95/P99) & Tail Latency
Set explicit latency budgets for each endpoint and step (auth, DB, downstream) and monitor **tail** percentiles, not just averages. Tail latency (P99) often dominates user perception in multi‑hop calls and is sensitive to queuing and GC. Budgets focus engineering on practical improvements (e.g., caching a hot read; reducing fan‑out) and let you evaluate regressions objectively in CI and canaries. Include payload sizes in budgets; bigger responses inflate tails.  
**Example:** `GET /orders/{id}` budget: P50≤60 ms, P95≤200 ms, P99≤400 ms; response size ≤ 32 KiB; DB query ≤ 50 ms.

## 35) Load Testing (ramp & soak)
**Ramp tests** increase traffic to discover maximum sustainable throughput and see where latency curves bend. **Soak tests** hold steady high load for hours to reveal memory leaks, connection exhaustion, and slow state growth. Automate both in CI or a pre‑deploy stage with realistic data and warm caches; define pass/fail thresholds linked to your SLOs. Use the results to tune autoscaling and resource limits.  
**Example:** A nightly soak runs at 80% of expected peak for 3 hours; failing if error rate >0.1%, P95 > target by 20%, or memory growth exceeds 5%.

## 36) N+1 Queries & Index Hygiene
The **N+1** pattern happens when you fetch a list (N items) and then query again per item, creating N additional queries. This kills performance at scale. Fix by preloading related data, using joins, or batching. Ensure every common filter or sort has an appropriate **index**, and verify with your database’s `EXPLAIN` plans. Cache can hide issues in dev but will betray you under load; fix root causes early.  
**Example:** Instead of querying each refund’s order, do `SELECT * FROM orders WHERE id IN (...)` using the set of order IDs from the refunds page.

## 37) Throughput Model (Little’s Law) & Limits
Throughput planning benefits from simple queueing math. **Little’s Law** says `L = λ × W`: average in‑flight requests equal arrival rate (λ) times average time in system (W). If you accept 200 RPS and average 200 ms, you’ll have ~40 concurrent in‑flight requests; ensure your worker threads and DB pools can handle that with headroom. Bound **fan‑out** (downstream calls per request), cap payload sizes, and set queue length limits to prevent unbounded growth during spikes.  
**Example:** With 50 worker threads and P95 of 300 ms, a safe cap might be ~150 in‑flight requests, implying a ~500 RPS ceiling without shedding.

## 38) Content Size & Compression (cost vs. CPU)
Large payloads dominate network time and inflate tail latency. Cap response sizes, paginate aggressively, and enable compression (Gzip/Brotli) for text and JSON. Avoid recompressing already compressed media (JPEG/MP4), and prefer streaming for large downloads/uploads to avoid buffering entire files in memory. Measure CPU overhead from compression; it’s often worth it for JSON payloads above a few kilobytes.  
**Example:** Responses include `Content-Encoding: gzip` when `Accept-Encoding` allows it; server enforces `max_response_bytes=1MB` and streams files via ranged requests.
