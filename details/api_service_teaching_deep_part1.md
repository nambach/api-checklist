
# API & Service Build — Teaching Edition (Deep Dive) — **Part 1**
**Entries 1–15: Scope & Semantics, Protocol & Shape**

> This part explains the foundational concepts and interface design. Each entry includes an in‑depth paragraph and a concrete example.

---

## 1) Problem Statement, Users, Success Metrics, Out‑of‑Scope
A strong API or service begins with a crisp articulation of *who needs what and why*, because every technical decision flows from that narrative. Write a single paragraph that names the user(s), their job‑to‑be‑done, the pain they experience today, and what “better” looks like in measurable terms. Convert that into **success metrics** (e.g., adoption counts, P95 latency, error rate, cost per call) and record **non‑goals** so the team knows what not to build right now. This creates alignment with product and stakeholders, protects the timeline from scope creep, and gives you a yardstick to decide trade‑offs (like correctness vs. speed) as you implement.
**Example:**  
```
Problem: Support teams need to automate refunds to reduce ticket time.  
Users: Internal support bot; partner apps.  
Success in 60 days: 200 daily API users, P95 < 400 ms, refund failure rate < 0.5%.  
Out of scope: Cross‑currency refunds; multi‑capture reversals.
```

## 2) Resource Model (nouns) & Operations (verbs)
Design your API around **resources (nouns)** representing real‑world objects and a small set of **operations (verbs)** that match user actions. The resource model becomes the mental map clients use; if it mirrors their domain concepts, the API will feel intuitive. Resist one‑off RPC‑style verbs; instead, encode actions as resource creations or state transitions. Keep the surface area small and consistent so documentation and SDKs remain approachable. Align data storage and indexes with the read/write patterns implied by your resources and queries.  
**Example:**  
Resources: `order`, `refund`. Operations: `POST /refunds` (create), `GET /refunds/{id}` (retrieve), `GET /orders/{id}/refunds` (list by parent), `POST /refunds/{id}:cancel` (state transition).

## 3) Consistency Model (strong, read‑your‑writes, eventual)
Consistency says how fresh a read is after a write. **Strong consistency** guarantees that a read immediately reflects the latest write, which simplifies clients but often requires single‑leader coordination and can limit scale during partitions. **Read‑your‑writes** ensures a caller sees its own changes even if others may not yet, typically via session stickiness or version tokens. **Eventual consistency** favors availability and performance by allowing replicas and caches to lag for a bounded time; clients must tolerate short‑term staleness and sometimes reconcile. State the guarantee per endpoint and, when not strong, publish maximum lag and give clients hints (ETag/version) to reason about freshness.  
**Example:** `GET /refunds/{id}` is strong; `GET /orders/{id}/refunds` may lag up to 2s while indices update. Responses include `version: 7` so clients can detect stale reads.

## 4) Idempotency (safe retries for non‑GET writes)
**Idempotency** means repeating the same request doesn’t multiply side‑effects—crucial when networks drop connections and clients retry. Require an `Idempotency-Key` (a UUID) for non‑GET writes; store a short‑lived record mapping that key to the original request hash and canonical response. On a retry with the same key and same body, return the first response; if the body differs, reject with `409 Conflict`. Choose the deduplication window based on worst‑case client retry behavior. This protects against double charges, duplicate tickets, or multiple emails on flaky connections.  
**Example:**  
```
POST /refunds
Idempotency-Key: 94e2...

{ "order_id": "o_123", "amount": 1000 }
```
Server returns 201 once; subsequent identical retries return the same 201 body, not a second refund.

## 5) Concurrency Control (optimistic via ETag/If‑Match)
When two writers update the same record, **lost updates** can silently overwrite data. With **optimistic concurrency**, each resource carries a version (e.g., integer or ETag). Reads return the version; writes include `If-Match: <version>`. The server applies the change only if the version matches the current one; otherwise it returns `412 Precondition Failed`, prompting the client to re‑read and merge. This avoids global locks, scales well when contention is low, and makes conflicts explicit. Document how clients should recover (e.g., re‑fetch, merge, retry).  
**Example:**  
```
GET /orders/123  →  ETag: "v7"
PATCH /orders/123  If-Match: "v7"  { "note": "ship ASAP" }  →  200
```
If another client updated to `"v8"` first, your PATCH would return 412 instead.

## 6) Time Semantics (UTC, RFC3339, ISO‑8601, skew)
Time is fraught with pitfalls: time zones, daylight saving changes, leap seconds, and client clock drift. Normalize all timestamps to **UTC** and encode them in **RFC3339** (e.g., `2025-03-15T14:57:00Z`) so they sort lexicographically and parse consistently. Represent durations and periods using **ISO‑8601** (e.g., `PT30S` for thirty seconds, `P1D` for one day). Make servers authoritative for created/updated timestamps and allow reasonable clock **skew** (e.g., ±5 minutes) in validations like token expiry or request signing. Document any time windows or retention periods clearly so clients don’t guess.  
**Example:** API returns `created_at: "2025-05-01T09:20:33Z"` and accepts `window: "PT5M"` for a five‑minute grace period.

## 7) Transport Choice (REST/JSON, gRPC/Protobuf, GraphQL, Events)
The protocol you choose shapes both performance and developer experience. **REST/JSON** is human‑readable, benefits from HTTP caches and proxies, and has broad client support—excellent for public APIs. **gRPC/Protobuf** gives you strict schemas, compact messages, and HTTP/2 streams—ideal for low‑latency, service‑to‑service calls. **GraphQL** lets clients request exactly the fields they need but requires defenses against expensive queries (depth/complexity limits, server‑side caching). **Event streams** decouple producers from consumers and shine for notifications or pipelines, but delivery semantics and idempotent handlers become central. Pick based on the call patterns, latency targets, and audience, and write down why.  
**Example:** External partner integration: REST/JSON; internal high‑QPS microservice calls: gRPC; UI screen with varied field needs: GraphQL query; cross‑system updates: events to Kafka.

## 8) Versioning Policy (additive changes, deprecations, majors)
APIs evolve; your policy prevents breaking client apps. In a given **major** version (e.g., `v1`), allow only **additive** changes: add optional fields and new endpoints; never remove, rename, or change semantics. For breaking changes, release a new **major** (`v2`), provide a migration guide, and publish a **deprecation window** (e.g., 180 days) before retiring the old version. Automate breaking‑change detection in CI (OpenAPI or Protobuf linters) and maintain a human‑readable changelog. Communicate changes via email/webhooks so consumers aren’t surprised.  
**Example:** Adding `reason` to refund objects is additive in `v1`. Changing `amount` from cents to dollars requires `v2` with clear migration steps.

## 9) Pagination with Cursor & Limit Caps
Large lists can overwhelm servers and clients; **cursor pagination** returns small chunks and an opaque `next_page_token` to fetch the next chunk. Sort by a **stable key** (commonly `(created_at, id)`), encode your last‑seen tuple in the cursor, and include a server‑enforced **limit cap** so `?limit=5000` doesn’t melt your database. This yields consistent paging even during concurrent inserts and avoids the slowness and duplication of `OFFSET/LIMIT` on big tables. Document defaults (e.g., `limit=50`) and the maximum (e.g., `limit<=100`).  
**Example:**  
```
GET /refunds?limit=100&page_token=eyJjcmVhdGVkX2F0IjoiMjAyNS0wNS0wMVQwOToyMDozM1oiLCJpZCI6InJfMTAwIn0=
→  { "items": [...], "next_page_token": "..." }
```

## 10) Filtering & Sorting (bounded and index‑backed)
Filtering narrows results server‑side; sorting orders them predictably. To keep queries safe, **whitelist** the fields and operators you support (e.g., `status=`, `created_after=`) and ensure there are **indexes** to serve common filters quickly. Cap the number of predicates, guard against unbounded `LIKE %term%` scans, and always combine filters with paging to avoid huge result sets. Publish the default sort and how to request ascending/descending.  
**Example:** `GET /orders?status=paid&created_after=2025-01-01&sort=-created_at` with an index on `(status, created_at)` for speed.

## 11) Error Model (one typed schema)
Consistency in error responses dramatically simplifies client code. Define a single JSON schema with a **stable code** (machine‑readable), human message, optional details, and a `request_id` for correlation. Map exceptions to this taxonomy uniformly across services so logs, dashboards, and support tooling work the same way everywhere. Distinguish clearly between **4xx** (client mistakes; don’t retry without changes) and **5xx** (server/transient; may be retried) and document whether an error is retryable.  
**Example:**  
```json
{
  "type": "https://errors.example.com/validation",
  "code": "amount_too_small",
  "message": "Minimum is 1.00",
  "details": { "field": "amount", "min": 100 },
  "request_id": "req_47f8"
}
```

## 12) Partial Success for Batch Operations
Batch endpoints improve efficiency but should allow **partial success**, returning per‑item results so clients can retry only the failures. Treat each item atomically and include its index or ID in the response. Pick a response code that reflects the overall outcome (200 or 207 for mixed results) and provide crisp retry guidance. This avoids punishing clients for large batches and helps them converge quickly under transient errors.  
**Example:**  
```json
{
  "successes": [{"index": 0, "id": "r_1"}],
  "errors":    [{"index": 1, "code": "invalid_amount"}]
}
```

## 13) Rate Limiting & Quotas (fairness and protection)
To keep your system healthy and fair, enforce per‑principal request budgets. Common algorithms include **token bucket** (supports bursts up to capacity while sustaining a fill rate) and **sliding window** counters for simpler accuracy. Return `429 Too Many Requests` with headers like `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` so clients back off gracefully. Provide differentiated limits for internal apps, partners, and anonymous users; monitor cap hits to decide when to offer bulk endpoints.  
**Example:**  
```
HTTP/1.1 429 Too Many Requests
Retry-After: 12
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1735689600
```

## 14) Standard Headers (tracing, idempotency, caching, content type)
Standardizing headers unlocks observability and performance. Propagate `traceparent` (W3C) across services so one request can be followed end‑to‑end in logs and traces. Require `Idempotency-Key` on non‑GET writes to support safe retries. Use `ETag`/`Last-Modified` and `Cache-Control` to enable conditional requests and revalidation. Set `Content-Type` precisely (`application/json; charset=utf-8`) and reject mismatches to avoid silent parse errors. Document which headers are accepted/returned so SDKs and proxies don’t strip them inadvertently.  
**Example:**  
```
GET /refunds/123
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Accept: application/json
```

## 15) Localization (codes over prose)
APIs are machine interfaces. Prefer **stable codes** (enums or error codes) over translatable prose; let user interfaces choose language and formatting. If you must return human‑readable text, treat it as best‑effort and never require clients to parse it. Keep numeric and money fields as numbers (e.g., cents as integers) and dates in standard formats; UIs can localize for display. This keeps client logic stable across locales and avoids brittle scraping of messages.  
**Example:** Instead of `"message": "Le montant est trop petit"`, return `"code": "amount_too_small"` and let the UI render French text.
