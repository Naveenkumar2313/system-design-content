# CS-018 — Design a Link Preview Generator

> **Content status:** Educational reference design. This document proposes a generic link preview service and does not claim to describe a named company's private architecture. Workload, latency, and availability values are illustrative assumptions.
>
> **Source alignment:** The assigned case-study catalogue identifies this as Case Study 18. The required structure follows the repository's 18-section case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-018 |
| System name | Link Preview Generator |
| Real-world reference | WhatsApp, Telegram, Slack, Discord, Facebook, LinkedIn, and X link cards are product examples, not asserted architectures |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 18 |
| Difficulty | Intermediate |
| Short description | A secure service that validates submitted URLs, fetches public webpages through an isolated network, extracts card metadata, caches results, and returns previews to messaging and social applications. |
| Primary users | Web/mobile users, messaging platforms, social applications, content publishers, trust and safety operators |
| Key concepts | SSRF prevention, DNS rebinding protection, HTML parsing, Open Graph, Twitter Cards, Redis, queues, workers, negative caching |
| Related component IDs | CMP-001 (Load Balancer) |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-20 |

### Scope and assumptions

- The service accepts a user-supplied HTTP/HTTPS URL and returns title, description, image, favicon, site name, canonical URL, and fetch status.
- The client never fetches external webpages as the security boundary. Fetching occurs on isolated server workers.
- Every URL, resolved DNS answer, redirect target, and image URL is revalidated.
- Private, loopback, link-local, reserved, and cloud-metadata destinations are blocked. Egress network policy and application checks are independent controls.
- Redis provides positive/negative caching and single-flight locking. A database stores preview metadata and status.
- HTML is parsed as text; downloaded markup is never executed in application origin or rendered as active HTML.
- JavaScript-rendered pages are optional and are processed only by a separate sandboxed browser worker.
- All throughput and latency numbers below are teaching assumptions.

---

## 01 — Story

When a user shares `https://example.com/article`, a messaging or social application should display a useful link card instead of only a raw URL. Client-side fetching is unsuitable because browsers have inconsistent network behavior, mobile applications cannot safely share platform credentials, and arbitrary URL fetching creates a server-side trust problem.

The platform submits the URL to a preview service. The service normalizes the URL, applies authentication/rate limits, checks Redis and database records, and either returns a cached preview or creates a bounded fetch job. Workers validate DNS/IP/redirect policy, fetch a limited public response, extract metadata, validate any preview image, and persist the preview for reuse.

### Actors

1. **Client application:** Submits URLs and displays generated link cards.
2. **API Gateway/load balancer:** Terminates TLS, authenticates, routes, and applies limits.
3. **Link Preview Generator:** Normalizes URLs, coordinates cache/database/job work, and returns status.
4. **URL Validation Service:** Parses schemes and hosts and enforces URL/DNS/IP policy.
5. **DNS Resolver:** Resolves hosts for initial and redirected destinations.
6. **URL Fetcher:** Performs bounded HTTP(S) requests in isolated egress workers.
7. **HTML Parser:** Recovers text and metadata from bounded HTML.
8. **Metadata Extractor:** Prioritizes Open Graph, Twitter Card, standard meta, Schema.org, and fallbacks.
9. **Image Processor:** Validates and safely transforms accepted preview images.
10. **Redis:** Stores previews, negative entries, TTL, and single-flight locks.
11. **Database:** Stores preview identity, normalized URL, metadata, status, and lifecycle.
12. **Queue/workers:** Execute asynchronous fetch and refresh jobs.
13. **Monitoring/audit:** Measures latency, cache, origin, queue, worker, and security outcomes.

### Core user journey

1. User enters or shares a URL.
2. Client sends `POST /v1/previews`.
3. Gateway authenticates and applies per-user/domain limits.
4. Preview service validates syntax and normalizes the URL.
5. Redis cache is checked.
6. A cache HIT returns title, description, image, favicon, and canonical URL.
7. A MISS checks the database and creates or obtains a stable preview ID.
8. A short synchronous fetch may run for a fast page; slower pages are queued.
9. Worker validates DNS and IP before connection and repeats checks after redirects.
10. Worker fetches bounded HTML and parser extracts metadata.
11. Preview image URL is independently checked and optionally processed.
12. Database and Redis store a versioned preview or negative result.
13. Client receives the preview or polls status.

### Problem boundaries

The initial design covers preview lookup/generation, safe fetching, extraction, cache, asynchronous processing, and delivery. It excludes general web crawling, search indexing, arbitrary authenticated browsing, unrestricted browser automation, and executing downloaded HTML.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Validate and normalize submitted HTTP/HTTPS URLs | Must |
| FR-02 | Generate preview title, description, image, favicon, site name, canonical URL, and status | Must |
| FR-03 | Prioritize Open Graph, Twitter Card, standard HTML, Schema.org, and fallback metadata | Must |
| FR-04 | Cache successful and negative previews | Must |
| FR-05 | Support synchronous and asynchronous generation | Must |
| FR-06 | Prevent duplicate preview jobs for a normalized URL | Must |
| FR-07 | Follow bounded redirects and revalidate every destination | Must |
| FR-08 | Reject private/internal/cloud-metadata destinations | Must |
| FR-09 | Refresh, delete, and report invalid previews | Should |
| FR-10 | Check preview generation status and completion | Must |
| FR-11 | Sanitize returned text and preview image URLs | Must |
| FR-12 | Emit audit and operational events | Must |

### Non-functional requirements

| ID | Requirement | Initial target |
|---|---|---|
| NFR-01 | Cache lookup latency | Illustrative p95 < 100 ms |
| NFR-02 | Fast synchronous preview | Illustrative p95 < 1.5 s |
| NFR-03 | Asynchronous preview | Illustrative p95 < 30 s |
| NFR-04 | Cache availability | 99.9% for preview lookup |
| NFR-05 | SSRF | No public fetch worker may reach private/reserved destinations |
| NFR-06 | Response limit | Illustrative 2 MiB HTML, 10 MiB image |
| NFR-07 | Redirect limit | Configurable default of five |
| NFR-08 | Scalability | API, Redis, database, queue, and workers scale independently |
| NFR-09 | Isolation | Downloaded HTML is never active application content |
| NFR-10 | Retention | Raw HTML is temporary; preview metadata follows TTL policy |

### Out of scope

Web crawling, search ranking, authenticated page rendering, unrestricted JavaScript execution, screenshot rendering, and private social metadata APIs.

---

## 03 — Scale

### Workload assumptions

| Input | Assumption |
|---|---:|
| Preview requests/day | 5,000,000 |
| Cache hit ratio | 90% |
| Unique preview fetches/day | 500,000 |
| Average HTML size | 500 KiB |
| Fetch timeout | 1.5 s average allowed work |
| Worker baseline | 50 workers × 100 fetch slots |
| Daily external fetch bytes | ≈ 244 GiB before retries |
| Peak-to-average factor | 8× planning assumption |

Average lookup load is approximately `5,000,000 / 86,400 ≈ 58` requests/second. A 20× burst can reach roughly 1,160 lookups/second. A 90% hit ratio leaves about 500,000 external preview misses per day.

Capacity planning must include API lookup, Redis memory, database writes, queue throughput, worker concurrency, DNS queries, response bytes, parser CPU, browser capacity, image processing, and origin fairness. Popular uncached URLs require single-flight and stale-while-revalidate controls.

### Questions for production sizing

- Which TTL and refresh policy applies by domain?
- Which page formats require browser rendering?
- What SLO applies to synchronous versus asynchronous preview?
- How many concurrent fetches are permitted per user, tenant, and domain?
- How long is preview metadata retained?
- What provider DDoS, DNS, queue, Redis, and CDN limits apply?

---

## 04 — Naive Architecture

### Simplest viable design

One application server accepts a URL, performs a basic syntax check, sends an HTTP GET, parses HTML, stores preview fields, and returns JSON.

**Components**
- Client
- API application
- HTTP client
- HTML parser
- Database

**Request flow**
1. Client submits URL.
2. Server fetches HTML.
3. Server extracts title/description/image.
4. Server writes a database row.
5. Server returns preview JSON.

### Naive architecture diagram

See `architecture.png`.

### Why begin here?

This design demonstrates the core path but exposes SSRF, DNS rebinding, redirect loops, memory/time exhaustion, repeated origin fetches, unsafe HTML handling, and application-tier blocking.

---

## 05 — Problems

| Problem | Impact | Improvement |
|---|---|---|
| SSRF | Access to private/cloud services | Validate URL, DNS, IP, redirects, and egress |
| DNS rebinding | Validation and socket destination differ | Pin validated resolution or recheck socket target |
| Redirect loops | Worker/resource waste | Max redirect count and final-target revalidation |
| Slow webpage | API latency and worker exhaustion | Timeout, queue, and isolated worker pools |
| Oversized HTML | Memory and parser abuse | Stream and byte limits |
| Malformed encoding | Incorrect metadata | Robust parser and bounded fallback |
| JavaScript-only page | Missing preview fields | Optional isolated browser worker |
| Cache stampede | Burst origin fetches | Single-flight lock and stale-while-revalidate |
| Malicious HTML | Script injection | Parse text only; never execute remote markup |
| Invalid image | Broken card or SSRF | Validate image URL, type, size, and decoder |
| Domain hot spot | One site exhausts workers | Per-domain concurrency and fairness |
| Duplicate jobs | Repeated external fetch | Stable normalized URL and distributed lock |
| Queue failure | Preview remains pending | Durable queue, retry, reconciliation, DLQ |
| Stale cache | Incorrect card | Versioned records and refresh invalidation |
| Raw HTML retention | Privacy/storage cost | Temporary storage and bounded lifecycle |

---

## 06 — Architecture Evolution

### V1 — Application fetch

A server fetches and parses external HTML. Slow targets and SSRF risks block or threaten the application.

**Change:** isolate fetch/parse work.

### V2 — Cache and database

Redis stores successful and negative previews; database stores durable metadata and status.

**Benefit:** repeated requests avoid origin fetches.

### V3 — Hardened fetcher

The fetcher validates URLs, DNS answers, IP ranges, content type, byte limits, redirects, and read timeouts.

**Benefit:** a private-network destination cannot become an external-fetcher's target.

### V4 — Queue and background workers

Slow preview generation becomes an asynchronous job. Workers retry safely and publish idempotent status updates.

### V5 — Metadata and image processing

The parser prioritizes OG/Twitter/standard/Schema metadata and validates preview images through an isolated decoder.

### V6 — Regional scale and refresh

API/Redis/database/worker tiers scale independently and stale previews can refresh asynchronously.

### Proposed evolved architecture

1. Client calls the API through DNS, gateway, and load balancer.
2. Link Preview Generator normalizes the URL and validates scheme/host/policy.
3. Redis is checked before the database or external fetch.
4. A positive/negative preview or database record is reused.
5. A single-flight lock joins or creates one job.
6. Queue publishes work to an isolated fetch worker.
7. Worker resolves DNS and validates every IPv4/IPv6 result.
8. Worker fetches with redirect, timeout, and byte limits and revalidates each redirect.
9. Parser extracts prioritized metadata.
10. Image processor validates image URL and safely creates a bounded derivative if enabled.
11. Database/Redis receive a versioned preview or negative result.
12. API returns or exposes status for polling.

### Cache HIT

1. Redis key contains normalized URL and policy dimensions.
2. A valid positive or negative record is returned.
3. No external fetch is performed.

### Cache MISS

1. Redis miss checks database for a reusable preview.
2. Preview row becomes `PENDING`.
3. Single-flight lock prevents duplicate job publication.
4. Worker fetches and extracts metadata.
5. Database and Redis receive the final result.

### SSRF protection flow

1. Reject malformed URL, userinfo, unsupported scheme, and unsafe ports.
2. Normalize host and IDNA/punycode representation.
3. Resolve all DNS answers.
4. Reject loopback, RFC1918, link-local, metadata, multicast, reserved, unspecified, private IPv6, and other blocked ranges.
5. Pin or recheck the validated socket destination.
6. Apply redirect count and revalidate `Location` and DNS/IP again.
7. Enforce host allow/block policy and an independent outbound firewall.

### Metadata extraction

| Priority | Source | Example |
|---|---|---|
| 1 | Open Graph | `og:title`, `og:description`, `og:image`, `og:site_name`, `og:url` |
| 2 | Twitter Cards | `twitter:title`, `twitter:description`, `twitter:image`, `twitter:site` |
| 3 | Standard HTML | `<title>`, `meta name=description`, application name |
| 4 | Schema.org | name, headline, description, image |
| 5 | Page fallback | heading, first safe paragraph, canonical, favicon |

Example:

```html
<head>
  <meta property="og:title" content="Example Story">
  <meta property="og:description" content="Preview description">
  <meta property="og:image" content="https://cdn.example/story.jpg">
  <meta property="og:site_name" content="Example News">
  <link rel="canonical" href="https://example.com/story">
</head>
```

The extractor validates each URL and field length, removes control characters, escapes text, and never treats raw HTML as trusted rendering content.

### Link preview card generation

The metadata extractor produces a normalized card model rather than raw HTML:

```json
{
  "preview_id": "prv_01J...",
  "title": "Example Story",
  "description": "Preview description",
  "image_url": "https://cdn.example/story.jpg",
  "favicon_url": "https://example.com/favicon.ico",
  "site_name": "Example News",
  "domain": "example.com",
  "canonical_url": "https://example.com/story",
  "layout": "large",
  "status": "AVAILABLE"
}
```

- **Large card:** Used for a sufficiently large or landscape preview image.
- **Compact card:** Used for a missing image, portrait-only image, or small-screen layout.
- **Text-only card:** Used when no safe image is available or a domain policy prohibits images.
- Title is bounded and ellipsized in the client but stored as normalized text.
- Description is bounded and may fall back to the first safe paragraph or be omitted.
- Domain and favicon are shown consistently even when the page title or site name is missing.
- Canonical URL is used for identity/attribution but not trusted until URL policy accepts it.
- Missing title/image combinations have deterministic defaults to prevent layout instability.
- Responsive card layouts preserve consistent field order while adapting aspect ratio, text length, and image position.
- Image processing validates format and dimensions, removes unsafe metadata, generates bounded thumbnails, and stores derivatives outside the application origin.

### Database model

| Field | Purpose |
|---|---|
| `preview_id` | Stable opaque preview identity |
| `original_url` | User-submitted URL for audit/display |
| `normalized_url` | Unique cache/database key |
| `canonical_url` | Publisher-selected final URL |
| `domain_name` | Domain fairness and throttling |
| `title` / `description` | Extracted text |
| `preview_image_url` | Validated image URL or stored derivative |
| `favicon_url` | Validated favicon |
| `site_name` | Publisher label |
| `fetch_status` | PENDING/FETCHING/AVAILABLE/FAILED/DELETED |
| `http_status_code` | Final public origin status |
| `created_at` / `updated_at` | Lifecycle |
| `expires_at` | Refresh/eviction deadline |

---

## 07 — Components

| Component ID | Component | Responsibility |
|---|---|---|
| CMP-001 | Load Balancer | Distributes preview API traffic |
| TBD | API Gateway | TLS, auth, routing, rate limits |
| TBD | URL Validation | URL/DNS/IP/redirect policy |
| TBD | Preview Service | Cache/DB/job coordination |
| TBD | DNS Resolver | Public DNS resolution |
| TBD | HTTP Fetcher | Bounded external fetch |
| TBD | HTML Parser | Safe text/meta extraction |
| TBD | Metadata Extractor | Source prioritization/fallback |
| TBD | Image Processor | Safe image validation/transformation |
| TBD | Redis | Positive/negative cache and locks |
| TBD | Database | Preview metadata/status |
| TBD | Message Queue | Durable preview jobs |
| TBD | Background Workers | Fetch, parse, process, retry |
| TBD | CDN | Optional derived preview image delivery |
| TBD | Monitoring/Audit | Metrics, logs, traces, security events |

---

## 08 — Data Flow

### Flow A: Request and cache HIT

1. Client sends URL.
2. Gateway validates auth/rate.
3. Preview service normalizes URL.
4. Redis returns positive or negative preview.
5. API returns `200`.

### Flow B: Cache MISS

1. Redis miss checks database.
2. Service creates `PENDING` record or joins existing preview.
3. Single-flight lock prevents duplicate jobs.
4. Job is queued or processed synchronously.
5. Worker safely fetches and parses.
6. Database stores result and Redis caches it.

### Flow C: Asynchronous response

1. API returns `202 Accepted` with preview ID/status URL.
2. Worker updates status and preview version.
3. Client polls or receives notification.
4. A failed preview is cached negatively.

### Flow D: Redirect and SSRF rejection

1. Initial URL and DNS answer pass policy.
2. Worker connects to validated public address.
3. `3xx` response exposes a bounded `Location`.
4. Location is normalized and checked as a new URL.
5. DNS and IP checks repeat.
6. Private/loopback/metadata destination is rejected and audited.

### API design

All control-plane endpoints require authenticated TLS unless the platform explicitly exposes an anonymous preview endpoint. Apply per-user, per-tenant, and per-domain limits.

#### Generate a link preview

`POST /v1/link-previews`

```json
{
  "url": "https://example.com/story",
  "mode": "synchronous",
  "client_hint": {
    "width": 600,
    "layout": "responsive"
  }
}
```

Response for a cached or completed preview (`200 OK`):

```json
{
  "preview_id": "prv_01J...",
  "status": "AVAILABLE",
  "original_url": "https://example.com/story",
  "canonical_url": "https://example.com/story",
  "title": "Example Story",
  "description": "Preview description",
  "image_url": "https://cdn.example/story.jpg",
  "favicon_url": "https://example.com/favicon.ico",
  "site_name": "Example News",
  "domain": "example.com",
  "layout": "large"
}
```

For asynchronous generation, return `202 Accepted`:

```json
{
  "preview_id": "prv_01J...",
  "status": "PENDING",
  "status_url": "/v1/link-previews/prv_01J..."
}
```

#### Retrieve a preview

`GET /v1/link-previews/{preview_id}`

Returns `200 OK` for an available preview or `404 Not Found` for a deleted/hidden preview. The response must not contain raw HTML or internal worker URLs.

#### Check generation status

`GET /v1/link-previews/{preview_id}/status`

```json
{
  "preview_id": "prv_01J...",
  "status": "FETCHING",
  "attempt": 2,
  "updated_at": "2026-11-20T13:00:00Z"
}
```

Possible states include `PENDING`, `FETCHING`, `AVAILABLE`, `FAILED`, `UNSUPPORTED`, `EXPIRED`, and `DELETED`.

#### Refresh or delete

- `POST /v1/link-previews/{preview_id}:refresh` queues a bounded refresh and returns `202 Accepted`.
- `DELETE /v1/link-previews/{preview_id}` marks the preview deleted, evicts cache, and schedules image derivative cleanup.
- `GET /v1/link-previews/{preview_id}/metadata` returns the authoritative extracted fields plus source-priority information.
- `POST /v1/link-previews/{preview_id}:report` accepts a bounded invalid-preview report and creates a moderation/audit event.

API status codes include `400` for malformed URL or payload, `401/403` for auth/policy denial, `404` for missing preview, `409` for conflicting refresh/state, `429` for rate/domain limits, `422` for unsupported fetch policy, and `503` for temporary queue/dependency failure.

---

## 09 — Failure Scenarios

| Failure | Detection | Recovery |
|---|---|---|
| Invalid URL | Syntax/normalizer | `400`, no fetch |
| DNS failure | Resolver timeout/error | Retry bounded resolver; negative cache |
| Website unavailable | Timeout/5xx/connection failure | Bounded retry, stale/negative cache |
| Redirect loop | Redirect counter | Reject, short negative TTL |
| Private redirect | DNS/IP policy | Reject and audit SSRF category |
| Malformed HTML | Parser recovery failure | Partial/fallback preview |
| Missing metadata | Empty fields | Title/heading/canonical/favicon fallback |
| Oversized response | Byte limit | Abort and mark failure |
| Redis failure | Timeout | Database fallback and reduced cache |
| Database failure | Transaction error | Preserve/return PENDING; retry |
| Queue failure | Publish/consume error | Durable retry or DLQ; reconciliation |
| Worker crash | Visibility timeout | Duplicate-safe worker retry |
| Cache stampede | Redis miss burst | Single-flight/stale cache |
| Invalid image | URL/type/decode failure | Remove image, keep text preview |
| Domain hot spot | Per-domain queue health | Fairness/concurrency limits |

---

## 10 — Scaling

- Run stateless API instances across zones behind CMP-001.
- Scale Redis and database reads independently.
- Use distributed single-flight locks for common URLs.
- Scale ordinary HTTP workers separately from browser workers.
- Apply per-domain and per-tenant concurrency.
- Partition database or queue only after measured bottlenecks.
- Deploy cache, queue, and worker replicas regionally where required.
- Use CDN for validated preview-image derivatives.
- Test Redis, queue, database, DNS, egress, and browser failures independently.

Millions of requests should be mostly cache hits. External fetch capacity is governed by worker concurrency, DNS, bandwidth, target response, parser CPU, and image CPU.

---

## 11 — Trade-offs

| Choice | Benefit | Risk |
|---|---|---|
| Synchronous fetch | Immediate result | Slow target blocks lookup |
| Asynchronous fetch | Isolated latency | Polling/status complexity |
| Redis cache | Fast repeated lookup | Eviction/consistency |
| Negative cache | Prevents retry storm | Temporary failures may remain hidden |
| OG-first extraction | Rich metadata | Publisher fields may be misleading |
| Browser rendering | Handles dynamic pages | Cost and larger security boundary |
| Text-only result | Safe UI | Fewer visual/layout details |

Major decisions should record context, alternatives, consequences, and revisit triggers.

---

## 12 — Cost

### Cost categories

API compute, Redis memory, database storage/I/O, queue volume, worker CPU, DNS/egress, browser compute, image processing, CDN storage/egress, logs, and regional redundancy.

### Estimation worksheet

`daily_fetch_bytes = requests × (1 - cache_hit_ratio) × unique_url_ratio × average_HTML_bytes`

`worker_compute = fetches × average_worker_seconds / 3600`

`monthly_cost = fetch/egress + workers + Redis + database + queue/logging + image/CDN`

Use provider pricing calculators and document assumptions. Values are not real vendor quotes.

---

## 13 — Security

### SSRF and URL controls

- Accept only HTTP/HTTPS, reject userinfo and unsupported/suspicious ports.
- Normalize host/IDNA, resolve DNS, and classify all IP addresses.
- Block loopback, private, link-local, metadata, multicast, reserved, and unspecified destinations.
- Revalidate DNS/IP for every redirect and image URL.
- Pin validated resolution where supported or recheck the socket destination.
- Use isolated workers and restricted outbound firewall rules.
- Apply response, redirect, timeout, decompression, parser, and image limits.
- Never execute downloaded HTML or JavaScript in application origin.
- Sanitize text and filenames; escape all UI rendering.
- Redact credentials, secret query parameters, HTML bodies, and sensitive headers from logs.
- Add synthetic tests for localhost, RFC1918, IPv6-private, metadata endpoints, rebinding, redirect escape, and oversized pages.

---

## 14 — Observability

### Metrics

Preview requests RPS, cache hit/miss ratio, lookup latency, fetch latency, parser latency, image latency, queue depth/oldest age, worker processing time, DNS/HTTP status, timeout/rate/redirect counts, SSRF rejection count, response bytes, success/failure, negative cache, and image failures.

### Logs and traces

Propagate request/preview/job IDs. Log normalized URL/domain category, status, latency, size, outcome, worker, and final public status category. Redact secrets, HTML bodies, raw internal targets, and image/authentication data.

### Example alerts

- Queue oldest job age exceeds target.
- Worker CPU/memory or timeout/error rate is elevated.
- Cache hit ratio drops materially.
- SSRF rejection category spikes.
- Oversize/redirect/decompression limits trigger often.
- Redis/database/queue/worker availability fails.

---

## 15 — Simulation

### Learning objective

Demonstrate cache behavior, metadata extraction, SSRF prevention, redirect handling, worker capacity, queue health, negative caching, and preview cost.

### Initial controls

| Key | Default | Range |
|---|---:|---|
| preview_requests_per_second | 100 | 1–100000 |
| unique_url_ratio | 0.10 | 0–1 |
| cache_hit_ratio | 0.90 | 0–1 |
| html_size_kib | 500 | 1–2048 |
| fetch_timeout_ms | 1500 | 50–30000 |
| max_redirects | 5 | 0–20 |
| worker_count | 50 | 1–1000 |
| concurrency_per_worker | 100 | 1–500 |
| ssrf_probability | 0.01 | 0–1 |
| origin_failure_probability | 0.05 | 0–1 |
| negative_ttl_seconds | 300 | 30–86400 |
| positive_ttl_seconds | 86400 | 60–2592000 |
| browser_render_ratio | 0.05 | 0–1 |

### Required scenarios

1. Cache HIT versus MISS.
2. Localhost/private/link-local/metadata URL attempts.
3. DNS rebinding and redirect escape.
4. OG/Twitter/standard/Schema/fallback metadata.
5. Malformed HTML and missing metadata.
6. Cache stampede.
7. Worker saturation.
8. Redis/DB/queue failure.
9. Duplicate or out-of-order worker completion.
10. Oversized HTML/compression/image.
11. Per-domain hot spot.
12. Preview refresh/invalidation.

### Acceptance checks

- Private destinations are rejected before connection.
- Every redirect is revalidated.
- One normalized URL creates one logical in-flight fetch.
- Cache HIT does not contact origin.
- Remote HTML is never executed.
- Preview text is escaped and bounded.
- Status survives queue/worker failure.
- Negative cache prevents immediate retry storms.

---

## 16 — Design Challenge

Design a safe global link preview service for messaging/social platforms.

**Requirements**
- Normalize and cache URLs.
- Extract OG, Twitter Card, standard meta, Schema.org, canonical, favicon, and fallback fields.
- Generate previews synchronously or asynchronously.
- Revalidate every DNS answer and redirect.
- Block SSRF/private/cloud-metadata targets.
- Bound HTML, redirects, timeout, parser, image, and browser work.
- Use Redis, database, queue, workers, refresh, delete, status, and DLQ.
- Scale API/cache/fetch/parser/image tiers independently.
- Monitor cache, queue, fetch, worker, SSRF, and generation health.

**Learner deliverables**
1. Requirements and workload.
2. Naive/evolved diagrams.
3. Cache HIT/MISS and SSRF flows.
4. REST and JSON examples.
5. Redis/database data model.
6. Metadata extraction rules.
7. Failure/retry/DLQ design.
8. Security threat model.
9. Cost and scaling worksheet.
10. Simulation plan.

---

## 17 — AI Interview

| ID | Difficulty | Question | Strong answer |
|---|---|---|---|
| Q01 | Beginner | What is a link preview service? | Server-side fetch, parse, cache, return metadata |
| Q02 | Beginner | Explain cache HIT and MISS. | Redis key, DB fallback, job, result |
| Q03 | Intermediate | How does SSRF happen? | User URL, redirect, DNS/IP, private target |
| Q04 | Intermediate | How does DNS rebinding evade checks? | DNS answer changes before socket connection |
| Q05 | Intermediate | Why revalidate redirects? | Redirect destination can move to private network |
| Q06 | Intermediate | Compare OG, Twitter, and HTML meta. | Priority and fallback behavior |
| Q07 | Intermediate | How is cache stampede prevented? | Single-flight lock, stale serving, negative cache |
| Q08 | Advanced | How do queue retries remain idempotent? | Stable preview/version and conditional update |
| Q09 | Advanced | How are preview jobs partitioned? | Lightweight HTTP, parser, browser, image pools |
| Q10 | Advanced | How is downloaded HTML made safe? | Text parsing, escaping, no active rendering |
| Q11 | Advanced | How does Redis failure affect consistency? | DB fallback, bounded stale behavior |
| Q12 | Advanced | How are preview images secured? | Independent URL/IP/type/size/decode checks |
| Q13 | Advanced | How are millions of requests scaled? | Cache-first lookup and independent workers |
| Q14 | Advanced | How are negative results designed? | Failure classes and bounded TTL |
| Q15 | Advanced | How is sensitive preview data removed? | DB delete, cache eviction, image purge |

---

## 18 — Architecture Score

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements and workload | 10 | Scale, latency, cache, retention |
| Architecture and data flow | 15 | API, Redis, DB, queue, worker, parser |
| Metadata extraction | 15 | OG, Twitter, meta, Schema, fallback |
| SSRF and fetch security | 20 | DNS/IP/redirect/egress/limits |
| Scaling and availability | 15 | Independent tiers, regional recovery |
| Cache/worker consistency | 10 | Stampede, retries, negative cache |
| Cost and trade-offs | 5 | Fetch, worker, Redis, DB, CDN |
| Observability | 10 | Metrics, logs, traces, alerts |
| **Total** | **100** | |

### Scoring guidance

- 0: missing or incorrect
- 1: mentioned without explanation
- 2: partially explained
- 3: correct and reasonably justified
- 4: detailed, measurable, and handles edge cases

Weighted score = sum of (dimension score / 4 × dimension weight).

