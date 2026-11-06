# CS-001 — Design a URL Shortener (Bitly)

> **Content status:** Educational reference design. This document is a proposed design for a Bitly-like URL-shortening service. It does not claim to describe Bitly's private production architecture.
>
> **Source alignment:** The Codorythm case-study catalogue identifies this as Case Study 1 in Category A: Web & Internet Systems. The required structure follows the 18-component case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-001 |
| System name | URL Shortener |
| Real-world reference | Bitly (reference product, not an asserted architecture source) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 1 |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | A service that accepts a long URL, generates a compact unique alias, stores the mapping, and redirects visitors from the short URL to the original destination. |
| Primary users | Link creators, link visitors, administrators, analytics consumers |
| Key concepts | HTTP, DNS, API design, unique ID generation, relational/key-value storage, caching, redirects, rate limiting, analytics, observability |
| Related component IDs | CMP-001 (Load Balancer); additional components can be added to the component index as they are documented |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-06 |

### Scope and assumptions

- Users can create a short link from a valid HTTP or HTTPS URL.
- The service returns a short URL using a domain controlled by the service, such as `sho.rt`.
- Visiting the short URL redirects the visitor to the stored destination.
- Links may optionally have a custom alias, expiration time, and owner.
- Basic click analytics are included as an extension.
- The initial design is a single-region educational deployment; global distribution is an evolution stage.
- Traffic figures below are illustrative planning assumptions, not measured Bitly figures.

---

## 01 — Story

People frequently share long URLs through messages, documents, social posts, QR codes, and printed material. Long URLs are difficult to read, consume space, and can break when copied or displayed.

A URL-shortening service creates a compact alias that points to the original URL. A creator submits a destination URL and receives a short link. A visitor opens the short link; the service looks up the destination and returns an HTTP redirect.

### Actors

1. **Link creator:** Submits a long URL and manages links.
2. **Link visitor:** Opens a short URL.
3. **Analytics consumer:** Views click counts and aggregated activity.
4. **Administrator:** Handles abuse reports, policy enforcement, and service operations.

### Core user journey

1. Creator submits `https://example.com/articles/system-design-introduction`.
2. Service validates the URL.
3. Service generates a unique code, for example `aB91xQ`.
4. Service stores `aB91xQ → https://example.com/articles/system-design-introduction`.
5. Service returns `https://sho.rt/aB91xQ`.
6. Visitor requests `https://sho.rt/aB91xQ`.
7. Service resolves the code and redirects the visitor to the destination.

### Problem boundaries

This case study focuses on link creation, alias resolution, redirects, basic link management, and click-event collection. It does not attempt to reproduce every commercial feature of Bitly.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Accept a valid destination URL and create a short link | Must |
| FR-02 | Generate a unique short code | Must |
| FR-03 | Redirect a visitor to the mapped destination | Must |
| FR-04 | Reject malformed or unsupported destination URLs | Must |
| FR-05 | Prevent accidental alias collisions | Must |
| FR-06 | Allow a user to request a custom alias, subject to availability | Should |
| FR-07 | Allow an owner to view and deactivate their links | Should |
| FR-08 | Support optional expiration timestamps | Should |
| FR-09 | Record click events for analytics | Should |
| FR-10 | Provide aggregated click statistics | Could |
| FR-11 | Detect and restrict abusive or malicious links | Should |

### Non-functional requirements

| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Redirect latency | Target p95 under 100 ms inside the service region, excluding visitor-to-service network time |
| NFR-02 | Availability | Illustrative target 99.9% monthly for redirect service |
| NFR-03 | Durability | A successfully created mapping should not be lost after an ordinary process restart |
| NFR-04 | Consistency | A newly created alias should resolve after successful creation; cache propagation must not expose stale mappings |
| NFR-05 | Scalability | Redirect traffic should scale independently from link creation |
| NFR-06 | Security | Validate URLs, protect management APIs, rate-limit creation, and prevent open redirect abuse |
| NFR-07 | Observability | Track request rate, latency, errors, cache hit ratio, and dependency health |
| NFR-08 | Maintainability | Keep link management, redirect, and analytics responsibilities separable |

### Out of scope for the initial version

- Full enterprise account management and billing.
- Advanced campaign attribution.
- Multi-region active-active writes.
- Guaranteed real-time analytics.
- Browser extensions and mobile applications.

---

## 03 — Scale

All values in this section are **illustrative workload assumptions for teaching and simulation**. They are not Bitly traffic disclosures.

### Workload assumptions

| Metric | Illustrative value | Reasoning |
|---|---:|---|
| Registered users | 100,000 | Small-to-medium learning scenario |
| Links created per day | 100,000 | Write workload |
| Redirects per day | 10,000,000 | Redirect-heavy workload |
| Average redirect RPS | ~116 | 10,000,000 / 86,400 |
| Peak redirect RPS | 1,000 | Approx. 8.6× average |
| Average new links per second | ~1.16 | 100,000 / 86,400 |
| Average mapping size | 1 KB | Simplified storage estimate |
| Mapping growth | ~100 MB/day | 100,000 × 1 KB, excluding indexes and replicas |
| Redirect latency target | p95 < 100 ms | Service-side target |
| Availability target | 99.9% | Illustrative objective |

### Capacity notes

- Redirect reads greatly outnumber link-creation writes in this scenario.
- Cache frequently accessed aliases to reduce database reads.
- Store click events asynchronously so analytics work does not delay redirects.
- Capacity planning must include indexes, replication, backups, logs, and retention—not just raw mapping bytes.

### Questions for a production sizing exercise

- What percentage of aliases are accessed repeatedly?
- How skewed is traffic toward popular links?
- What is the average and maximum destination URL length?
- How long must mappings and click events be retained?
- What percentage of links expire or are deleted?

---

## 04 — Naive Architecture

### Simplest viable design

A single web application receives both creation and redirect requests. It stores mappings in one relational database.

**Components**
- Client/browser
- Single application server
- Relational database

**Create flow**
1. Client sends `POST /api/links`.
2. Application validates the destination URL.
3. Application generates a code.
4. Application inserts the code and destination into the database.
5. Application returns the short URL.

**Redirect flow**
1. Browser requests `GET /{code}`.
2. Application queries the database by code.
3. Application returns a redirect response if a mapping exists.
4. Browser follows the redirect.

### Naive architecture diagram

See `architecture.png`.

### Why begin here?

The naive design is easy to understand and implement. It establishes the core mapping behaviour before introducing caching, asynchronous analytics, traffic distribution, and failure handling.

---

## 05 — Problems

| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single-server bottleneck | All traffic reaches one process | High latency and limited throughput | Replicate application instances |
| Single point of failure | One server or database outage affects service | Service unavailable | Redundancy and failover |
| Database read pressure | Every redirect queries the database | Database saturation | Cache-aside lookup |
| Hot keys | Popular short links receive disproportionate traffic | Uneven load and cache/database pressure | Cache hot aliases and protect origin |
| Collision risk | Random or poorly designed code generation | Duplicate alias or failed creation | Unique constraint and collision retry |
| Analytics slows redirects | Click processing occurs synchronously | Increased redirect latency | Asynchronous event pipeline |
| Abuse | Attackers create malicious or excessive links | User harm and service reputation risk | Validation, rate limiting, abuse detection |
| Expiration ambiguity | Expiry checks are inconsistent across layers | Expired links may still redirect | Define authoritative expiry check and cache TTL policy |
| Cache staleness | Deleted or changed links remain cached | Incorrect redirects | Invalidation or short TTL |
| Poor visibility | No metrics or tracing | Slow diagnosis | Structured logs, metrics, traces, alerts |

---

## 06 — Architecture Evolution

These are proposed learning stages, not a historical account of Bitly's actual architecture.

### V1 — Single application and database

- One application handles creation and redirects.
- One relational database stores mappings.
- Suitable for a prototype and low traffic.
- Limitation: limited horizontal scalability and single points of failure.

### V2 — Replicated application behind a load balancer

- Add multiple stateless application instances.
- Place CMP-001 Load Balancer in front of them.
- Use health checks to avoid routing to unhealthy instances.
- Keep persistent mappings in a shared database.
- Benefit: application capacity and availability improve.
- New concern: database may become the bottleneck.

### V3 — Cache frequent redirects

- Add a distributed cache in front of the mapping database.
- Use cache-aside: read cache; on miss, read database; then populate cache.
- Cache key: `short_code`.
- Cache value: destination URL plus status and expiry metadata.
- Invalidate or update cache when a link is deactivated or changed.
- New concern: cache eviction, stampedes, stale entries, and hot keys.

### V4 — Separate analytics from redirect path

- Redirect service publishes a click event to a queue or event stream.
- Analytics workers consume events asynchronously.
- Redirect response does not wait for analytics aggregation.
- Store raw events separately from link mappings.
- New concern: duplicate events, delayed aggregation, queue backlog, and retention.

### Global evolution

- Use regional traffic entry points and regional redirect capacity.
- Replicate or distribute mapping data according to a defined consistency strategy.
- Route visitors to an appropriate region.
- Define behaviour during regional failure and replication lag.
- Decide whether link creation is region-local or globally coordinated.
- Do not claim globally consistent writes without specifying the mechanism and trade-offs.

---

## 07 — Components

| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Distributes incoming requests across healthy application instances | V2 onward |
| TBD | Client | Creates links and follows redirects | Yes |
| TBD | DNS | Resolves the short-link hostname | Yes |
| TBD | API / Application Service | Validates requests and coordinates link operations | Yes |
| TBD | ID / Short-Code Generator | Produces candidate aliases | Yes |
| TBD | Mapping Database | Persists alias-to-destination mappings | Yes |
| TBD | Distributed Cache | Stores frequently accessed mappings | V3 |
| TBD | Message Broker | Buffers click events for asynchronous processing | V4 |
| TBD | Analytics Worker | Processes click events | V4 |
| TBD | Analytics Store | Stores event records or aggregates | V4 |
| TBD | Rate Limiter | Restricts abusive request rates | Recommended |
| TBD | Abuse Detection / URL Safety | Screens suspicious destinations and usage | Recommended |
| TBD | Monitoring / Observability | Collects logs, metrics, traces, and alerts | Recommended |

**Component ID policy:** CMP-001 is the only confirmed component ID in this starter package. Other IDs remain TBD until their component specifications are created and added to the master component index. Do not invent IDs independently in case-study documents.

### Component interactions

- Client → DNS → Load Balancer → Application Service.
- Application Service → Short-Code Generator.
- Application Service → Mapping Database.
- Redirect Service → Cache → Mapping Database on cache miss.
- Redirect Service → Message Broker for click events.
- Analytics Worker → Analytics Store.
- Monitoring receives telemetry from all services.

---

## 08 — Data Flow

### Flow A: Create a short link

1. Client sends `POST /api/v1/links` with the destination URL.
2. Load balancer selects a healthy application instance.
3. Application authenticates the user if the endpoint requires an account.
4. Application validates scheme, syntax, length, and policy.
5. Short-code generator produces a candidate code.
6. Application attempts to insert the mapping under a unique database constraint.
7. If the code already exists, generate another candidate and retry a bounded number of times.
8. Database commits the mapping.
9. Application returns `201 Created` with the short URL and metadata.

### Flow B: Redirect a visitor

1. Browser requests `GET /{short_code}`.
2. Load balancer forwards the request to a healthy redirect-capable instance.
3. Application validates the code format.
4. Application looks up the code in cache.
5. On cache hit, check status and expiry metadata.
6. On cache miss, read the authoritative mapping database.
7. If no active mapping exists, return `404 Not Found` or a defined expired/deactivated response.
8. If valid, populate cache according to the cache policy.
9. Publish a click event asynchronously where possible.
10. Return `302 Found` (temporary redirect) with the destination in the `Location` header.
11. Browser requests the destination URL.

**Redirect status policy:** Use `302` as the default in this educational design when destinations may change or analytics/disable behaviour must remain under service control. A `301` can be considered for permanent mappings, but browser and intermediary caching can make later changes difficult.

### Flow C: Analytics

1. Redirect service emits an event containing event ID, short code, timestamp, and permitted metadata.
2. Broker acknowledges durable acceptance according to its configured delivery semantics.
3. Worker consumes the event.
4. Worker validates and deduplicates using event ID where required.
5. Worker updates aggregate counters or stores raw event data.
6. Analytics API reads aggregates for reporting.

### Example mapping record

```json
{
  "short_code": "aB91xQ",
  "destination_url": "https://example.com/articles/system-design-introduction",
  "owner_id": "user_123",
  "status": "active",
  "created_at": "2026-11-06T10:00:00Z",
  "expires_at": null
}
```

### Example click event

```json
{
  "event_id": "evt_000001",
  "short_code": "aB91xQ",
  "occurred_at": "2026-11-06T10:05:00Z",
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
| Database unavailable during creation | Connection errors, failed writes | New links cannot be reliably created | Return retriable error; do not claim creation succeeded |
| Database unavailable during cache miss | Database errors | Uncached links cannot resolve | Serve valid cached entries if policy allows; return controlled error otherwise |
| Cache unavailable | Cache connection errors | More database reads | Bypass cache with load protection and database capacity limits |
| Cache contains stale mapping | Version/consistency checks or user report | Wrong or disabled destination may be served | Invalidate, version, or expire entry |
| Code collision | Unique constraint violation | Creation attempt fails | Generate a new code and retry |
| Broker unavailable | Publish failures / queue health | Analytics event may be delayed or lost depending on design | Durable local buffering or defined best-effort policy; alert |
| Analytics worker crash | Consumer lag increases | Analytics delayed | Restart worker; replay retained events; make processing idempotent |
| Traffic spike | RPS, queue depth, latency rise | Saturation and timeouts | Scale stateless tier, cache hot links, rate-limit abusive traffic |
| Malicious destination | Safety signals / reports | Visitors may be exposed | Block, warn, deactivate, and provide reporting process |
| Region outage | Regional health checks fail | Regional traffic unavailable | Route to healthy region if data and DNS/routing strategy support it |

For every failure exercise, document: trigger, expected system state, user-visible outcome, recovery action, and metrics proving recovery.

---

## 10 — Scaling

### 10× traffic

- Increase application instance count behind CMP-001.
- Measure load balancer capacity and connection limits.
- Increase cache capacity and monitor hit ratio.
- Add database read replicas only where read consistency requirements permit.
- Separate redirect and creation workloads if they compete for resources.

### 100× traffic

- Partition mapping data using a well-defined key strategy such as short-code hash.
- Distribute cache capacity.
- Protect databases from cache stampedes and hot-key overload.
- Partition analytics events by a suitable key and process in parallel.
- Apply per-user and per-IP rate limits with privacy-aware controls.

### 1,000× traffic

- Introduce regional traffic entry and a global routing strategy.
- Define data placement, replication lag, conflict handling, and regional failover.
- Consider dedicated redirect-serving infrastructure.
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
| Code generation | Random code | Encoded unique numeric ID | Random codes need collision checks; sequential IDs may reveal creation patterns |
| Storage | Relational database | Key-value store | Relational constraints and queries vs. simple high-volume key lookup |
| Cache strategy | Cache-aside | Write-through | Simpler read path vs. write complexity and cache/database coordination |
| Redirect | 301 | 302 | Permanent caching vs. flexibility and control |
| Analytics | Synchronous | Asynchronous | Immediate processing vs. lower redirect latency and eventual aggregation |
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
| Database | Compute, storage, IOPS, backups, replicas | Mapping count, read/write rate, retention |
| Cache | Node-hours and memory | Working set, hit ratio, replication |
| Message broker | Throughput, storage, retention | Events/sec, event size, retention |
| Analytics storage | Data scanned and retained | Events/day, bytes/event, query frequency |
| Network | Data transfer and egress | Redirect and analytics bytes |
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
| Malicious URL destination | Scheme allowlist, destination safety checks, abuse reporting |
| Phishing through trusted short domain | Warning/interstitial policy, reputation checks, takedown process |
| Link-creation spam | Per-account and per-network rate limits, quotas, abuse detection |
| Alias guessing | Sufficiently large code space; avoid exposing sensitive destinations |
| Unauthorized link management | Authentication, authorization, ownership checks |
| Injection in stored URLs or metadata | Parameterized database access, output encoding |
| Denial of service | Rate limiting, request size limits, edge protections, capacity controls |
| Analytics privacy leakage | Data minimization, access controls, retention policy |
| Secret exposure | Managed secrets, least privilege, rotation |
| Internal-network URL abuse | Block unsafe schemes and private/link-local destinations where server-side fetching is performed |

### Validation rules

- Accept only `http` and `https` schemes unless a deliberate product requirement says otherwise.
- Reject malformed URLs and control characters.
- Define maximum URL and alias lengths.
- Normalize carefully; do not alter semantically meaningful URL components.
- If the service fetches a destination for preview or safety scanning, defend against SSRF and DNS rebinding.
- Do not assume that a syntactically valid URL is safe.

---

## 14 — Observability

### Metrics

| Metric | Type | Why it matters |
|---|---|---|
| `redirect_requests_total` | Counter | Redirect traffic volume |
| `redirect_latency_ms` | Histogram | User-facing performance |
| `redirect_errors_total` | Counter | Failed redirect requests |
| `link_creation_total` | Counter | Creation volume |
| `link_creation_failures_total` | Counter | Creation reliability |
| `cache_hit_ratio` | Gauge / derived | Cache effectiveness |
| `database_query_latency_ms` | Histogram | Storage bottleneck detection |
| `load_balancer_healthy_targets` | Gauge | Backend health |
| `broker_consumer_lag` | Gauge | Analytics processing delay |
| `analytics_events_dropped_total` | Counter | Event loss |
| `rate_limit_rejections_total` | Counter | Abuse or capacity pressure |

### Logs and traces

- Use a correlation/request ID across the request path.
- Log status, route, latency, dependency outcome, and error class.
- Avoid logging full destination URLs if they may contain sensitive query parameters.
- Trace cache lookup, database fallback, and event publication.
- Do not use short codes, user IDs, or full URLs as unbounded metric labels.

### Example alerts

- Redirect p95 latency exceeds the service objective for a sustained period.
- Redirect 5xx rate rises above an agreed threshold.
- Healthy backend count falls below minimum capacity.
- Cache hit ratio drops sharply.
- Database connection pool is close to exhaustion.
- Analytics consumer lag exceeds the freshness objective.

---

## 15 — Simulation

### Learning objective

Let learners observe how traffic distribution, caching, storage latency, and failures affect a URL-shortening service.

### Initial simulation controls

| Control | Default | Suggested range |
|---|---:|---:|
| Incoming redirect RPS | 1,000 | 10–100,000 |
| Link-creation RPS | 10 | 0–10,000 |
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
2. Increase redirect traffic by 10×.
3. Reduce application instance count.
4. Fail one backend instance.
5. Set cache hit ratio to zero.
6. Increase database latency.
7. Disable the analytics broker and observe redirect-path behaviour.
8. Compare routing algorithms under uneven backend capacities.

### Simulation model warning

The simulator must state its assumptions and equations. Do not present a simplified queueing model as a physically exact cloud deployment.

---

## 16 — Design Challenge

### Challenge: Build a URL-shortening service

Design a service that accepts long URLs, returns compact links, and redirects visitors reliably.

**Requirements**
- Create a short link.
- Redirect a visitor.
- Support unique aliases.
- Handle invalid and expired links.
- Support high read traffic.
- Collect click analytics asynchronously.
- Explain abuse prevention.

**Constraints for the exercise**
- Start with one application instance and one database.
- Then evolve to three application instances.
- Add a cache.
- Inject an application failure.
- Increase redirect traffic to 10×.

**Learner deliverables**
1. Functional and non-functional requirements.
2. High-level architecture diagram.
3. API contracts.
4. Data model.
5. Create and redirect sequence flows.
6. Capacity estimates and assumptions.
7. Failure analysis.
8. Trade-off decision records.
9. Security and observability plan.

**Evaluation prompts**
- How do you guarantee code uniqueness?
- What happens if the database is unavailable?
- How do you invalidate a cached mapping?
- How do you keep analytics from delaying redirects?
- How would you support multiple regions?

---

## 17 — AI Interview

### Interviewer question bank

| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What are the core requirements of a URL shortener? | Create, map, redirect, uniqueness |
| Q02 | Beginner | How would you generate a short code? | Code space, uniqueness, collision handling |
| Q03 | Beginner | Which database would you start with and why? | Access pattern, constraints, scale assumptions |
| Q04 | Intermediate | Why is redirect traffic usually a read-heavy workload in this scenario? | Workload assumptions and read/write ratio |
| Q05 | Intermediate | How would you introduce caching? | Cache-aside, TTL, invalidation |
| Q06 | Intermediate | What happens when a popular short link becomes a hot key? | Hot-key protection, replication, cache strategy |
| Q07 | Intermediate | How would you keep analytics from slowing redirects? | Async events, broker, consumer |
| Q08 | Intermediate | What happens when one application server fails? | Health checks, load balancer, retry boundaries |
| Q09 | Advanced | How would you partition the mapping database? | Key distribution, shard lookup, rebalancing |
| Q10 | Advanced | How would multi-region link creation work? | ID coordination, consistency, conflict policy |
| Q11 | Advanced | How would you defend against malicious redirects? | URL policy, reputation, abuse workflow |
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
| Latency and performance | 10 | Redirect path, cache strategy, measured assumptions |
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
