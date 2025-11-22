
# API & Service Build — Teaching Edition (Deep Dive) — **Part 4**
**Entries 39–49: Security & Privacy, Observability**

---

## 39) Threat Model (STRIDE) & Data Flow Diagrams
A threat model systematically asks: what could go wrong? Draw a **data flow diagram** of your system (clients, services, stores, third‑party APIs) and for each trust boundary, enumerate **STRIDE** threats—Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege. For each, list mitigations (authN/mTLS, signatures, audit logs, encryption, rate limits, least privilege). Revisit the model when architecture or scope changes. This exercise prevents blind spots and converts vague “security” into concrete controls with owners.  
**Example:** For webhooks, threats include spoofing and tampering; mitigations: HMAC signatures with timestamp tolerance, mTLS to known hosts, replay protection via nonces.

## 40) TLS Everywhere; mTLS for Service‑to‑Service
Transport Layer Security protects data in transit and resists man‑in‑the‑middle attacks. Enforce HTTPS on all external endpoints, enable HSTS so browsers always use TLS, and prefer modern cipher suites. For **service‑to‑service** traffic, use **mutual TLS** so both sides authenticate; rotate short‑lived certificates automatically and verify SANs to prevent impostors. Terminate TLS at trusted edges only; if you break and re‑encrypt, treat internal links with the same care in zero‑trust environments.  
**Example:** A service mesh provisions mTLS certs for each pod and rotates them daily; policies require TLS from ingress to backend, with certificate pinning for outbound calls.

## 41) Input Validation & SSRF Protection
Treat all inputs as hostile until proven otherwise. Validate and canonicalize parameters against schemas; reject unexpected fields and overly long values early. For outputs (HTML, SQL), use proper encoding or parameterization. **SSRF** occurs when user input causes your server to fetch arbitrary URLs; defend with an egress allow‑list, DNS pinning, blocked private ranges (169.254.169.254, 10.0.0.0/8), and no raw redirects. Log validation failures without leaking data.  
**Example:** An image fetcher only allows `https://img.example-cdn.com/*` and refuses IP literals or private subnets; HTTP client disables redirects and strips response metadata from errors.

## 42) Secrets Management (vault, rotation, least privilege)
Centralize secret storage in a **vault/KMS**; never hard‑code or commit secrets. Grant each service the minimum privileges it needs (least privilege), prefer short‑lived tokens or dynamic credentials, and automate rotation. Encrypt data at rest using managed keys and envelope encryption for field‑level protection. Monitor for secret exposure (e.g., in logs, repos) and revoke quickly when detected.  
**Example:** The payments service uses a dynamic DB credential with a 1‑hour TTL issued by the platform; its IAM role allows only that DB and a specific S3 bucket for receipts.

## 43) Data Classification, Retention & Deletion
Not all data is equal. Classify fields and datasets (public, internal, confidential, regulated) and apply controls accordingly (encryption, access limits, audit requirements). Define retention periods so you don’t keep personal data longer than needed; implement deletion workflows that propagate to caches, search indexes, and backups. Verify deletion with automated tests and periodic drills, and document legal holds that pause deletion when required.  
**Example:** PII like email and phone are marked “confidential” with 2‑year retention; user deletion triggers removal from primary DB, caches, and search, and enqueues a backup‑prune job.

## 44) Browser Protections (CORS, CSRF, Security Headers)
If your API is called from browsers, configure **CORS** to a tight allow‑list of origins and avoid `*` with credentials. For cookie‑based auth, add **CSRF** tokens to state‑changing requests and set cookies with `SameSite=Lax/Strict`, `HttpOnly`, and `Secure`. Use **CSP** to restrict script sources and **HSTS** to enforce HTTPS. These guard against cross‑site request forgery, injection, and downgrade attacks that target browser contexts.  
**Example:** `Access-Control-Allow-Origin: https://app.example.com` and `Vary: Origin` on responses; mutation endpoints require a CSRF header validated against a session secret.

## 45) Structured Logs & Correlation (trace/log IDs)
Logs are far more useful when structured (JSON) with consistent fields: timestamp, level, service, endpoint, `request_id`, and `trace_id` that matches your distributed tracing. This allows fast filtering, aggregation, and linkage to metrics and traces. Redact sensitive fields at the source and apply sampling to high‑volume debug logs so storage stays affordable. Define retention to meet compliance without hoarding.  
**Example:** Each request logs `{ "trace_id": "...", "request_id": "...", "route": "/refunds", "status": 200, "duration_ms": 132 }` and links to a trace with matching ID.

## 46) Metrics (RED for APIs, USE for resources)
Metrics are your heartbeat. The **RED** method tracks **Rate** (requests/sec), **Errors** (non‑2xx), and **Duration** (latency) per endpoint so you know if users are successful and how fast. The **USE** method tracks **Utilization**, **Saturation**, and **Errors** for infrastructure (CPU, memory, queues, DB). Combine both to get a 360° view and alert on user‑impacting conditions, not just server internals. Publish histograms for latency and expose exemplars linked to traces for rapid forensics.  
**Example:** A dashboard shows `GET /refunds` rate, error %, and latency percentiles alongside DB CPU, connection saturation, and queue depth.

## 47) Distributed Tracing (follow a request end‑to‑end)
Tracing shows where time is spent across services. Propagate W3C `traceparent` so each hop adds a **span** with start/stop times and tags (endpoint, db.statement, tenant). Sample enough traffic to capture rare slow paths, and link traces to logs and metrics for context. Inspect traces during incidents to find which dependency or query dominates latency and to validate that retries and breakers behave as expected.  
**Example:** A slow trace reveals 80% of latency comes from a `searchIndex.query` span; caching that result drops P95 by half.

## 48) Dashboards, Alerts & Runbooks (actionable operations)
Dashboards present the golden signals for user journeys; alerts fire only when human action is needed; runbooks tell on‑call engineers exactly what to do. Tie alerts to SLO burn rates or thresholds that reflect user pain, include context (service, region, recent deploys), and reference a runbook with decision trees and commands. Review noisy alerts regularly and retire those that never lead to action. This discipline shortens incidents and preserves focus.  
**Example:** “P95 latency of `POST /refunds` > 400 ms for 10 min and error rate > 0.5%” alerts the owner, includes links to traces and a runbook section “payment‑gateway slowness.”

## 49) Audit Logs (tamper‑evident records of sensitive actions)
For security‑significant actions (login, role changes, payouts, data exports), keep **audit logs** that are append‑only, tamper‑evident, and access‑controlled. Include who did what, to which resource, when, from where, and whether it succeeded. Store separately from application logs, consider immutability (WORM buckets), and review regularly. Audit trails are invaluable for investigations and compliance, but only if integrity and coverage are maintained.  
**Example:** Writing to an append‑only store: `{ "actor":"u_9", "action":"refund.approve", "target":"r_42", "result":"success", "ip":"203.0.113.9", "time":"2025-06-01T12:04:22Z" }`.
