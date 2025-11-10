
# API & Service Build **Notebook** (Beginner‑Friendly Markdown)

> A practical, self‑contained guide you can **check off** as you design an API and build the service behind it.  
> **Goal:** make every term understandable, with plain‑English explanations, copy‑paste examples, and mini checklists.

---

## How to use this notebook

- Skim the **Table of Contents** and jump to what you need.
- Each item includes: **What it is → Why it matters → How to do it → Example → Checklist**.
- Tick the `- [ ]` boxes as you complete work. (In many markdown tools, you can turn them into `- [x]` as you go.)
- Use the **Glossary** for any unfamiliar word.
- This file is yours — adapt it to your stack and policies.

---

## Table of Contents

1. [API Definition (interface & behavior)](#api-definition-interface--behavior)  
   1. [Scope & Semantics](#scope--semantics)  
   2. [Protocol & Shape](#protocol--shape)  
   3. [AuthN/AuthZ](#authnauthz)  
   4. [Documentation & DX](#documentation--dx)  
2. [Service Build (internals & ops)](#service-build-internals--ops)  
   1. [Architecture & Data](#architecture--data)  
   2. [Reliability & Resilience](#reliability--resilience)  
   3. [Performance & Scalability](#performance--scalability)  
   4. [Security & Privacy](#security--privacy)  
   5. [Observability](#observability)  
   6. [Testing Strategy](#testing-strategy)  
   7. [Delivery & Change Management](#delivery--change-management)  
   8. [Cost & Capacity](#cost--capacity)  
   9. [Compliance & Governance](#compliance--governance)  
3. [Go‑Live Readiness](#go-live-readiness)  
4. [Protocol Quick Guides](#protocol-quick-guides)  
   - [REST](#rest-quick-guide) | [gRPC](#grpc-quick-guide) | [Eventing/Async](#eventingasync-quick-guide)  
5. [Templates & Snippets](#templates--snippets)  
6. [Glossary (simple definitions)](#glossary-simple-definitions)  
7. [Scorecard (one‑page checklist)](#scorecard-one-page-checklist)

---

## API Definition (interface & behavior)

### Scope & Semantics

#### 1) Problem statement, success metrics, out‑of‑scope
- **What:** Write one paragraph that says *who needs what and why*, plus 3 success metrics (e.g., adoption, latency). Also write what **won’t** be done now.
- **Why:** Prevent scope creep, set expectations, guide trade‑offs.
- **How:** Fill the template below.
- **Template:**  
  ```md
  **Problem:** Merchants need to issue refunds via API so support can automate workflows.  
  **Users:** Internal support bot, partner apps.  
  **Success by 60 days:** 200 DAU, P95 < 400 ms, refund error rate < 0.5%.
  **Out of scope:** Cross‑currency refunds, partial capture reversal.
  ```
- **Checklist**
  - [ ] One‑paragraph problem
  - [ ] Three measurable success metrics
  - [ ] Clear out‑of‑scope list

#### 2) Resource model (nouns) & operations (verbs)
- **What:** List your main **things** (e.g., `orders`, `refunds`) and allowed **actions** (`create`, `list`, `get`, `update`, `cancel`).
- **Why:** Keeps API consistent and predictable.
- **How:** Write a table of resources and operations.
- **Example:**  
  | Resource | Key fields | Operations |
  |---|---|---|
  | `refund` | `id`, `order_id`, `amount`, `status` | `POST /refunds`, `GET /refunds/{id}`, `GET /refunds?order_id=...`, `POST /refunds/{id}:cancel` |
- **Checklist**
  - [ ] Resource list with key fields
  - [ ] Explicit allowed operations per resource

#### 3) Consistency model
- **What:** How quickly reads reflect writes. Common choices: **strong** (immediate), **read‑your‑writes** (caller sees own recent writes), **eventual** (may lag).
- **Why:** Sets expectations for UX and caches; avoids “why didn’t I see my update?” confusion.
- **How:** Decide per endpoint and state it in docs.
- **Example:** “`GET /refunds/{id}` is **strongly consistent**. `GET /orders/{id}/refunds` may be **eventually consistent** up to 2 seconds.”
- **Checklist**
  - [ ] Each read endpoint states its consistency
  - [ ] Clients told how to detect freshness (e.g., ETag/version)

#### 4) Idempotency (safe retries)
- **What:** The same request sent again has the **same effect** as the first time.
- **Why:** Network hiccups cause retries; idempotency prevents double‑charges/duplicates.
- **How:** Require an **Idempotency‑Key** header for non‑GET writes; store request hash + result for a window (e.g., 24h).
- **Example:**  
  ```http
  POST /refunds
  Idempotency-Key: 3bbf9b0c-...

  { "order_id": "o_123", "amount": 1000 }
  ```
- **Checklist**
  - [ ] Non‑GET endpoints accept `Idempotency-Key`
  - [ ] Server dedupes and returns the first result

#### 5) Concurrency control
- **What:** Protect against **lost updates** when two writers edit the same thing.
- **Why:** Prevent one client from silently overwriting another’s changes.
- **How:** Use **optimistic concurrency** with version/`ETag` and `If‑Match`.
- **Example:**  
  ```http
  GET /orders/123
  ETag: "v7"

  PATCH /orders/123
  If-Match: "v7"
  { "note": "Ship ASAP" }
  ```
- **Checklist**
  - [ ] Resources have version/ETag
  - [ ] Update endpoints honor `If‑Match`

#### 6) Time semantics
- **What:** Use standard time formats and rules.
- **Why:** Time bugs are sneaky (zones, DST). Consistency avoids them.
- **How:** Timestamps in **RFC 3339 UTC** (`2025-01-30T14:57:00Z`). Durations/intervals in **ISO‑8601**. Allow small **clock skew** (e.g., ±5m).
- **Checklist**
  - [ ] All timestamps UTC RFC 3339
  - [ ] Durations ISO‑8601 (`PT30S`, `P1D`)
  - [ ] Server tolerates client clock skew

---

### Protocol & Shape

#### 7) Transport choice (REST/JSON, gRPC/Protobuf, GraphQL, events)
- **What:** The “wire” your API uses.
- **Why:** Impacts performance, tooling, and developer experience.
- **How:** Pick based on needs:
  - **REST/JSON:** human‑readable, caching, broad tooling.
  - **gRPC/Protobuf:** binary, fast, great for internal service‑to‑service.
  - **GraphQL:** flexible querying from browsers/mobile; server must protect against heavy queries.
  - **Events:** async messaging for notifications/streams.
- **Checklist**
  - [ ] Transport chosen and justified
  - [ ] Security & limits suited to transport

#### 8) Versioning policy
- **What:** Rules for changes over time.
- **Why:** Avoid breaking existing clients.
- **How:** Make **additive** changes in the same major version; use a new **major** for breaking changes; publish a **deprecation window**.
- **Example:** “v1 guarantees no breaking changes; v2 will be announced 180 days before retirement of v1.”
- **Checklist**
  - [ ] Major/minor policy written
  - [ ] Deprecation timeline/process in docs

#### 9) Pagination
- **What:** Return large lists in pages using a **cursor** (opaque token) and `limit`.
- **Why:** Protects servers and clients; stable paging.
- **How:** Include `next_page_token` in responses; sort by a stable key (e.g., `created_at,id`).
- **Example:**  
  ```http
  GET /refunds?limit=50&page_token=eyJvZmZzZXQiOiIxNzAw...

  200 OK
  { "items": [...], "next_page_token": "..." }
  ```
- **Checklist**
  - [ ] Cursor‑based paging
  - [ ] Stable sort and `limit` caps

#### 10) Filtering & sorting
- **What:** Parameters to narrow and order results safely.
- **Why:** Avoids unbounded scans that hurt performance.
- **How:** Whitelist fields and operators; combine with `limit`.
- **Example:** `GET /orders?status=paid&created_after=2025-01-01&sort=-created_at`
- **Checklist**
  - [ ] Documented filters and sorts
  - [ ] Guardrails to prevent full‑table scans

#### 11) Error model
- **What:** A **single, typed** error shape so clients handle problems consistently.
- **Why:** Clear debugging; better UX.
- **How:** Return HTTP code + JSON body with fields like `type`, `code`, `message`, `details`, `request_id`.
- **Example:**  
  ```json
  {
    "type": "validation_error",
    "code": "amount_too_small",
    "message": "Minimum is 1.00",
    "details": { "field": "amount", "min": 100 },
    "request_id": "req_abc123"
  }
  ```
- **Checklist**
  - [ ] One error schema used everywhere
  - [ ] Error catalog documented

#### 12) Partial success for batch ops
- **What:** Some items succeed, some fail.
- **Why:** Clients can retry only failures.
- **How:** Include `successes` and `errors` arrays.
- **Example:**  
  ```json
  { "successes": [{"id":"r_1"}], "errors": [{"index":1,"code":"invalid"}] }
  ```
- **Checklist**
  - [ ] Batch endpoints specify partial‑success format

#### 13) Rate limiting & quotas
- **What:** Control request **rate** per client/user to keep the system healthy.
- **Why:** Prevent overload and fair use.
- **How:** Token bucket or leaky bucket; return 429 with `Retry‑After` and limit headers.
- **Example headers:**  
  ```http
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 72
  X-RateLimit-Reset: 1735689600
  Retry-After: 12
  ```
- **Checklist**
  - [ ] Limits defined per principal (user/app/tenant)
  - [ ] 429 behavior and headers documented

#### 14) Headers: tracing, idempotency, caching, content type
- **What:** Standard headers to correlate and cache.
- **Why:** Speeds debugging and performance.
- **How:** Accept/emit: `traceparent`, `Idempotency-Key`, `ETag`, `Cache-Control`, `Content-Type: application/json`.
- **Checklist**
  - [ ] Trace/correlation ID flows end‑to‑end
  - [ ] Caching headers used where safe

#### 15) Localization
- **What:** User‑visible text vs. machine‑readable codes.
- **Why:** Avoid parsing human text; keep errors stable.
- **How:** Put **stable codes** in APIs; translate only in UI.
- **Checklist**
  - [ ] APIs return codes/enums, not prose meant for UI

---

### AuthN/AuthZ

#### 16) Authentication (AuthN)
- **What:** Verifying **who** is calling. Use OAuth2/OIDC for users; mTLS or signed JWTs for service‑to‑service.
- **Why:** Only legit callers get in.
- **How:** Pick grant types (client creds, auth code), rotate keys, support token introspection.
- **Checklist**
  - [ ] Auth method(s) chosen and documented
  - [ ] Key/secret rotation policy

#### 17) Authorization (AuthZ)
- **What:** What a caller is **allowed** to do (RBAC = roles; ABAC = attributes/conditions).
- **Why:** Least privilege and tenant isolation.
- **How:** Define resources, actions, and policy checks; default to “read your own.”
- **Checklist**
  - [ ] Policy model defined (RBAC/ABAC)
  - [ ] Tenancy boundaries enforced in queries

#### 18) Secret handling & logging
- **What:** Keep secrets out of URLs and logs; mask PII in logs.
- **Why:** URLs leak; logs are long‑lived.
- **How:** Pass secrets in headers/bodies; log with structured fields and masking.
- **Checklist**
  - [ ] No secrets in URLs
  - [ ] PII/log masking rules in place

---

### Documentation & DX (Developer Experience)

#### 19) Executable spec
- **What:** OpenAPI (for REST) or Protobuf (for gRPC) with comments, examples, and error catalog.
- **Why:** Single source of truth; auto‑gen SDKs and mocks.
- **How:** Keep spec in repo; validate in CI.
- **Checklist**
  - [ ] Spec complete and versioned
  - [ ] Examples and error catalog embedded

#### 20) Quickstart
- **What:** A 5‑minute path to “Hello World” (auth + one call).
- **Why:** Reduces friction and support tickets.
- **How:** Provide `curl` and one SDK snippet.
- **Checklist**
  - [ ] Minimal auth flow shown
  - [ ] Sample request/response

#### 21) Change log & deprecations
- **What:** One page tracking changes and timelines.
- **Why:** Consumers plan upgrades.
- **How:** Keep dated entries; email/webhooks for breaking changes.
- **Checklist**
  - [ ] Changelog page exists
  - [ ] Deprecation notifications defined

#### 22) Mocks & SDKs
- **What:** Mock server/Postman collection and generated clients.
- **Why:** Faster onboarding and testing.
- **How:** Publish artifacts beside docs.
- **Checklist**
  - [ ] Mock/collection available
  - [ ] SDKs for top languages

---

## Service Build (internals & ops)

### Architecture & Data

#### 23) Data model review
- **What:** Keys, indexes, and how data will be queried.
- **Why:** Prevents slow queries and data anomalies.
- **How:** Map **access patterns** → indexes; write sample queries.
- **Checklist**
  - [ ] Primary keys & secondary indexes defined
  - [ ] Access patterns validated

#### 24) Migrations (expand → migrate → contract)
- **What:** A safe 3‑step rollout: **add** new columns, **backfill**, then **remove** old ones.
- **Why:** Zero‑downtime schema changes.
- **How:** Gate code by feature flags; backfill safely.
- **Checklist**
  - [ ] Plan for expand/migrate/contract
  - [ ] Backfill and rollback steps written

#### 25) Identifiers
- **What:** Unique IDs; sometimes sortable (by time).
- **Why:** Avoid collisions; enable time‑ordered queries.
- **How:** Use UUIDv7/ULID for sortable IDs where helpful.
- **Checklist**
  - [ ] ID format chosen (UUIDv7/ULID/DB seq)
  - [ ] No guessable identifiers for sensitive data

#### 26) Caching
- **What:** Store copies (CDN, edge, service memory, DB cache) to speed reads.
- **Why:** Lower latency and cost.
- **How:** Define what to cache, TTLs, and invalidation/signals.
- **Checklist**
  - [ ] Cache layers and TTLs chosen
  - [ ] Invalidation rules documented

#### 27) State boundaries
- **What:** What is stateless vs. where session/state lives.
- **Why:** Scalability and reliability.
- **How:** Keep services stateless; store state in DB/redis/object storage.
- **Checklist**
  - [ ] Stateless by default
  - [ ] Clear state stores per kind of data

---

### Reliability & Resilience

#### 28) SLOs & error budgets
- **What:** Target reliability (e.g., 99.9% availability) and allowed error time (budget).
- **Why:** Guides release pace and incident response.
- **How:** Define per endpoint latency/availability; tie alerts to SLOs.
- **Checklist**
  - [ ] SLOs per critical endpoint
  - [ ] Error budget policy agreed

#### 29) Timeouts, retries, circuit breakers
- **What:** Timeouts stop waiting; retries reattempt; breakers stop cascades.
- **Why:** Prevents stuck threads and meltdowns.
- **How:** Set **shorter client timeouts** than server; retry **idempotent** ops with backoff/jitter; open breaker on high failure.
- **Checklist**
  - [ ] Timeouts on every hop
  - [ ] Safe retry policy; jitter on backoff
  - [ ] Circuit breaker thresholds

#### 30) Bulkheads & isolation
- **What:** Separate resource pools/queues per dependency or tenant.
- **Why:** One slow dependency shouldn’t sink the ship.
- **How:** Connection pools, thread pools, queue partitioning.
- **Checklist**
  - [ ] Pools per critical dependency
  - [ ] Isolation tests in staging

#### 31) Load shedding & overload protection
- **What:** Say “no” early when overwhelmed.
- **Why:** Preserve service for most users.
- **How:** 503 with `Retry‑After`, priority lanes, queue limits.
- **Checklist**
  - [ ] Shed strategy defined
  - [ ] Priority rules documented

#### 32) Startup/shutdown hooks
- **What:** Health gates on start; graceful drain on shutdown.
- **Why:** Avoid serving before ready; avoid dropping in‑flight requests.
- **How:** Readiness/liveness probes; connection drain on SIGTERM.
- **Checklist**
  - [ ] Readiness blocks traffic until warm
  - [ ] Graceful shutdown implemented

#### 33) Chaos & fault injection
- **What:** Intentionally add latency/errors to test resilience.
- **Why:** Find weaknesses before production does.
- **How:** Enable in staging and limited production canaries.
- **Checklist**
  - [ ] Fault injection plan
  - [ ] Results feed into fixes

---

### Performance & Scalability

#### 34) Perf budgets
- **What:** Target latency (P50/P95) per endpoint and resource usage.
- **Why:** Keeps UX snappy and costs in check.
- **How:** Measure, set targets, track in dashboards.
- **Checklist**
  - [ ] P50/P95 per critical endpoint
  - [ ] CPU/mem/IO budgets documented

#### 35) Load testing (ramp & soak)
- **What:** Simulate increasing traffic and long steady traffic.
- **Why:** Catch bottlenecks and memory leaks.
- **How:** Automate in CI; define pass/fail thresholds.
- **Checklist**
  - [ ] Ramp test and soak test scripts
  - [ ] Regressions gate releases

#### 36) N+1 and indexes
- **What:** Avoid many tiny queries; ensure indexes support your reads.
- **Why:** Massive slowdowns otherwise.
- **How:** Profile hot paths; add `JOIN`/preload; verify index usage.
- **Checklist**
  - [ ] N+1 eliminated on hot paths
  - [ ] Slow queries have proper indexes

#### 37) Throughput model & limits
- **What:** Expected QPS, payload sizes, fan‑out, queue depth.
- **Why:** Capacity planning and safe limits.
- **How:** Document assumptions and enforce server caps.
- **Checklist**
  - [ ] QPS & payload limits set
  - [ ] Fan‑out and queue depth bounded

#### 38) Content size & compression
- **What:** Cap response sizes; compress where it helps.
- **Why:** Avoid timeouts and high bandwidth.
- **How:** Gzip/Brotli for text; streaming for large payloads.
- **Checklist**
  - [ ] Max payload sizes defined
  - [ ] Compression enabled when appropriate

---

### Security & Privacy

#### 39) Threat model (STRIDE) & mitigations
- **What:** Systematically list threats (Spoofing, Tampering, Repudiation, Info disclosure, DoS, Elevation).
- **Why:** Don’t miss obvious risks.
- **How:** Draw data flows; list threats; map to controls.
- **Checklist**
  - [ ] Threats documented per data flow
  - [ ] Mitigations mapped and owned

#### 40) TLS everywhere; mTLS internal
- **What:** Encrypt in transit; mutual TLS for service‑to‑service.
- **Why:** Protect data on the wire; authenticate services.
- **How:** Modern TLS, cert rotation, HSTS for web.
- **Checklist**
  - [ ] TLS enforced externally
  - [ ] mTLS for internal calls (where feasible)

#### 41) Input validation & SSRF protection
- **What:** Validate/normalize inputs; block server‑side request forgery on egress.
- **Why:** Prevent injection and data exfiltration.
- **How:** Schema validation; allow‑list outbound hosts; no raw URL fetches from user input.
- **Checklist**
  - [ ] Strong input validation
  - [ ] Egress allow‑list and metadata stripping

#### 42) Secrets management
- **What:** Store secrets in a vault/KMS; rotate regularly.
- **Why:** Reduce blast radius.
- **How:** No plaintext in env vars or code; short‑lived credentials.
- **Checklist**
  - [ ] Vault/KMS integrated
  - [ ] Rotation automation

#### 43) Data classification & lifecycle
- **What:** Classify data (public/internal/sensitive) and set retention/deletion.
- **Why:** Meet laws and reduce risk.
- **How:** Tag fields with sensitivity; define retention; implement deletion flows.
- **Checklist**
  - [ ] Data classes defined
  - [ ] Retention & deletion implemented

#### 44) Browser‑specific protections (if applicable)
- **What:** CORS, CSRF, and security headers (CSP, HSTS).
- **Why:** Prevent web attack classes.
- **How:** Tight CORS allow‑list; CSRF tokens for cookies; strict CSP.
- **Checklist**
  - [ ] CORS allow‑list only
  - [ ] CSRF protection in place
  - [ ] CSP/HSTS headers set

---

### Observability

#### 45) Structured logs & trace IDs
- **What:** JSON logs with a `trace_id` to tie requests across services.
- **Why:** Faster debugging.
- **How:** Use `traceparent` header; include `request_id`, user/tenant (non‑PII) where needed.
- **Checklist**
  - [ ] JSON logs with correlation IDs
  - [ ] Log sampling and retention policy

#### 46) Metrics (RED & USE)
- **What:** **RED:** Rate, Errors, Duration for APIs. **USE:** Utilization, Saturation, Errors for resources.
- **Why:** Standard views keep teams aligned.
- **How:** Export metrics per endpoint and resource; alert on SLOs.
- **Checklist**
  - [ ] RED for all endpoints
  - [ ] USE for key resources (CPU, DB, queues)

#### 47) Distributed tracing
- **What:** Spans for each outbound call; link to logs.
- **Why:** See where time is spent.
- **How:** Propagate trace headers; add tags (endpoint, tenant, result).
- **Checklist**
  - [ ] Tracing enabled with outbound spans
  - [ ] Exemplars link logs ↔ traces

#### 48) Dashboards, alerts, runbooks
- **What:** Visuals and alerts with how‑to‑fix steps.
- **Why:** Cut MTTR; reduce alert fatigue.
- **How:** Dashboard per SLO; alerts with owner and runbook.
- **Checklist**
  - [ ] Actionable alerts (no noise)
  - [ ] Runbooks up‑to‑date

#### 49) Audit logs
- **What:** Tamper‑evident logs of sensitive actions.
- **Why:** Security and compliance.
- **How:** Append‑only store; access controlled; regular review.
- **Checklist**
  - [ ] Audit log coverage defined
  - [ ] Storage is tamper‑evident

---

### Testing Strategy

#### 50) Unit tests
- **What:** Test small pieces of logic.
- **Why:** Prevent regressions cheaply.
- **How:** Cover edge cases and invariants.
- **Checklist**
  - [ ] High‑value logic covered
  - [ ] Edge cases included

#### 51) Contract tests (consumer‑driven)
- **What:** Ensure the API/event contract matches real consumers’ expectations.
- **Why:** Protect against breaking changes.
- **How:** Pact‑style or schema‑based tests run in CI.
- **Checklist**
  - [ ] Producer & consumer tests in CI
  - [ ] Schemas versioned

#### 52) Integration tests
- **What:** Test with real deps or faithful fakes.
- **Why:** Catch wiring/config issues.
- **How:** Stand up minimal stack; seed data.
- **Checklist**
  - [ ] Critical flows tested end‑to‑end
  - [ ] Realistic test data

#### 53) E2E smoke & canaries
- **What:** Simple “does it work” tests in prod‑like env; canary deploy checks.
- **Why:** Catch last‑mile issues.
- **How:** Synthetic monitors hit key endpoints; verify canary metrics.
- **Checklist**
  - [ ] Synthetic checks cover core journeys
  - [ ] Canary gates deployment

#### 54) Fuzz & property‑based tests
- **What:** Randomized inputs and math‑like property checks.
- **Why:** Finds weird edge cases.
- **How:** Generate inputs; assert invariant properties.
- **Checklist**
  - [ ] Fuzz on parsers/validators
  - [ ] Properties defined for critical logic

#### 55) Resilience, load, and soak tests
- **What:** Test timeouts, retries; sustained traffic.
- **Why:** Validate robustness at scale.
- **How:** Inject faults; run hours‑long soaks.
- **Checklist**
  - [ ] Retry/timeout scenarios covered
  - [ ] Soak tests pass without leaks

---

### Delivery & Change Management

#### 56) Infrastructure as Code (IaC)
- **What:** Define infra in code (Terraform/Pulumi) with reviews.
- **Why:** Reproducible, auditable infra.
- **How:** PR reviews, drift detection, state backups.
- **Checklist**
  - [ ] IaC repo with reviews
  - [ ] Drift detection enabled

#### 57) CI/CD pipeline
- **What:** Build → test → security scan → deploy → verify → rollback.
- **Why:** Safe, fast releases.
- **How:** Automate each stage; keep rollback simple.
- **Checklist**
  - [ ] Verified pipeline with gates
  - [ ] One‑click rollback

#### 58) Release strategies
- **What:** Canary, blue‑green, feature flags.
- **Why:** Reduce risk of bad deploys.
- **How:** Route small % traffic; keep dual stacks until healthy.
- **Checklist**
  - [ ] Canary or blue‑green plan
  - [ ] Flags for risky features

#### 59) Supply chain security
- **What:** Artifact provenance and SBOM; dependency scanning.
- **Why:** Guard against compromised builds.
- **How:** Sign artifacts; generate SBOM; scan deps.
- **Checklist**
  - [ ] Artifact signing
  - [ ] SBOM + scans in CI

#### 60) Rollout for consumers
- **What:** Communicate changes; dual‑read/write during transitions.
- **Why:** Smooth migrations.
- **How:** Feature flags, compatibility layers, migration guides.
- **Checklist**
  - [ ] Consumer rollout plan
  - [ ] Dual‑write/read where needed

---

### Cost & Capacity

#### 61) Capacity planning
- **What:** Headroom for P95 load plus bursts; multi‑AZ baseline.
- **Why:** Avoid brown‑outs.
- **How:** Model CPU/mem/IO; auto‑scale policies.
- **Checklist**
  - [ ] Headroom targets set
  - [ ] Multi‑AZ and autoscaling configured

#### 62) Cost guardrails
- **What:** Budgets, alerts, and per‑tenant cost visibility.
- **Why:** Prevent surprises; enable chargeback.
- **How:** Tagging; dashboards; anomaly alerts.
- **Checklist**
  - [ ] Budget alerts
  - [ ] Cost per tenant/feature tracked

#### 63) Hot‑spot analysis
- **What:** Find the top resource hogs (queries, allocators).
- **Why:** Focus optimization.
- **How:** Top‑N dashboards; cache hit rate monitors.
- **Checklist**
  - [ ] Top‑N views in place
  - [ ] Cache hit rate tracked

---

### Compliance & Governance (if applicable)

#### 64) Regulatory scope
- **What:** Which laws apply (GDPR/CCPA/PCI/HIPAA).
- **Why:** Avoid violations and fines.
- **How:** List applicable regs; map controls.
- **Checklist**
  - [ ] Applicable regulations identified
  - [ ] Controls mapped to requirements

#### 65) Data subject rights & DLP
- **What:** Export/delete personal data; prevent leaks.
- **Why:** Legal obligation and trust.
- **How:** Self‑service exports; deletion queues; DLP checks on logs/exports.
- **Checklist**
  - [ ] Export/delete flows implemented
  - [ ] DLP scanning configured

#### 66) Access reviews & break‑glass
- **What:** Regular audits of who can do what; emergency access process.
- **Why:** Least privilege and safe emergencies.
- **How:** Quarterly reviews; audit trail for break‑glass usage.
- **Checklist**
  - [ ] Scheduled access reviews
  - [ ] Documented break‑glass process

---

## Go‑Live Readiness

- [ ] On‑call rotation and escalation paths are ready; runbooks linked
- [ ] SLO dashboards green in staging with production‑like traffic
- [ ] Synthetic monitors for critical user journeys
- [ ] Error budgets and alert thresholds reviewed
- [ ] Backups & restore drills passed; **RPO**/**RTO** verified
- [ ] Incident comms templates ready; status page playbook exists
- [ ] Business continuity test (kill a zone; system still serves)
- [ ] Legal/privacy docs current; ToS/DPAs done if needed

---

## Protocol Quick Guides

### REST Quick Guide

- **Resource naming:** use **nouns** and plurals (`/orders`, `/orders/{id}/refunds`).  
- **HTTP semantics:** GET/HEAD = safe; PUT/DELETE = idempotent; POST = create/unsafe.  
- **Partial updates:** use **PATCH**; document rules.  
- **Caching:** `ETag`/`Last-Modified`; `Cache-Control: max-age=..., stale-while-revalidate=...`; support 304 Not Modified.  
- **Errors:** Use **Problem Details** style fields (`type`, `title`, `status`, `detail`, `instance`) or your standard schema.

### gRPC Quick Guide

- **Compatibility:** Never reuse field numbers; if removed, mark as `reserved`. Prefer adding optional fields.
- **Deadlines:** Every call sets a deadline (timeout); server enforces max message sizes.
- **Streaming:** Implement backpressure/cancellation for client/server streaming.
- **Errors:** Use `google.rpc.Status` with rich details; map to client UX.
- **Ops:** Enable health checking (`grpc.health.v1`) and server reflection for tooling.

### Eventing/Async Quick Guide

- **Schema:** Version your event schema; keep a registry.
- **Delivery:** Pick semantics (at‑least‑once is common); ensure **consumer idempotency**.
- **Ordering:** Use ordering/shard keys where needed.
- **DLQ:** Dead‑letter queue with replay; make handlers **replay‑safe** (dedupe or no side‑effects).

---

## Templates & Snippets

### A) Error schema (JSON)
```json
{
  "type": "https://errors.example.com/validation",
  "code": "invalid_parameter",
  "message": "Parameter 'amount' must be >= 1.00.",
  "details": { "field": "amount", "min": 100 },
  "request_id": "req_123"
}
```

### B) OpenAPI snippet (YAML)
```yaml
openapi: 3.0.3
info:
  title: Refund API
  version: 1.0.0
paths:
  /refunds:
    post:
      summary: Create a refund
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateRefundRequest'
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Refund'
        '409':
          description: Conflict (idempotency)
  /refunds/{id}:
    get:
      summary: Get a refund
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        '200':
          description: OK
components:
  schemas:
    CreateRefundRequest:
      type: object
      required: [order_id, amount]
      properties:
        order_id: { type: string }
        amount: { type: integer, description: "Minor units (e.g., cents)" }
    Refund:
      type: object
      properties:
        id: { type: string }
        order_id: { type: string }
        amount: { type: integer }
        status: { type: string, enum: [pending, succeeded, failed] }
```

### C) Rate limit & caching headers (HTTP)
```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "v7"
Cache-Control: max-age=60, stale-while-revalidate=120
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 72
X-RateLimit-Reset: 1735689600
```

### D) Idempotent POST with replay‑safe server
```http
POST /refunds
Idempotency-Key: 3bbf9b0c-...

{ "order_id": "o_123", "amount": 1000 }
```

Server stores (`Idempotency-Key` → hash(request) + response) for 24h; a retry with same key returns the first response.

### E) ETag / If‑Match optimistic concurrency
```http
GET /orders/123
ETag: "v7"

PATCH /orders/123
If-Match: "v7"
{ "note": "Ship ASAP" }
```

---

## Glossary (simple definitions)

- **ABAC:** Authorization by attributes/conditions (e.g., “tenant == request.tenant && role == ‘manager’”).  
- **Audit log:** Special log of sensitive actions, protected from tampering.  
- **Blue‑green:** Keep old and new versions; switch traffic once healthy.  
- **Breaker (circuit breaker):** Temporarily stop calling a failing dependency.  
- **Canary:** Release to a small %; watch metrics before full rollout.  
- **Cache (CDN/edge/service/DB):** Temporary storage to speed reads.  
- **Clock skew:** Small time differences between machines.  
- **Consistency (strong/eventual/read‑your‑writes):** How fresh your reads are after writes.  
- **CORS:** Browser rule that controls which sites can call your API from JS.  
- **CSP:** Browser header that limits what scripts/resources can load.  
- **CSRF:** Browser attack where a victim’s browser sends unwanted requests.  
- **DLQ (dead‑letter queue):** Holding area for messages the consumer can’t process.  
- **DLP:** Tools that prevent sensitive data from leaking (e.g., in logs).  
- **ETag:** Version tag for a resource; helps caching and concurrency control.  
- **Fan‑out:** How many downstream calls one request triggers.  
- **Feature flag:** Toggle a feature on/off at runtime.  
- **gRPC/Protobuf:** High‑performance binary RPC framework and schema language.  
- **HSTS:** Header telling browsers to only use HTTPS for your domain.  
- **Idempotency:** Safe to retry the same call without double side‑effects.  
- **ISO‑8601 / RFC 3339:** Standard formats for durations and timestamps.  
- **JWT:** Signed token that asserts identity/claims.  
- **Least privilege:** Give only the permissions needed, no more.  
- **mTLS:** Mutual TLS where both sides present certs to authenticate.  
- **N+1:** Pattern of many small queries instead of one efficient query.  
- **OpenAPI:** Machine‑readable spec for REST APIs.  
- **P50/P95:** Median and 95th‑percentile latency; P95 shows tail behavior.  
- **PII:** Personally identifiable information.  
- **Problem Details:** Standard style for HTTP error bodies.  
- **RBAC:** Authorization by roles (admin, editor, viewer).  
- **RED/USE:** Metric groupings for APIs (RED) and resources (USE).  
- **Replay‑safe:** If the same message/process runs again, it doesn’t cause harm.  
- **RPO/RTO:** Max acceptable data loss (RPO) and time to restore (RTO).  
- **SBOM:** Software Bill of Materials — list of components in a build.  
- **Schema registry:** Place to publish and version your event schemas.  
- **SLO:** Service Level Objective — your reliability target.  
- **Soak test:** Long‑running load test.  
- **SSRF:** Attack that tricks your server to fetch internal resources.  
- **Stale‑while‑revalidate:** Serve cached content while fetching fresh in background.  
- **STRIDE:** Threat modeling mnemonic (Spoofing, Tampering, Repudiation, Info disclosure, DoS, Elevation).  
- **Traceparent:** W3C header carrying the trace ID for distributed tracing.  
- **ULID/UUIDv7:** ID formats that sort by time (useful for paging).  
- **Versioning (API):** Rules for changing an API without breaking clients.

---

## Scorecard (one‑page checklist)

> Copy this block into PRs or design reviews and tick items as you complete them.

```md
### API Definition
- [ ] Problem, users, success metrics, out‑of‑scope
- [ ] Resource model & operations
- [ ] Consistency model per endpoint
- [ ] Idempotency for non‑GET writes
- [ ] Concurrency (ETag/If‑Match)
- [ ] Time semantics (UTC RFC3339, ISO‑8601)
- [ ] Transport chosen & justified
- [ ] Versioning & deprecation policy
- [ ] Pagination (cursor), filtering, sorting
- [ ] Error schema + partial success rules
- [ ] Rate limit contract (429, headers)
- [ ] Standard headers (trace, cache, content type)
- [ ] AuthN/AuthZ model; secret handling
- [ ] Executable spec; quickstart; changelog; mocks/SDKs

### Service Build
- [ ] Data model & access patterns; indexes
- [ ] Migrations (expand → migrate → contract)
- [ ] Identifier format (UUIDv7/ULID); non‑guessable
- [ ] Caching plan; invalidation
- [ ] Stateless services; state stores
- [ ] SLOs & error budgets
- [ ] Timeouts, retries (with jitter), breakers
- [ ] Bulkheads & load shedding
- [ ] Startup readiness; graceful shutdown
- [ ] Chaos/fault injection plan
- [ ] Perf budgets; load/soak tests; N+1 fixed
- [ ] Throughput model; size limits; compression
- [ ] Threat model; TLS/mTLS; SSRF protection
- [ ] Secrets in vault; rotation
- [ ] Data classification; retention & deletion
- [ ] CORS/CSRF (if browser); CSP/HSTS
- [ ] Logs (JSON) with trace IDs; metrics RED/USE; tracing
- [ ] Dashboards, alerts, runbooks; audit logs
- [ ] IaC with reviews & drift detection
- [ ] CI/CD with scans; rollback
- [ ] Canary/blue‑green; feature flags
- [ ] Supply chain (signing, SBOM, scans)
- [ ] Consumer rollout; dual‑read/write if needed
- [ ] Capacity plan; multi‑AZ; autoscaling
- [ ] Cost guardrails; per‑tenant attribution
- [ ] Hot‑spot dashboards & cache hit rate
- [ ] Compliance scope; DSR flows; DLP
- [ ] Access reviews; break‑glass process

### Go‑Live
- [ ] On‑call & runbooks; escalation paths
- [ ] SLO dashboards green with prod‑like traffic
- [ ] Synthetic monitors for critical journeys
- [ ] Error budgets & alerts reviewed
- [ ] Backups/restore drill; RPO/RTO met
- [ ] Incident comms templates; status page playbook
- [ ] Zone failure test passed
- [ ] Legal/privacy docs updated
```
