# CS-ReverseProxy — Design a Reverse Proxy

> **Content status:** Educational reference design, not a claim about NGINX or any vendor's private architecture.
>
> **Source alignment:** Category A: Web & Internet Systems; supplied 18-component template. Official case number unverified.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-ReverseProxy |
| System name | Reverse Proxy |
| Real-world reference | NGINX / HAProxy (reference products) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | To be verified |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | Receive client HTTP requests and forward them to configured upstream services, returning responses to clients. |
| Primary users | Website visitors, service operators, application developers |
| Key concepts | HTTP, TLS, routing, load balancing, health checks, caching, connection pooling, observability |
| Related component IDs | CMP-001 (Load Balancer); additional components TBD |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-15 |

### Scope and assumptions
- HTTP reverse proxy for configured origins; not a general-purpose forward proxy.
- Single region initially; global deployment is an extension.
- TLS termination, upstream health checks and optional public-response caching.
- All figures are illustrative, not vendor measurements.

---
## 01 — Story
Clients need a stable public endpoint while applications run on multiple private backend instances. A reverse proxy accepts requests on behalf of those applications, chooses an upstream and relays its response.
### Actors
1. **Client:** Requests a website or API.
2. **Application operator:** Configures routes and upstream pools.
3. **Backend service:** Processes forwarded requests.
4. **Administrator:** Maintains certificates, health and policies.
### Core user journey
1. Client resolves `app.example.com` and connects using HTTPS.
2. Proxy terminates TLS and matches the configured host and path.
3. Proxy selects a healthy backend or serves an eligible cached response.
4. Backend processes the request; proxy relays the response.
### Problem boundaries
HTTP routing, upstream selection, TLS, caching and operational controls; not application business logic or arbitrary internet forwarding.

---
## 02 — Requirements
### Functional requirements
| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Accept HTTP requests and return upstream responses | Must |
| FR-02 | Route configured hosts and paths | Must |
| FR-03 | Terminate TLS with valid certificates | Must |
| FR-04 | Select healthy upstreams | Must |
| FR-05 | Enforce request and upstream timeouts | Must |
| FR-06 | Support connection pooling and streaming | Should |
| FR-07 | Cache eligible public responses | Should |
| FR-08 | Apply rate and request-size limits | Should |
| FR-09 | Publish validated configuration updates | Should |
| FR-10 | Collect request metrics and logs | Should |
| FR-11 | Support WebSocket upgrades | Could |
### Non-functional requirements
| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Latency | Proxy-added p95 under 10 ms at baseline, excluding network and origin time |
| NFR-02 | Availability | Illustrative 99.9% monthly |
| NFR-03 | Durability | Persist last known good configuration |
| NFR-04 | Consistency | Versioned configuration; bounded rollout delay |
| NFR-05 | Scalability | Scale proxy and origin pools independently |
| NFR-06 | Security | TLS, strict routing and HTTP parsing, trusted forwarding metadata |
| NFR-07 | Observability | Request rates, latency, errors, health and saturation |
| NFR-08 | Maintainability | Separate configuration management from request serving |
### Out of scope for the initial version
- Arbitrary forward proxying, service mesh, global active-active configuration writes and full HTTP/3 support.

---
## 03 — Scale
### Workload assumptions
| Metric | Illustrative value | Reasoning |
|---|---:|---|
| Requests/day | 10,000,000 | Teaching workload |
| Average RPS | ~116 | 10,000,000 / 86,400 |
| Peak RPS | 1,000 | ~8.6× average |
| Average response size | 20 KB | Simplified payload |
| Daily response traffic | 200 GB | Excludes headers and retries |
| Peak response bandwidth | 160 Mbps | 1,000 × 20 KB × 8 |
| Cache-eligible fraction | 50% | Illustrative public GET traffic |
| Hit ratio among eligible requests | 80% | 40% overall hit rate |
| Origin peak RPS | 600 | 1,000 × (1 − 0.4) |
| Availability target | 99.9% | Illustrative objective |
### Capacity notes
- Account for TLS handshakes, concurrent connections, streaming, network and buffers.
- Cache only eligible responses; origin capacity must tolerate cache misses.
### Questions for a production sizing exercise
- What are response sizes, connection durations, traffic bursts, cacheability and origin latency?

---
## 04 — Naive Architecture
### Simplest viable design
**Components:** Client, single reverse proxy, single origin.
**Request flow:** Client connects to proxy; proxy validates route, forwards request and relays response.
### Naive architecture diagram
See `architecture.png`.
### Why begin here?
It isolates the basic forwarding behavior before adding redundancy and caching.

---
## 05 — Problems
| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Proxy bottleneck | One process handles traffic | Saturation | Replicate proxies |
| Single point of failure | One proxy or origin | Outage | Redundant pools |
| Origin overload | Every request forwarded | High latency | Cache eligible responses |
| Unhealthy origin | No health detection | Failed requests | Health checks |
| Slow origin | Long-lived requests | Connection exhaustion | Timeouts and bounded concurrency |
| Stale cache | Response changes | Incorrect content | TTL and invalidation |
| Configuration error | Invalid route rollout | Misrouting | Validate and roll back |
| TLS expiry | Certificate not renewed | Connection failures | Automated renewal and alerts |

---
## 06 — Architecture Evolution
### V1 — Single proxy and origin
One proxy forwards requests to one origin. See `architecture.png`.
### V2 — Replicated proxies and load balancing
CMP-001 distributes connections to healthy proxies; each selects healthy origins. See `architecture-v2.png`.
### V3 — Cache-optimized reverse proxy
Add per-proxy HTTP cache; eligible hits bypass origins. Honor cache directives and Vary. See `architecture-v3.png`.
### V4 — Separate control and data planes
Versioned configuration service distributes validated routes and certificates; proxies retain last known good state. Add asynchronous telemetry. See `architecture-evolved.png`.
### Global evolution
Regional proxy pools, geographic DNS or anycast entry, regional origins and controlled configuration rollout. See `architecture-global.png`.

---
## 07 — Components
| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Distributes connections across healthy proxies | V2 onward |
| TBD | Client | Sends HTTP requests | Yes |
| TBD | DNS | Resolves public hostname | Yes |
| TBD | Reverse Proxy | TLS, routing and response relay | Yes |
| TBD | Origin Pool | Processes application requests | Yes |
| TBD | Health Checker | Tracks upstream readiness | V2 |
| TBD | HTTP Cache | Stores eligible public responses | V3 |
| TBD | Configuration Service | Validates and distributes versions | V4 |
| TBD | Certificate Manager | Manages certificate lifecycle | Recommended |
| TBD | Monitoring / Observability | Collects telemetry | Recommended |

**Component ID policy:** Only CMP-001 is confirmed; other IDs remain TBD.
### Component interactions
Client → DNS lookup → public entry → proxy → origin; cache hits return locally. Control plane publishes configuration separately from traffic.

---
## 08 — Data Flow
### Flow A: Forward a request
1. Accept TLS connection and validate HTTP request.
2. Match configured host/path and select healthy origin.
3. Forward with trusted request metadata and bounded timeouts.
4. Relay status, headers and body; record telemetry asynchronously.
### Flow B: Serve an eligible cached response
1. Check request eligibility and cache key including configured Vary dimensions.
2. Return fresh hit; otherwise forward to origin.
3. Store response only when allowed by HTTP cache policy; relay response.
### Flow C: Configuration update
1. Operator submits authenticated change.
2. Validate routes and certificates; persist version.
3. Distribute to proxies; activate atomically per instance and report status.
### Example route record
```json
{"host":"app.example.com","path_prefix":"/api/","upstream_pool":"api","version":12,"cache_enabled":false}
```
### Example request event
```json
{"request_id":"req_001","route":"api","status":200,"duration_ms":35,"cache_status":"bypass"}
```

---
## 09 — Failure Scenarios
| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| Proxy crash | Health probe | Connections fail | Remove target; clients reconnect |
| Origin crash | Health checks | Some requests fail | Remove origin; bounded safe retries |
| All origins down | Pool health | Cache misses fail | Return controlled 503 |
| Origin timeout | Deadline | Request fails | Return 504 before headers sent; close incomplete streams |
| Cache unavailable | Cache metrics | Origin load rises | Bypass with load protection |
| Bad configuration | Validation / errors | Route failures | Reject or roll back |
| Control plane unavailable | Update failures | Updates delayed | Serve last known good configuration |
| Region outage | Regional probes | Regional service loss | Route to healthy region with capacity |

---
## 10 — Scaling
### 10× traffic
Add proxy and origin instances, tune pools and measure network saturation.
### 100× traffic
Partition by service or tenant, expand cache capacity and isolate noisy workloads.
### 1,000× traffic
Deploy regional pools, global traffic routing, regional capacity and disaster recovery.
### Scaling decision checklist
Measure CPU, bandwidth, concurrent connections, cache hit rate, origin latency, error rate and cost.

---
## 11 — Trade-offs
| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| Routing | Round Robin | Least Connections | Simplicity vs variable request duration |
| TLS | Termination | Passthrough | HTTP inspection vs end-to-end TLS; re-encrypt upstream if required |
| Cache | Local | Shared | Low latency vs coordination and shared dependency |
| Retry | None | Bounded safe retry | Reliability vs amplification and duplicate effects |
| Configuration | Static | Dynamic | Simplicity vs rollout complexity |
| Deployment | Single region | Multi-region | Cost vs resilience |
### Required decision record format
Context, options, chosen option, rationale, gains, sacrifices and revisit conditions.

---
## 12 — Cost
### Cost categories
| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Load balancer | Connections and bytes | RPS, bandwidth |
| Proxy compute | CPU and memory | TLS rate, concurrency |
| Cache | Memory/disk | Working set |
| Network | Egress | Response bytes |
| Control plane | Compute and storage | Config updates |
| Observability | Ingestion and retention | Log volume |
### Estimation worksheet
Monthly compute = instances × hourly price × hours; add load balancer, cache, network, control plane and telemetry charges. Record provider, region, currency, date and pricing source. No fabricated price estimate.

---
## 13 — Security
### Threats and controls
| Threat | Mitigation |
|---|---|
| Unauthorized route changes | Authentication, RBAC, audit logs |
| Untrusted forwarding headers | Replace or validate at trusted boundary |
| HTTP parsing ambiguity | Strict consistent parsing and supported protocol handling |
| Private response leakage | Do not cache private or authenticated responses by default |
| Excessive traffic | Rate limits, bounded concurrency and request sizes |
| Certificate compromise | Protected keys, rotation and renewal |
### Validation rules
Allow configured hosts and upstreams only; validate configuration, enforce HTTP limits and TLS verification for HTTPS origins. Avoid logging credentials or sensitive payloads.

---
## 14 — Observability
### Metrics
| Metric | Type | Why it matters |
|---|---|---|
| proxy_requests_total | Counter | Traffic |
| proxy_latency_ms | Histogram | End-to-end proxy request time |
| upstream_latency_ms | Histogram | Origin latency |
| proxy_errors_total | Counter | Failure rate |
| active_connections | Gauge | Saturation |
| cache_hit_ratio | Gauge / derived | Cache effectiveness |
| healthy_upstreams | Gauge | Origin health |
| config_version | Gauge | Rollout state |
### Logs and traces
Use request IDs, route labels, status and timing; redact secrets and avoid unbounded labels.
### Example alerts
High 5xx rate, p95 latency, low healthy targets, certificate expiry, configuration divergence.

---
## 15 — Simulation
### Learning objective
Observe proxy capacity, cache eligibility, origin pressure and failures.
### Initial simulation controls
| Control | Default | Suggested range |
|---|---:|---:|
| Incoming RPS | 1000 | 10–100000 |
| Proxy instances | 3 | 1–20 |
| Per-instance capacity | 500 | 100–5000 |
| Cache enabled | Yes | On / Off |
| Eligible fraction | 50% | 0–100% |
| Eligible hit ratio | 80% | 0–100% |
| Origin capacity | 2000 | 100–20000 |
| Origin latency | 20 ms | 1–1000 ms |
| Routing algorithm | Round Robin | Supported algorithms |
| Failed proxy instances | 0 | 0–N |
| Origin available | Yes | On / Off |
### Expected outputs
Incoming and successful RPS, rejected RPS, per-instance distribution, cache hits/misses, origin demand, latency and health.
### Experiments
Baseline, cache disabled, 10× traffic, proxy failure, overload, origin latency spike, origin outage, routing comparison.
### Simulation model warning
Simplified teaching model; not production benchmarks. See `simulation-spec.md`.

---
## 16 — Design Challenge
### Challenge: Build a reverse proxy
**Requirements:** HTTP forwarding, configured routing, TLS, health checks, caching and observability.
**Constraints for the exercise:** Start with one proxy and origin; add three proxies, cache, failure and 10× traffic.
**Learner deliverables:** Requirements, diagrams, route schema, request flows, capacity estimates, failure analysis, trade-offs, security and observability plan.
**Evaluation prompts:** How are origins selected? What happens on timeout? How is private content protected? How are configuration updates rolled back?

---
## 17 — AI Interview
### Interviewer question bank
| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What is a reverse proxy? | Client-facing intermediary |
| Q02 | Beginner | How does it differ from a forward proxy? | Represents servers vs clients |
| Q03 | Beginner | How does routing work? | Host/path and origin pools |
| Q04 | Intermediate | How are unhealthy origins handled? | Health checks and failover |
| Q05 | Intermediate | What can be cached? | HTTP directives and privacy |
| Q06 | Intermediate | How do timeouts work? | Bounded deadlines |
| Q07 | Intermediate | How does TLS termination work? | Certificates and upstream TLS |
| Q08 | Intermediate | How do proxies scale? | Replication and connection capacity |
| Q09 | Advanced | How are updates rolled out? | Versioning and rollback |
| Q10 | Advanced | How does global failover work? | Routing and regional capacity |
| Q11 | Advanced | How is tenant isolation maintained? | Route and cache boundaries |
| Q12 | Advanced | How is cost estimated? | Compute, network, cache, telemetry |
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
| Latency and performance | 10 | Proxy and origin latency |
| Data design and consistency | 15 | Route versions and cache policy |
| Security | 10 | TLS and access controls |
| Cost efficiency | 10 | Cost model |
| Trade-off justification | 10 | Decision records |
| Observability and operations | 5 | Metrics and alerts |
| **Total** | **100** | |
### Scoring guidance
0 missing; 1 mentioned; 2 partial; 3 correct and justified; 4 detailed and measurable. Weighted score = sum(score / 4 × weight). Include reviewer evidence and comments.
