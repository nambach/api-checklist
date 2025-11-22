
# API & Service Build — Teaching Edition (Deep Dive) — **Part 5**
**Entries 50–66: Testing, Delivery/Change, Cost/Capacity, Compliance; Go‑Live; Protocol Guides**

---

## 50) Unit Tests (fast correctness checks)
Unit tests validate small pieces of logic quickly and deterministically, catching regressions before they ship. Aim for clear inputs/outputs, cover edge cases and invariants, and avoid external dependencies by using fakes. Keep them fast so they run on every commit, and name tests after the behavior they guarantee. Treat flaky tests as broken features; fix or remove them promptly.  
**Example:** Testing an amount validator: inputs below 1.00 fail with `code="amount_too_small"`, exactly 1.00 passes, and non‑integers are rejected.

## 51) Contract Tests (consumer‑driven confidence)
Contract tests align producers and consumers on the exact request/response shapes and error codes. Consumers publish expectations (a “pact”); providers verify they still satisfy them on every change. This catches breaking changes that ordinary unit/integration tests might miss, especially across teams or companies. Keep contracts versioned and run verification in CI so incompatible changes never merge.  
**Example:** A mobile app’s pact asserts `/refunds/{id}` includes fields `{id, status, amount}` and that `status` is one of `[pending, succeeded, failed]`; the API CI fails if `status` is renamed.

## 52) Integration Tests (wiring and configuration)
Integration tests exercise real components together—your service, database, cache, and message bus—to catch misconfigurations and incompatibilities. Seed deterministic data, isolate tests to avoid cross‑talk, and set realistic timeouts. Run them in a prod‑like environment (containerized CI or ephemeral test environments) and keep logs/traces for debugging. They’re slower than unit tests, so run on PRs or nightly with fail‑fast signals.  
**Example:** Spinning up Postgres and Redis in CI, the test creates an order, issues a refund, and verifies both DB rows and an emitted event on the queue.

## 53) End‑to‑End Smokes & Canaries
E2E smokes exercise the **user journey** through all layers: auth, API, DB, and external dependencies. They run in staging and after deploys to prove that “the basics work.” **Canary releases** send a small slice of production traffic to the new version; if SLOs degrade, automated rollback triggers. Together they reduce the chance of shipping obviously broken builds.  
**Example:** A synthetic monitor logs in, creates a refund, checks its status, and logs out every minute; canary deploy gates on the monitor’s success rate and P95 latency.

## 54) Fuzz & Property‑Based Tests
Fuzz tests throw many semi‑random inputs at parsers and validators to uncover crashes, while **property‑based** tests assert general truths (e.g., sorting twice yields the same result, decoded(encode(x)) == x). These tests find edge cases humans don’t think of. Give them a time budget and seed so failures are reproducible, and target code that handles untrusted input.  
**Example:** Fuzzing the webhook signature verifier with random headers and payload truncations; property test: parsing → serializing → parsing a refund should preserve fields.

## 55) Resilience, Load & Soak Tests
Beyond functional correctness, verify that **timeouts/retries** behave as intended under induced faults and that the system remains stable over hours of load. Inject dependency outages, increase latency, and drop packets; confirm that breakers open and recover, request queues don’t grow without bound, and memory doesn’t leak. These tests are the dress rehearsal for production chaos.  
**Example:** During a 3‑hour soak at 80% of peak, you inject 5‑minute outages to the payment provider; success rate remains within SLO thanks to retries with jitter.

## 56) Infrastructure as Code (IaC)
Represent infrastructure—networks, databases, queues, permissions—as code (Terraform, Pulumi) so changes are reviewable, reproducible, and auditable. Use modules, remote state with locking, and drift detection to catch out‑of‑band edits. Combine with policy‑as‑code to enforce guardrails (e.g., encryption required, public buckets forbidden). Back up state and document how to recover if it’s lost.  
**Example:** A PR adding a new queue shows diff of IAM roles and alarms; CI runs `terraform plan` and policy checks before allowing merge.

## 57) CI/CD Pipeline (build → test → scan → deploy → verify → rollback)
A disciplined pipeline compiles code, runs tests, scans dependencies, signs artifacts, deploys gradually, verifies health, and offers a quick **rollback**. Bake post‑deploy checks (synthetics, metrics thresholds) into the pipeline so a bad rollout automatically halts and reverses. Keep pipelines fast and observable; developers should see where and why a build failed in seconds.  
**Example:** After deploy, the pipeline watches P95 latency and error rate for 10 minutes; if thresholds breach, it rolls back to the last good artifact automatically.

## 58) Release Strategies (canary, blue‑green, feature flags)
Choose a release strategy that fits risk. **Canary** shifts a small percent of traffic and watches metrics; **blue‑green** runs two full stacks and switches traffic atomically; **feature flags** toggle behavior at runtime for targeted rollouts and quick kills. Combine strategies for critical changes (flagged canary) and ensure flags are short‑lived and removed after rollout to avoid config sprawl.  
**Example:** A risky query planner ships behind a flag to 5% of tenants for a day, monitored by dashboards; if stable, it ramps to 100% and the flag is retired.

## 59) Supply Chain Security (provenance, SBOM, scanning)
Attackers increasingly target your build and dependencies. Sign artifacts, generate a **Software Bill of Materials (SBOM)**, pin versions, and scan for known vulnerabilities during CI. Use private registries and verify provenance (SLSA‑style attestations) so production only runs trusted builds. Keep an inventory of transitive dependencies so you can respond quickly to new CVEs.  
**Example:** The deploy step verifies an attestation that the artifact was built by your CI with a specific commit and toolchain; SBOM is stored and checked by policy.

## 60) Consumer Rollout & Compatibility (migration paths)
Plan how API consumers will adopt changes. Provide dual‑read/write periods, compatibility shims, and clear migration guides. Instrument who is still using old versions and nudge them with targeted communication. Never yank a widely used endpoint without notice and an alternative. This reduces breakage and preserves partner goodwill.  
**Example:** For a new refunds index, the service dual‑writes for two weeks, exposes both search endpoints, and emails remaining `v1beta` users as the sunset date approaches.

## 61) Capacity Planning (headroom, multi‑AZ, failover)
Estimate demand (RPS, payload sizes), apply peaks and growth, and reserve **headroom** beyond P95 so bursts don’t tip you into queuing collapse. Deploy across multiple availability zones by default, test N‑1 capacity (losing a node/zone), and configure autoscaling to react to real load signals (in‑flight requests, queue depth) rather than raw CPU alone.  
**Example:** With 300 peak RPS and 250 ms P95, target ~150 concurrent capacity per AZ and ensure remaining AZs can carry traffic if one fails.

## 62) Cost Guardrails (budgets, tags, per‑tenant visibility)
Control cost with tagging, budgets, and dashboards that attribute spend to services and tenants. Set anomaly alerts so you catch runaway jobs or misconfigurations early. Use caching and instance right‑sizing to reduce waste, and track cost per request/user to guide product decisions. Treat cost regressions like performance regressions with reviews and fixes.  
**Example:** A dashboard shows S3 egress spiking for a single tenant; rate limits and a different data export flow bring spend back within budget.

## 63) Hot‑Spot Analysis (find the biggest hogs)
Continuously identify where time and money go: top‑N slow queries, endpoints by P99 latency, highest allocator sites in profiles, and cache hit/miss rates. Focus optimization where it matters; avoid micro‑tuning cold paths. Establish targets (e.g., cache hit rate > 80%) and review trends in performance councils.  
**Example:** Profiling reveals 40% of CPU in JSON serialization; switching to a faster encoder drops P95 by 20% and reduces compute cost.

## 64) Regulatory Scope (GDPR/CCPA/PCI/HIPAA, etc.)
Know which regulations apply based on your data and markets, and translate them into engineering controls (consent storage, encryption, access logs, breach notification). Keep a data inventory and privacy impact assessments up to date. Work with counsel to avoid collecting unnecessary personal data and to design compliant defaults.  
**Example:** Handling card payments? Segregate PCI‑scoped components, never store PAN directly, and use a specialized provider with tokenization.

## 65) Data Subject Rights & DLP (delete/export; prevent leaks)
Users may have the right to access or delete their data. Provide self‑serve export and deletion flows, and ensure deletion propagates to caches, logs, analytics, and backups within defined windows. Add **Data Loss Prevention** checks to block sensitive data from leaving via logs or exports. Test these flows regularly like any critical feature.  
**Example:** A user deletion enqueues tasks that scrub primary DB, Redis, search, and scheduled log retention; dashboards show backlog and success rates.

## 66) Access Reviews & Break‑Glass (least privilege with escape hatch)
Regularly review who has which permissions, remove unnecessary access, and use time‑bound, auditable **break‑glass** credentials for emergencies. Alert on usage of break‑glass and rotate them immediately after. This limits insider risk and mistakes while preserving the ability to repair production quickly.  
**Example:** Quarterly access review removes dormant admin rights; break‑glass account requires ticket reference and triggers alerts on login.

---

## Go‑Live Readiness (final pre‑launch gate)
Before launch, ensure on‑call rotations and runbooks are ready; SLO dashboards are green under production‑like load; synthetic monitors cover critical journeys; error budgets and alert thresholds are reviewed; backups and restore drills meet RPO/RTO; incident communication templates and a status page playbook exist; an AZ/zone failure test passes; and legal/privacy docs are current. A calm launch is a sign your preparation worked.  
**Example:** A game‑day exercise disables one AZ; traffic shifts smoothly, SLOs hold, and on‑call follows the playbook without surprises.

---

## Protocol Quick Guides (expanded)

### REST
Design with resources and HTTP semantics. Use nouns and plural routes (`/orders`), support conditional requests (`ETag`, `If-None-Match`), and document query parameters, filters, and pagination clearly. For partial updates, use `PATCH` with precise rules. Errors follow a uniform schema or RFC Problem Details. Secure with OAuth2, rate limit, and prefer JSON with explicit `Content-Type`.  
**Example:** `GET /orders/{id}` returns 200 with cache validators; `PATCH /orders/{id}` requires `If-Match` and returns 412 on version conflict.

### gRPC
Define clear protobuf messages, never reuse field numbers, and mark removed fields as `reserved`. Set deadlines on every call, enforce max message sizes, and implement streaming with flow control and cancellation. Use `google.rpc.Status` for rich errors and enable health checks and reflection for tooling. Prefer for internal, high‑QPS, low‑latency calls.  
**Example:** A `RefundService` with unary `CreateRefund` and server‑streaming `ListRefunds`, clients pass deadlines and handle `RESOURCE_EXHAUSTED` properly.

### Eventing / Async
Treat events as contracts: version schemas and publish them to a registry. Choose delivery semantics (at‑least‑once is most common) and make consumers idempotent. Use ordering/shard keys when sequence matters and provide DLQs with replay. Design handlers to be **replay‑safe** and to checkpoint progress so they can recover after failures.  
**Example:** An `refund.created` event includes `id`, `order_id`, `created_at` and schema version; consumers dedupe via a persistent `last_seen_event_id`.
