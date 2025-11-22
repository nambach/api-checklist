
# API & Service Build — Teaching Edition (Deep Dive) — **Part 2**
**Entries 16–27: Auth, Documentation/DX, Architecture & Data**

---

## 16) Authentication (AuthN)
Authentication proves **who** is calling your API. For user‑facing flows, OAuth2/OIDC is the industry baseline: clients obtain tokens (often JWTs) from a trusted identity provider and present them with each request. For service‑to‑service calls, prefer short‑lived **mTLS** certificates or signed JWTs issued by your platform. Validate tokens rigorously—issuer, audience, expiry, signature—and allow small clock skew. Rotate signing keys via JWKS and keep token lifetimes short to limit blast radius if leaked. Document accepted grant types, scopes, and token formats so integrators don’t guess.  
**Example:** A CLI obtains a token via OAuth2 client credentials and calls `GET /orders` with `Authorization: Bearer <JWT>`; your gateway validates signature and `aud` before routing.

## 17) Authorization (AuthZ)
Authorization governs **what** an authenticated caller may do. **RBAC** assigns roles (admin, editor, viewer) and is simple to reason about, while **ABAC** uses attributes/conditions (tenant, ownership, time) for finer control. Centralize your policy evaluation so checks are visible and auditable, and default to deny‑by‑default with explicit allow rules. Enforce tenant scoping in the data layer (e.g., always filter by `tenant_id`) to prevent cross‑tenant access even if higher layers err. Record decisions for sensitive actions in audit logs.  
**Example:** A policy allows `refund:read` when `request.tenant_id == resource.tenant_id` and subject has role `support` or attribute `department=finance`.

## 18) Secret Handling & Logging
Secrets (tokens, API keys, passwords) must never appear in URLs or logs. Pass secrets in headers or bodies, store them in a **vault/KMS**, and rotate them automatically. Adopt **structured logging** and redact sensitive fields at the logger (e.g., mask `Authorization`, `Set-Cookie`, email, phone). Make logs tamper‑evident and set least‑privilege access to log storage. Avoid dumping full request/response bodies in production; sample or scrub them with allow‑lists.  
**Example:** NGINX is configured to drop query strings from access logs and your app’s logger replaces credit‑card digits with `****` before writing JSON logs.

## 19) Executable Spec (OpenAPI/Proto)
A machine‑readable spec is the **source of truth** for your API, enabling documentation, SDK generation, mocks, and contract tests. Treat the spec like code: keep it in the repo, review via pull requests, and validate in CI for breaking changes. Include examples and your error catalog so generated docs and clients are useful by default. Keeping the spec authoritative prevents drift between implementation and docs and accelerates onboarding.  
**Example:** A GitHub action runs `openapi-diff` on pull requests and fails the build if any response field is removed or renamed in `v1` endpoints.

## 20) Quickstart (developer path to first success)
A great API minimizes the “time to first successful call.” Write a **Quickstart** that shows how to obtain credentials, make a single successful request with `curl`, and perform one end‑to‑end flow with an SDK. Keep secrets copy‑pastable but short‑lived, and provide a sandbox environment. Test this doc monthly with a fresh account to ensure it stays honest. Good Quickstarts cut support load and raise adoption because developers can *feel* success quickly.  
**Example:** A single page gets developers from sign‑up → token → `curl POST /refunds` → view result in dashboard in under five minutes.

## 21) Changelog & Deprecations
Change is inevitable; surprise is optional. A **changelog** lists what changed, when, and why, with migration guidance and links to docs. For breaking changes, announce well in advance, send reminders, and provide compatibility shims where possible. Keep old and new versions live for a planned overlap, and instrument usage so you can nudge lagging consumers. This builds trust and lets partners plan upgrades.  
**Example:** “2025‑03‑01: Added field `reason` to `Refund`. 2025‑06‑01: `status` enum adds `reversed` (non‑breaking). 2025‑10‑01: `/v1beta/refunds` sunset; use `/v1/refunds`.”

## 22) Mocks & SDKs
Mocks and SDKs turn your spec into developer velocity. A **mock server** (or Postman collection) lets integrators prototype without credentials or hitting production data, while SDKs hide boilerplate (auth, retries, pagination) and enforce typed models. Keep mocks realistic—include latency, idempotency, and error codes—so integrations don’t break on first contact with reality. Ship SDKs for your top languages, version them with the API, and test them in CI with contract tests.  
**Example:** You publish a Node/TypeScript SDK with `createRefund({ orderId, amount })` that automatically sets `Idempotency-Key` and unwraps partial‑success results.

## 23) Data Model Review (keys, indexes, access patterns)
Start from **access patterns**, not tables: list the queries you must support, then design keys and indexes to serve them efficiently and safely. For each list call, ensure an index supports the filter/sort combo; for each read, ensure uniqueness and referential integrity are clear. Consider cardinality (how many of X per Y), nullability, and how updates will occur. Write sample queries and verify query plans early to avoid late surprises.  
**Example:** If you need `GET /orders?customer_id=&created_after=...`, define a composite index on `(customer_id, created_at)` and ensure `created_at` has high cardinality.

## 24) Migrations (expand → migrate → contract)
Schema and behavior change safely through a three‑phase rollout. **Expand:** add new columns/tables and write code that can handle both old and new shapes; backfill data asynchronously. **Migrate:** dual‑write and dual‑read, preferring new fields but tolerating old ones. **Contract:** once confident, remove old code paths and columns. This avoids long locks, downtime, and data loss. Always script backfills, track progress, and have a rollback plan.  
**Example:** Adding `external_reference` to `orders`: expand by adding the column; migrate by writing to both `ref` and `external_reference`; contract by deleting `ref` after a week of clean metrics.

## 25) Identifiers (UUIDv7/ULID, non‑guessable)
Identifiers should be unique and, when helpful, **time‑sortable**. Formats like **UUIDv7** or **ULID** embed time so new items cluster in storage and sort naturally, which helps pagination and index locality. Never expose sequential, guessable IDs for sensitive objects; prefer opaque IDs that reveal nothing about volume or ordering. When human‑readable slugs are useful, treat them as aliases, not primary keys.  
**Example:** `r_01HZZK3TTZ2W4W2N9V6E7V3R6P` (ULID) used as the refund ID visible in URLs and logs; underlying DB uses the same value as primary key.

## 26) Caching Strategy (what/where/TTL/invalidation)
Caching reduces latency and cost by serving repeated reads from faster storage layers, but only when designed deliberately. Decide **what** to cache (stable reads, derived data), **where** (CDN, edge, service memory, DB cache), **for how long** (TTL), and **how to invalidate** (ETag revalidation, explicit purge, key versioning). Protect against cache stampedes by adding jitter to TTLs or locking around recomputation (“single‑flight”). Document keys and consistency expectations so clients know when data may be slightly stale.  
**Example:** Product details cached at CDN for 60s with `Cache-Control: max-age=60, stale-while-revalidate=120`; edge revalidates via `If-None-Match` against the origin.

## 27) State Boundaries (stateless services; where state lives)
Stateless services scale and recover easily because any instance can handle any request. Persist **state**—sessions, jobs, files—in purpose‑built stores (databases, Redis, object storage) rather than memory or local disk. If you need sticky sessions, question why; they complicate autoscaling and failover. For background processors, design idempotent handlers and store checkpoints so work can resume after restarts. Draw a diagram labeling which components are stateless and where each kind of state lives.  
**Example:** Web API pods are stateless; session data lives in Redis; file uploads go to object storage; background workers process a durable queue with retry‑safe handlers.
