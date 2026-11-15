# CS-Pastebin — Design a Pastebin

> **Content status:** Educational reference design. This document is a proposed design for a Pastebin-like text-sharing service. It does not claim to describe Pastebin's private production architecture.
>
> **Source alignment:** This adaptation uses Category A: Web & Internet Systems and the supplied URL-shortener template; the Pastebin catalogue number is not verified. The required structure follows the 18-component case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-Pastebin |
| System name | Pastebin |
| Real-world reference | Pastebin (reference product, not an asserted architecture source) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | To be verified |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | A service that accepts text, generates a unique paste ID, stores the content and metadata, and returns a shareable URL for retrieval. |
| Primary users | Paste creators, paste readers, administrators, analytics consumers |
| Key concepts | HTTP, DNS, API design, unique ID generation, relational/key-value storage, caching, reads, rate limiting, analytics, observability |
| Related component IDs | CMP-001 (Load Balancer); additional components can be added to the component index as they are documented |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-15 |

### Scope and assumptions

- Users can create a paste from non-empty text within a defined size limit.
- The service returns a paste URL using a domain controlled by the service, such as `paste.example`.
- Visiting the paste URL returns the stored text.
- Pastes may optionally have a custom paste ID, expiration time, and owner. Public, unlisted, and private visibility are supported; private pastes require authorization. Unlisted URLs are not access controls.
- Basic view analytics are included as an extension.
- The initial design is a single-region educational deployment; global distribution is an evolution stage.
- Traffic figures below are illustrative planning assumptions, not measured Pastebin figures.

---

## 01 — Story

People frequently share code snippets, logs, notes, and configuration examples through messages and documents. Long text can be difficult to format, copy, and share reliably.

A Pastebin service stores a text snippet under a unique identifier. A creator submits text and receives a shareable URL. A reader opens that URL; the service retrieves and returns the stored text.

### Actors

1. **Paste creator:** Submits a text snippet and manages pastes.
2. **Paste reader:** Opens a paste URL.
3. **Analytics consumer:** Views paste counts and aggregated activity.
4. **Administrator:** Handles abuse reports, policy enforcement, and service operations.

### Core user journey

1. Creator submits `Hello, system design!`.
2. Service validates the text.
3. Service generates a unique paste ID, for example `aB91xQ`.
4. Service stores `aB91xQ → Hello, system design!`.
5. Service returns `https://paste.example/aB91xQ`.
6. Visitor requests `https://paste.example/aB91xQ`.
7. Service resolves the paste ID and returns the stored text.

### Problem boundaries

This case study focuses on paste creation, paste ID resolution, reads, basic paste management, and view-event collection. It does not attempt to reproduce every commercial feature of Pastebin.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Accept valid paste content and create a paste | Must |
| FR-02 | Generate a unique paste ID | Must |
| FR-03 | Return stored text to an authorized reader | Must |
| FR-04 | Reject malformed or unsupported paste contents | Must |
| FR-05 | Prevent accidental paste ID collisions | Must |
| FR-06 | Allow a user to request a custom paste ID, subject to availability | Should |
| FR-07 | Allow an owner to view and deactivate their pastes | Should |
| FR-08 | Support optional expiration timestamps | Should |
| FR-09 | Record view events for analytics | Should |
| FR-10 | Provide aggregated view statistics | Could |
| FR-11 | Detect and restrict abusive or malicious pastes | Should |

### Non-functional requirements

| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Read latency | Target p95 under 100 ms inside the service region, excluding visitor-to-service network time |
| NFR-02 | Availability | Illustrative target 99.9% monthly for read service |
| NFR-03 | Durability | A successfully created paste record should not be lost after an ordinary process restart |
| NFR-04 | Consistency | A newly created paste ID should resolve after successful creation; cache propagation must not expose stale paste records |
| NFR-05 | Scalability | Read traffic should scale independently from paste creation |
| NFR-06 | Security | Validate text, protect management APIs, rate-limit creation, and prevent stored XSS |
| NFR-07 | Observability | Track request rate, latency, errors, cache hit ratio, and dependency health |
| NFR-08 | Maintainability | Keep paste management, read, and analytics responsibilities separable |

### Out of scope for the initial version

- Full enterprise account management and billing.
- Advanced syntax highlighting and collaborative editing.
- Multi-region active-active writes.
- Guaranteed real-time analytics.
- Browser extensions and mobile applications.

---

## 03 — Scale

All values in this section are **illustrative workload assumptions for teaching and simulation**. They are not Pastebin traffic disclosures.

### Workload assumptions

| Metric | Illustrative value | Reasoning |
|---|---:|---|
| Registered users | 100,000 | Small-to-medium learning scenario |
| Pastes created per day | 100,000 | Write workload |
| Reads per day | 10,000,000 | Read-heavy workload |
| Average read RPS | ~116 | 10,000,000 / 86,400 |
| Peak read RPS | 1,000 | Approx. 8.6× average |
| Average new pastes per second | ~1.16 | 100,000 / 86,400 |
| Average paste record size | 10 KB | Simplified text and metadata estimate |
| Paste record growth | ~1 GB/day | 100,000 × 10 KB, excluding indexes and replicas |
| Read latency target | p95 < 100 ms | Service-side target |
| Availability target | 99.9% | Illustrative objective |

### Capacity notes

- Paste reads greatly outnumber paste-creation writes in this scenario.
- Cache frequently accessed paste IDs to reduce database reads.
- Store view events asynchronously so analytics work does not delay reads.
- Capacity planning must include indexes, replication, backups, logs, and retention—not just raw paste record bytes.

### Questions for a production sizing exercise

- What percentage of paste IDs are accessed repeatedly?
- How skewed is traffic toward popular pastes?
- What is the average and maximum paste content length?
- How long must paste records and view events be retained?
- What percentage of pastes expire or are deleted?

---

## 04 — Naive Architecture

### Simplest viable design

A single web application receives both creation and read requests. It stores paste records in one relational database.

**Components**
- Client/browser
- Single application server
- Relational database

**Create flow**
1. Client sends `POST /api/pastes`.
2. Application validates the paste content.
3. Application generates a paste ID.
4. Application inserts the paste ID and content into the database.
5. Application returns the paste URL.

**Read flow**
1. Browser requests `GET /{paste_id}`.
2. Application queries the database by paste ID.
3. Application returns HTTP 200 with the text if an active paste exists.
4. Browser displays the text safely.

### Naive architecture diagram

See `architecture.png` (V1). The proposed evolved architecture is in `architecture-evolved.png` (V4).

### Why begin here?

The naive design is easy to understand and implement. It establishes the core paste record behaviour before introducing caching, asynchronous analytics, traffic distribution, and failure handling.

---

## 05 — Problems

| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single-server bottleneck | All traffic reaches one process | High latency and limited throughput | Replicate application instances |
| Single point of failure | One server or database outage affects service | Service unavailable | Redundancy and failover |
| Database read pressure | Every read queries the database | Database saturation | Cache-aside lookup |
| Hot keys | Popular pastes receive disproportionate traffic | Uneven load and cache/database pressure | Cache hot paste IDs and protect origin |
| Collision risk | Random or poorly designed paste ID generation | Duplicate paste ID or failed creation | Unique constraint and collision retry |
| Analytics slows reads | View processing occurs synchronously | Increased read latency | Asynchronous event pipeline |
| Abuse | Attackers create malicious or excessive pastes | User harm and service reputation risk | Validation, rate limiting, abuse detection |
| Expiration ambiguity | Expiry checks are inconsistent across layers | Expired pastes may still read | Define authoritative expiry check and cache TTL policy |
| Cache staleness | Deleted or changed pastes remain cached | Incorrect reads | Invalidation or short TTL |
| Poor visibility | No metrics or tracing | Slow diagnosis | Structured logs, metrics, traces, alerts |

---

## 06 — Architecture Evolution

These are proposed learning stages, not a historical account of Pastebin's actual architecture.

### V1 — Single application and database

- One application handles creation and reads.
- One relational database stores paste records.
- Suitable for a prototype and low traffic.
- Limitation: limited horizontal scalability and single points of failure.

### V2 — Replicated application behind a load balancer

See `architecture-v2.png`.

- Add multiple stateless application instances.
- Place CMP-001 Load Balancer in front of them.
- Use health checks to avoid routing to unhealthy instances.
- Keep persistent paste records in a shared database.
- Benefit: application capacity and availability improve.
- New concern: database may become the bottleneck.

### V3 — Cache frequent reads

See `architecture-v3.png`.

- Add a distributed cache in front of the paste database.
- Use cache-aside: read cache; on miss, read database; then populate cache.
- Cache key: `paste_id`.
- Cache value: paste content plus visibility, status and expiry metadata. Cache only public pastes; enforce authorization on private reads.
- Invalidate or update cache when a paste is deactivated or changed.
- New concern: cache eviction, stampedes, stale entries, and hot keys.

### V4 — Separate analytics from read path

See `architecture-evolved.png`.

- Read service publishes a view event to a queue or event stream.
- Analytics workers consume events asynchronously.
- Read response does not wait for analytics aggregation.
- Store raw events separately from paste paste records.
- New concern: duplicate events, delayed aggregation, queue backlog, and retention.

### Global evolution

See `architecture-global.png`.

- Use regional traffic entry points and regional read capacity.
- Replicate or distribute paste record data according to a defined consistency strategy.
- Route visitors to an appropriate region.
- Define behaviour during regional failure and replication lag.
- Decide whether paste creation is region-local or globally coordinated.
- Do not claim globally consistent writes without specifying the mechanism and trade-offs.

---

## 07 — Components

| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Distributes incoming requests across healthy application instances | V2 onward |
| TBD | Client | Creates pastes and retrieves pastes | Yes |
| TBD | DNS | Resolves the paste hostname | Yes |
| TBD | API / Application Service | Validates requests and coordinates paste operations | Yes |
| TBD | ID / Paste-ID Generator | Produces candidate paste IDs | Yes |
| TBD | Paste Database | Persists paste ID-to-content records | Yes |
| TBD | Distributed Cache | Stores frequently accessed paste records | V3 |
| TBD | Message Broker | Buffers view events for asynchronous processing | V4 |
| TBD | Analytics Worker | Processes view events | V4 |
| TBD | Analytics Store | Stores event records or aggregates | V4 |
| TBD | Rate Limiter | Restricts abusive request rates | Recommended |
| TBD | Abuse Detection / Content Safety | Screens suspicious contents and usage | Recommended |
| TBD | Monitoring / Observability | Collects logs, metrics, traces, and alerts | Recommended |

**Component ID policy:** CMP-001 is the only confirmed component ID in this starter package. Other IDs remain TBD until their component specifications are created and added to the master component index. Do not invent IDs independently in case-study documents.

### Component interactions

- Client → DNS → Load Balancer → Application Service.
- Application Service → Paste-ID Generator.
- Application Service → Paste Database.
- Read Service → Cache → Paste Database on cache miss.
- Read Service → Message Broker for view events.
- Analytics Worker → Analytics Store.
- Monitoring receives telemetry from all services.

---

## 08 — Data Flow

### Flow A: Create a paste

1. Client sends `POST /api/v1/pastes` with the paste content.
2. Load balancer selects a healthy application instance.
3. Application authenticates the user if the endpoint requires an account.
4. Application validates encoding, size, visibility, and policy.
5. Paste-ID generator produces a candidate paste ID.
6. Application attempts to insert the paste record under a unique database constraint.
7. If the paste ID already exists, generate another candidate and retry a bounded number of times.
8. Database commits the paste record.
9. Application returns `201 Created` with the paste URL and metadata.

### Flow B: Retrieve a paste

1. Browser requests `GET /{paste_id}`.
2. Load balancer forwards the request to a healthy read-capable instance.
3. Application validates the paste ID format.
4. Application looks up the paste ID in cache.
5. On cache hit, check status, visibility and expiry metadata; authorize private reads against authoritative data.
6. On cache miss, read the authoritative paste database.
7. If no active paste record exists, return `404 Not Found` or a defined expired/deactivated response.
8. If valid and public, populate cache according to the cache policy.
9. Publish a view event asynchronously where possible.
10. Return `200 OK` with the paste content and appropriate content type.
11. Browser displays the paste as escaped text.

**Read status policy:** Return `200 OK` for active accessible pastes, `404 Not Found` for unknown or inaccessible pastes, and a defined expiry response. Use `Cache-Control` appropriate to visibility and expiry; never publicly cache private content.

### Flow C: Analytics

1. Read service emits an event containing event ID, paste ID, timestamp, and permitted metadata.
2. Broker acknowledges durable acceptance according to its configured delivery semantics.
3. Worker consumes the event.
4. Worker validates and deduplicates using event ID where required.
5. Worker updates aggregate counters or stores raw event data.
6. Analytics API reads aggregates for reporting.

### Example paste record

```json
{
  "paste_id": "aB91xQ",
  "content": "Hello, system design!",
  "visibility": "public",
  "owner_id": "user_123",
  "status": "active",
  "created_at": "2026-11-15T10:00:00Z",
  "expires_at": null
}
```

### Example view event

```json
{
  "event_id": "evt_000001",
  "paste_id": "aB91xQ",
  "occurred_at": "2026-11-15T10:05:00Z",
  "referrer_host": "example.org",
  "user_agent_family": "ExampleBrowser"
}
```

Avoid collecting unnecessary personal data. IP addresses and detailed device identifiers require explicit privacy, retention, and access policies.

---

## 09 — Failure Scenarios

| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| One application instance crashes | Health check fails | Requests to that instance fail | Load balancer removes it; other instances continue |
| All application instances unavailable | Health checks and synthetic probes fail | Service unavailable | Restore capacity; use controlled maintenance/failover |
| Database unavailable during creation | Connection errors, failed writes | New pastes cannot be reliably created | Return retriable error; do not claim creation succeeded |
| Database unavailable during cache miss | Database errors | Uncached pastes cannot resolve | Serve valid cached entries if policy allows; return controlled error otherwise |
| Cache unavailable | Cache connection errors | More database reads | Bypass cache with load protection and database capacity limits |
| Cache contains stale paste record | Version/consistency checks or user report | Wrong or disabled content may be served | Invalidate, version, or expire entry |
| Paste ID collision | Unique constraint violation | Creation attempt fails | Generate a new paste ID and retry |
| Broker unavailable | Publish failures / queue health | Analytics event may be delayed or lost depending on design | Durable local buffering or defined best-effort policy; alert |
| Analytics worker crash | Consumer lag increases | Analytics delayed | Restart worker; replay retained events; make processing idempotent |
| Traffic spike | RPS, queue depth, latency rise | Saturation and timeouts | Scale stateless tier, cache hot pastes, rate-limit abusive traffic |
| Malicious content | Safety signals / reports | Visitors may be exposed | Block, warn, deactivate, and provide reporting process |
| Region outage | Regional health checks fail | Regional traffic unavailable | Route to healthy region if data and DNS/routing strategy support it |

For every failure exercise, document: trigger, expected system state, user-visible outcome, recovery action, and metrics proving recovery.

---

## 10 — Scaling

### 10× traffic

- Increase application instance count behind CMP-001.
- Measure load balancer capacity and connection limits.
- Increase cache capacity and monitor hit ratio.
- Add database read replicas only where read consistency requirements permit.
- Separate read and creation workloads if they compete for resources.

### 100× traffic

- Partition paste record data using a well-defined key strategy such as paste-ID hash.
- Distribute cache capacity. Consider object storage for large immutable text bodies, keeping metadata and object keys in the database; publish metadata only after the object is durable.
- Protect databases from cache stampedes and hot-key overload.
- Partition analytics events by a suitable key and process in parallel.
- Apply per-user and per-IP rate limits with privacy-aware controls.

### 1,000× traffic

- Introduce regional traffic entry and a global routing strategy.
- Define data placement, replication lag, conflict handling, and regional failover.
- Consider dedicated read-serving infrastructure.
- Use tiered caching and hot-key replication where justified by measurements.
- Define disaster recovery objectives, capacity headroom, and cost controls.

### Scaling decision checklist

- Is the bottleneck CPU, memory, network, database, cache, or broker?
- Is traffic read-heavy or write-heavy?
- Are hot keys dominating?
- Does scaling add capacity or merely move the bottleneck?
- What consistency guarantees are required?
- What is the operational and financial cost?

---

## 11 — Trade-offs

| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| Paste ID generation | Random ID | Encoded unique numeric ID | Random IDs need collision checks; sequential IDs may reveal creation patterns |
| Storage | Relational database | Key-value store | Relational constraints and queries vs. simple high-volume key lookup |
| Cache strategy | Cache-aside | Write-through | Simpler read path vs. write complexity and cache/database coordination |
| Content delivery | Inline database text | Object storage | Simple atomic writes vs. scalable blob storage and coordination |
| Analytics | Synchronous | Asynchronous | Immediate processing vs. lower read latency and eventual aggregation |
| Expiry | Delete expired records | Retain with expired status | Storage reclamation vs. auditability and predictable response |
| Deployment | Single region | Multi-region | Simplicity and consistency vs. latency, availability, and replication complexity |

### Required decision record format

For every major decision, interns should document:
- Context and problem.
- Options considered.
- Chosen option.
- Why it was selected.
- What is gained.
- What is sacrificed.
- Conditions that would justify revisiting it.

---

## 12 — Cost

No cloud provider, region, instance family, or current price list is specified in the source material. Therefore, this section provides a cost model rather than a fabricated currency estimate.

### Cost categories

| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Load balancer | Hours, capacity units, data processed | Requests, bytes, active connections |
| Application compute | Instance-hours / CPU / memory | RPS, CPU, memory, autoscaling count |
| Database | Compute, storage, IOPS, backups, replicas | Paste record count, read/write rate, retention |
| Cache | Node-hours and memory | Working set, hit ratio, replication |
| Message broker | Throughput, storage, retention | Events/sec, event size, retention |
| Analytics storage | Data scanned and retained | Events/day, bytes/event, query frequency |
| Network | Data transfer and egress | Read and analytics bytes |
| Observability | Ingested and retained telemetry | Log volume, metric cardinality, trace sampling |

### Estimation worksheet

Populate using a selected provider and region:

- Monthly load-balancer cost = provider quote for expected capacity and hours.
- Monthly compute cost = sum(instance count × hourly price × hours).
- Database cost = compute + storage + IOPS + backups + replicas.
- Cache cost = node count × hourly price × hours.
- Broker cost = throughput/storage/retention quote.
- Network cost = measured or estimated ingress/egress × provider rates.
- Observability cost = ingestion + retention + query charges.

Document date, region, currency, pricing URL, assumptions, and whether free-tier discounts are included.

---

## 13 — Security

### Threats and controls

| Threat | Mitigation |
|---|---|
| Harmful or sensitive submitted content | Content policy, abuse reporting, and takedown workflow |
| Phishing through shared pastes | Safe text rendering, abuse reporting, and takedown process |
| Paste-creation spam | Per-account and per-network rate limits, quotas, abuse detection |
| Paste ID guessing | Sufficiently large ID space; avoid exposing sensitive contents |
| Unauthorized paste management | Authentication, authorization, ownership checks |
| Injection in stored text payloads or metadata | Parameterized database access, output encoding |
| Denial of service | Rate limiting, request size limits, edge protections, capacity controls |
| Analytics privacy leakage | Data minimization, access controls, retention policy |
| Secret exposure | Managed secrets, least privilege, rotation |
| Stored cross-site scripting | Escape HTML, serve raw text safely, and apply content security policy |

### Validation rules

- Accept UTF-8 text within the configured byte limit; reject unsupported encodings.
- Reject empty or oversized payloads and invalid metadata.
- Define maximum text and paste ID lengths.
- Preserve text formatting; escape content when rendering HTML.
- Do not execute submitted code or automatically fetch URLs embedded in text.
- Treat all submitted text as untrusted content.

---

## 14 — Observability

### Metrics

| Metric | Type | Why it matters |
|---|---|---|
| `read_requests_total` | Counter | Read traffic volume |
| `read_latency_ms` | Histogram | User-facing performance |
| `read_errors_total` | Counter | Failed read requests |
| `paste_creation_total` | Counter | Creation volume |
| `paste_creation_failures_total` | Counter | Creation reliability |
| `cache_hit_ratio` | Gauge / derived | Cache effectiveness |
| `database_query_latency_ms` | Histogram | Storage bottleneck detection |
| `load_balancer_healthy_targets` | Gauge | Backend health |
| `broker_consumer_lag` | Gauge | Analytics processing delay |
| `analytics_events_dropped_total` | Counter | Event loss |
| `rate_limit_rejections_total` | Counter | Abuse or capacity pressure |

### Logs and traces

- Use a correlation/request ID across the request path.
- Log status, route, latency, dependency outcome, and error class.
- Avoid logging paste bodies or private access tokens.
- Trace cache lookup, database fallback, and event publication.
- Do not use paste IDs, user IDs, or full paste content as unbounded metric labels.

### Example alerts

- Read p95 latency exceeds the service objective for a sustained period.
- Read 5xx rate rises above an agreed threshold.
- Healthy backend count falls below minimum capacity.
- Cache hit ratio drops sharply.
- Database connection pool is close to exhaustion.
- Analytics consumer lag exceeds the freshness objective.

---

## 15 — Simulation

### Learning objective

Let learners observe how traffic distribution, caching, storage latency, and failures affect a text-sharing service.

### Initial simulation controls

| Control | Default | Suggested range |
|---|---:|---:|
| Incoming read RPS | 1,000 | 10–100,000 |
| Paste-creation RPS | 10 | 0–10,000 |
| Application instances | 3 | 1–20 |
| Per-instance capacity | 500 RPS | 100–5,000 |
| Cache enabled | Yes | On / Off |
| Cache hit ratio | 90% | 0–100% |
| Database read capacity | 2,000 ops/sec | 100–20,000 |
| Database read latency | 20 ms | 1–1,000 ms |
| Load-balancing algorithm | Round Robin | Supported algorithms |
| Failed application instances | 0 | 0–N |
| Broker availability | Available | Available / Unavailable |

These are simulator defaults, not production benchmarks.

### Expected outputs

- Requests received, served, rejected, and timed out.
- Per-instance request distribution.
- Estimated queue depth / in-flight requests.
- Cache hits and misses.
- Database reads and saturation.
- Average and p95 simulated latency.
- Error rate and availability indicator.
- Analytics queue backlog.

### Experiments

1. Disable cache and observe database read load.
2. Increase read traffic by 10×.
3. Reduce application instance count.
4. Fail one backend instance.
5. Set cache hit ratio to zero.
6. Increase database latency.
7. Disable the analytics broker and observe read-path behaviour.
8. Compare routing algorithms under uneven backend capacities.

### Simulation model warning

The simulator must state its assumptions and equations. Do not present a simplified queueing model as a physically exact cloud deployment.

---

## 16 — Design Challenge

### Challenge: Build a text-sharing service

Design a service that accepts text snippets, returns shareable paste URLs, and serves readers reliably.

**Requirements**
- Create a paste.
- Retrieve a paste.
- Support unique paste IDs.
- Handle invalid and expired pastes.
- Support high read traffic.
- Collect view analytics asynchronously.
- Explain abuse prevention.

**Constraints for the exercise**
- Start with one application instance and one database.
- Then evolve to three application instances.
- Add a cache.
- Inject an application failure.
- Increase read traffic to 10×.

**Learner deliverables**
1. Functional and non-functional requirements.
2. High-level architecture diagram.
3. API contracts.
4. Data model.
5. Create and read sequence flows.
6. Capacity estimates and assumptions.
7. Failure analysis.
8. Trade-off decision records.
9. Security and observability plan.

**Evaluation prompts**
- How do you guarantee paste ID uniqueness?
- What happens if the database is unavailable?
- How do you invalidate a cached paste record?
- How do you keep analytics from delaying reads?
- How would you support multiple regions?

---

## 17 — AI Interview

### Interviewer question bank

| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What are the core requirements of a Pastebin? | Create, map, read, uniqueness |
| Q02 | Beginner | How would you generate a paste ID? | ID space, uniqueness, collision handling |
| Q03 | Beginner | Which database would you start with and why? | Access pattern, constraints, scale assumptions |
| Q04 | Intermediate | Why is read traffic usually a read-heavy workload in this scenario? | Workload assumptions and read/write ratio |
| Q05 | Intermediate | How would you introduce caching? | Cache-aside, TTL, invalidation |
| Q06 | Intermediate | What happens when a popular paste becomes a hot key? | Hot-key protection, replication, cache strategy |
| Q07 | Intermediate | How would you keep analytics from slowing reads? | Async events, broker, consumer |
| Q08 | Intermediate | What happens when one application server fails? | Health checks, load balancer, retry boundaries |
| Q09 | Advanced | How would you partition the paste database? | Key distribution, shard lookup, rebalancing |
| Q10 | Advanced | How would multi-region paste creation work? | ID coordination, consistency, conflict policy |
| Q11 | Advanced | How would you defend against unsafe paste content? | content policy, safe rendering, abuse workflow |
| Q12 | Advanced | How would you estimate cost? | Workload, provider pricing, storage, network, observability |

### Follow-up evaluation dimensions

- Clarifies assumptions before choosing technology.
- Separates required behaviour from optional features.
- Explains data model and request flow.
- Identifies bottlenecks and failure modes.
- Justifies trade-offs.
- Avoids unsupported claims about real company internals.

---

## 18 — Architecture Score

### Proposed evaluation rubric

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements completeness | 10 | Required user journeys and constraints |
| Scalability | 15 | Traffic estimates, bottleneck analysis, scaling plan |
| Reliability | 15 | Failure handling, redundancy, recovery |
| Latency and performance | 10 | Read path, cache strategy, measured assumptions |
| Data design and consistency | 15 | Schema, uniqueness, expiry, cache consistency |
| Security | 10 | Validation, abuse prevention, access control |
| Cost efficiency | 10 | Transparent cost model and assumptions |
| Trade-off justification | 10 | Alternatives and decision records |
| Observability and operations | 5 | Metrics, logs, traces, alerts |
| **Total** | **100** | |

### Scoring guidance

- 0: Missing or technically incorrect.
- 1: Mentioned without explanation.
- 2: Partially explained.
- 3: Correct and reasonably justified.
- 4: Detailed, measurable, and handles edge cases.

Weighted score = sum of (dimension score / 4 × dimension weight).

The score should be accompanied by evidence and reviewer comments; it should not be treated as an objective measure of architecture quality without human review.
