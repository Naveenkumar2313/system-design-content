# CS-012 — Design a Web Analytics Platform

> **Content status:** Educational reference design, not a claim about Google Analytics or any vendor's private architecture.
>
> **Source alignment:** Category A: Web & Internet Systems; supplied 18-component template. CS-007 follows the local index; official catalogue number unverified.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-007 |
| System name | Web Analytics Platform |
| Real-world reference | Google Analytics / Matomo (reference products) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 7 (local index; catalogue unverified) |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | Collect permitted website events, process them into metrics and present reports to authorized site owners. |
| Primary users | Website owners, analysts, developers, administrators |
| Key concepts | Event ingestion, queues, stream processing, deduplication, OLAP, aggregation, caching, privacy, observability |
| Related component IDs | CMP-001 (Load Balancer); additional components TBD |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-19 |

### Scope and assumptions
- First-party, permission-aware page views and custom events; no cross-site identity tracking.
- Single region initially; global collection is an extension.
- Reports are eventually consistent; illustrative freshness target of 60 seconds under baseline load.
- All figures are teaching assumptions, not vendor measurements.

---
## 01 — Story
Website owners need to understand which pages are visited and whether users complete intended actions. An analytics platform receives permitted events and turns them into useful aggregate reports.
### Actors
1. **Visitor:** Uses a website and controls applicable analytics preferences.
2. **Website developer:** Integrates the event SDK.
3. **Analyst:** Views authorized reports.
4. **Administrator:** Manages sites, access, retention and operations.
### Core user journey
1. Owner registers a website and configures permitted event collection.
2. SDK sends a page-view event when permitted.
3. Collector validates and durably accepts the event.
4. Worker deduplicates and aggregates events.
5. Analyst opens a dashboard filtered by site and time range.
### Problem boundaries
Event collection, processing, reporting and privacy; not advertising attribution, fingerprinting or cross-site tracking.

---
## 02 — Requirements
### Functional requirements
| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Register websites and manage site access | Must |
| FR-02 | Collect permitted page views and custom events | Must |
| FR-03 | Validate event schema and size | Must |
| FR-04 | Persist accepted events durably | Must |
| FR-05 | Deduplicate retries using site and event ID | Must |
| FR-06 | Aggregate counts by site, time and page | Must |
| FR-07 | Provide filtered dashboards | Must |
| FR-08 | Support consent preferences and retention | Must |
| FR-09 | Track configured conversion events | Should |
| FR-10 | Export authorized aggregate reports | Should |
| FR-11 | Support session metrics where permitted | Could |
### Non-functional requirements
| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Latency | Ingestion p95 under 100 ms; dashboard p95 under 2 s for bounded queries |
| NFR-02 | Availability | Illustrative 99.9% monthly ingestion |
| NFR-03 | Durability | Acknowledge only after durable acceptance |
| NFR-04 | Consistency | Eventual reports; idempotent processing and late-event corrections |
| NFR-05 | Scalability | Independently scale collectors, workers and reporting |
| NFR-06 | Security | TLS, tenant isolation, access control and data minimization |
| NFR-07 | Observability | Event rates, lag, errors, freshness and query latency |
| NFR-08 | Maintainability | Versioned schemas and separable ingestion/reporting |
### Out of scope for the initial version
- Cross-device identity, advertising attribution, predictive analytics and globally consistent real-time reports.

---
## 03 — Scale
### Workload assumptions
| Metric | Illustrative value | Reasoning |
|---|---:|---|
| Registered sites | 10,000 | Teaching workload |
| Events/day | 10,000,000 | Page views and custom events |
| Average events/sec | ~116 | 10,000,000 / 86,400 |
| Peak events/sec | 1,000 | ~8.6× average |
| Average event size | 1 KB | Simplified payload |
| Raw event growth | 10 GB/day | Excludes replicas and indexes |
| Raw retention | 30 days | ~300 GB before compression/replication |
| Peak ingestion bandwidth | 8 Mbps | 1,000 × 1 KB × 8 |
| Dashboard peak RPS | 100 | Independent read workload |
| Availability target | 99.9% | Illustrative objective |
### Capacity notes
- Size broker for bursts and worker outages; size analytical storage for scan volume and retention.
- Batch events where appropriate; event rate is not necessarily HTTP request rate.
### Questions for a production sizing exercise
- What are event sizes, tenant skew, late arrival rate, query windows and retention requirements?

---
## 04 — Naive Architecture
### Simplest viable design
**Components:** Website SDK, single application server, relational database, dashboard.
**Request flow:** SDK submits events; application validates and inserts them; dashboard queries grouped counts through the application.
### Naive architecture diagram
See `architecture.png`.
### Why begin here?
It isolates collection and reporting before introducing queues and analytical storage.

---
## 05 — Problems
| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single-server bottleneck | Shared ingestion and reports | Slow requests | Replicate collectors |
| Single point of failure | One app/database | Outage | Redundancy |
| Traffic bursts | Synchronous inserts | Rejections | Durable broker |
| Expensive reports | Raw row scans | Slow dashboards | OLAP and rollups |
| Duplicate events | Client retries | Inflated counts | Idempotent processing |
| Late events | Delayed delivery | Incomplete windows | Event-time corrections |
| Hot tenants | Uneven traffic | Resource contention | Quotas and partitioning |
| Privacy leakage | Excessive collection | Exposure | Minimize data and enforce access |

---
## 06 — Architecture Evolution
### V1 — Single application and database
Synchronous event inserts and SQL reports. See `architecture.png`.
### V2 — Replicated collectors and load balancing
CMP-001 distributes requests across stateless collectors; shared database persists events. See `architecture-v2.png`.
### V3 — Durable event pipeline
Collectors acknowledge after broker acceptance; workers deduplicate and write events to analytical storage; reports query that store. See `architecture-v3.png`.
### V4 — Separate ingestion, aggregation and reporting
Add raw archive, event-time rollups, report API and tenant-scoped cache. Worker commits idempotent results before acknowledging offsets. See `architecture-evolved.png`.
### Global evolution
Regional collectors and brokers; regional processing and storage; authorized reporting merges compatible aggregates subject to residency policy. See `architecture-global.png`.

---
## 07 — Components
| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Distributes requests to healthy collectors | V2 onward |
| TBD | Website SDK | Sends permitted events | Yes |
| TBD | DNS | Resolves endpoints | Yes |
| TBD | Collector | Validates and accepts events | Yes |
| TBD | Event Database | Persists events in baseline | V1–V2 |
| TBD | Event Broker | Durable buffering | V3 |
| TBD | Processing Worker | Deduplication and normalization | V3 |
| TBD | Analytical Store | Stores events and rollups | V3 |
| TBD | Raw Archive | Retains events for controlled replay | V4 |
| TBD | Reporting API | Authorizes and queries reports | Yes |
| TBD | Report Cache | Stores tenant-scoped results | V4 |
| TBD | Monitoring / Observability | Collects telemetry | Recommended |

**Component ID policy:** Only CMP-001 is confirmed; other IDs remain TBD.
### Component interactions
SDK → load balancer → collector → broker → worker → analytical store; worker also writes raw archive in V4. Dashboard → reporting API → cache or analytical store.

---
## 08 — Data Flow
### Flow A: Collect an event
1. SDK checks applicable preferences and sends `POST /api/v1/events` with event ID and site ID.
2. Collector validates schema, site, size and quotas; public site ID is not a secret or proof of authenticity.
3. Collector publishes to durable broker and waits for acknowledgment.
4. Return `202 Accepted`; on unavailable broker return retriable 503, not success.
### Flow B: Process events
1. Worker consumes events and validates timestamps.
2. Deduplicate by `(site_id,event_id)` within the supported replay horizon.
3. Write canonical event and update rollups idempotently; commit offset after durable success.
4. Correct affected windows for late events and invalidate/version report cache.
### Flow C: Generate a report
1. Analyst requests `GET /api/v1/reports?site_id=site_123&metric=pageviews` with bounded time filters.
2. API authenticates and authorizes site access before cache lookup.
3. Query tenant-scoped cache or analytical rollups; return counts and freshness watermark.
### Example event record
```json
{"site_id":"site_123","event_id":"evt_001","event_type":"page_view","page_path":"/docs","occurred_at":"2026-11-19T10:00:00Z","schema_version":1}
```
### Example aggregate record
```json
{"site_id":"site_123","window_start":"2026-11-19T10:00:00Z","page_path":"/docs","page_views":42,"version":2}
```

---
## 09 — Failure Scenarios
| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| Collector crash | Health probe | Some requests fail | Remove target; retry with same ID |
| Broker unavailable | Publish errors | New events not accepted | Return 503; bounded retry |
| Worker crash | Consumer lag | Reports delayed | Restart and replay idempotently |
| Analytical store outage | Write/query errors | Backlog and report failures | Pause consumption; recover and replay |
| Duplicate delivery | Duplicate ID count | Potential overcount | Idempotent sink and rollups |
| Late event | Watermark lag | Window changes | Correct and version rollups |
| Cache unavailable | Cache errors | Query load rises | Bypass with query limits |
| Region outage | Regional probes | Collection interrupted | Route only where residency permits |

---
## 10 — Scaling
### 10× traffic
Add collectors and workers, increase broker partitions and batch writes.
### 100× traffic
Partition by site and event hash, isolate hot tenants, use columnar storage and precomputed rollups.
### 1,000× traffic
Regional pipelines, tiered retention, federated reporting and disaster recovery; avoid summing distinct-user counts across regions.
### Scaling decision checklist
Measure ingestion CPU, broker lag, partition skew, storage throughput, query scans, freshness and cost.

---
## 11 — Trade-offs
| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| Ingestion | Direct insert | Durable broker | Simplicity vs buffering and decoupling |
| Storage | Relational | Columnar OLAP | Transactions vs analytical scans |
| Processing | Batch | Streaming | Cost vs freshness |
| Delivery | At-most-once | At-least-once | Loss vs duplicates requiring idempotence |
| Reporting | Raw queries | Rollups | Flexibility vs speed |
| Deployment | Single region | Multi-region | Simplicity vs resilience and residency |
### Required decision record format
Context, options, chosen option, rationale, gains, sacrifices and revisit conditions.

---
## 12 — Cost
### Cost categories
| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Load balancer | Requests and bytes | RPS, bandwidth |
| Collector compute | CPU and memory | Events/sec |
| Broker | Throughput and retention | Bytes/sec, backlog |
| Workers | Processing compute | Events/sec |
| Analytical store | Storage and scans | GB retained, query volume |
| Archive | Storage and retrieval | Retention, replay volume |
| Report cache | Memory | Working set |
| Observability | Ingestion and retention | Telemetry volume |
### Estimation worksheet
Monthly compute = instances × hourly price × hours; add broker, storage, scans, network, cache and telemetry. Record provider, region, currency, date and pricing source. No fabricated price estimate.

---
## 13 — Security
### Threats and controls
| Threat | Mitigation |
|---|---|
| Unauthorized reports | Authentication, site-level RBAC and audit logs |
| Event spam | Quotas, validation and anomaly monitoring |
| Cross-tenant leakage | Tenant filters and scoped cache keys |
| Sensitive payloads | Allowlisted fields; strip query strings and secrets |
| Excessive tracking | Respect preferences, minimize identifiers and retention |
| Unauthorized deletion | Authorized workflows and audit trails |
### Validation rules
Use TLS, bounded payloads, schema validation and parameterized queries. Avoid collecting credentials, full URLs with sensitive parameters or unnecessary personal identifiers. Apply deletion and retention to raw data, derived data and caches according to policy.

---
## 14 — Observability
### Metrics
| Metric | Type | Why it matters |
|---|---|---|
| events_received_total | Counter | Offered traffic |
| events_accepted_total | Counter | Durable acceptance |
| events_rejected_total | Counter | Validation or capacity failures |
| ingestion_latency_ms | Histogram | Collector performance |
| broker_consumer_lag | Gauge | Backlog |
| duplicate_events_total | Counter | Retry behavior |
| report_freshness_seconds | Gauge | Data delay |
| report_latency_ms | Histogram | Dashboard performance |
### Logs and traces
Use request IDs, schema version, status and dependency timings; redact payloads and avoid unbounded tenant labels.
### Example alerts
High rejection rate, growing broker lag, stale reports, storage errors and elevated query latency.

---
## 15 — Simulation
### Learning objective
Observe collector capacity, durable buffering, worker throughput, report freshness and failures.
### Initial simulation controls
| Control | Default | Suggested range |
|---|---:|---:|
| Incoming events/sec | 1000 | 10–100000 |
| Collector instances | 3 | 1–20 |
| Per-collector capacity | 500 | 100–5000 |
| Worker instances | 3 | 1–20 |
| Per-worker capacity | 400 | 100–5000 |
| Broker capacity | 5000 | 100–100000 |
| Broker backlog limit | 100000 | 1000–1000000 |
| Failed collectors | 0 | 0–N |
| Broker available | Yes | On / Off |
| Store available | Yes | On / Off |
| Routing algorithm | Round Robin | Supported algorithms |
### Expected outputs
Accepted/rejected events, processed events, backlog, per-collector distribution, utilization and modeled freshness.
### Experiments
Baseline, 10× traffic, collector failure, worker saturation, broker outage, store outage, recovery and routing comparison.
### Simulation model warning
Simplified one-second fluid model; not production benchmarks. See `simulation-spec.md`.

---
## 16 — Design Challenge
### Challenge: Build a web analytics platform
**Requirements:** Event collection, durable acceptance, deduplication, aggregation, reports and privacy controls.
**Constraints for the exercise:** Start with one application and database; add three collectors, broker, workers, failure and 10× traffic.
**Learner deliverables:** Requirements, diagrams, event schema, flows, capacity estimates, failure analysis, trade-offs, security and observability plan.
**Evaluation prompts:** When is an event acknowledged? How are retries deduplicated? How are late events corrected? How is tenant access enforced?

---
## 17 — AI Interview
### Interviewer question bank
| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What is web analytics? | Events and aggregate reports |
| Q02 | Beginner | What is a page view? | Defined event semantics |
| Q03 | Beginner | Why use a broker? | Durable buffering |
| Q04 | Intermediate | How do retries affect counts? | Deduplication and idempotence |
| Q05 | Intermediate | Why use OLAP? | Columnar scans and rollups |
| Q06 | Intermediate | How are late events handled? | Event time and corrections |
| Q07 | Intermediate | How do reports stay fast? | Rollups, filters and cache |
| Q08 | Intermediate | What happens if workers fail? | Lag and replay |
| Q09 | Advanced | How do you partition events? | Tenant skew and hash distribution |
| Q10 | Advanced | How do global reports work? | Residency and compatible aggregates |
| Q11 | Advanced | How do you protect privacy? | Preferences, minimization, deletion |
| Q12 | Advanced | How do you estimate cost? | Throughput, retention, scans |
### Follow-up evaluation dimensions
Assumptions, requirements, flows, bottlenecks, trade-offs and evidence quality.

---
## 18 — Architecture Score
### Proposed evaluation rubric
| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements completeness | 10 | User journeys |
| Scalability | 15 | Capacity plan |
| Reliability | 15 | Failure recovery |
| Latency and performance | 10 | Ingestion and reporting latency |
| Data design and consistency | 15 | Event IDs, rollups and freshness |
| Security | 10 | Privacy and access controls |
| Cost efficiency | 10 | Cost model |
| Trade-off justification | 10 | Decision records |
| Observability and operations | 5 | Metrics and alerts |
| **Total** | **100** | |
### Scoring guidance
0 missing; 1 mentioned; 2 partial; 3 correct and justified; 4 detailed and measurable. Weighted score = sum(score / 4 × weight). Include reviewer evidence and comments.
