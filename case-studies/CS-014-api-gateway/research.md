# CS-014 — Design an API Gateway

> **Content status:** Educational reference design; not a claim about a vendor's private architecture.
>
> **Source alignment:** Category A: Web & Internet Systems; supplied 18-component template. CS-009 follows the local index; official catalogue number unverified.

## A. Basic information


| Field | Value |
| --- | --- |
| Case Study ID | CS-009 |
| System name | API Gateway |
| Real-world reference | Amazon API Gateway / Kong Gateway (reference products) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 9 (local index; catalogue unverified) |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | A managed entry point that authenticates, authorizes, limits and routes API requests to configured backend services. |
| Primary users | API consumers, service developers, platform operators, administrators |
| Key concepts | HTTP, TLS, routing, authentication, authorization, rate limiting, load balancing, caching, configuration, observability |
| Related component IDs | CMP-001 (Load Balancer); additional components TBD |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-19 |

### Scope and assumptions
- HTTP/HTTPS JSON APIs with configured upstream services; single region initially.
- Gateway validates access and routes requests; backend services retain business authorization and data ownership.
- Optional caching only for explicitly eligible public GET responses.
- All traffic figures are illustrative teaching assumptions.


---
## 01 — Story
Clients need a stable API endpoint rather than discovering every backend service. An API gateway centralizes cross-cutting policies while allowing services to evolve independently.
### Actors
1. **API consumer:** Sends authorized requests.
2. **Service developer:** Publishes routes and API versions.
3. **Platform operator:** Maintains gateway capacity and policies.
4. **Administrator:** Approves configuration and access.
### Core user journey
1. Client calls `GET /api/v1/catalog` over HTTPS.
2. Gateway matches a configured route, validates credentials and applies quota.
3. Gateway forwards to a healthy catalog service and returns its response.
4. Operator reviews latency and errors.
### Problem boundaries
API traffic management, not business logic, identity issuance or a general-purpose forward proxy.

---
## 02 — Requirements
### Functional requirements
| ID | Requirement | Priority |
| --- | --- | --- |
| FR-01 | Match configured host, method and path | Must |
| FR-02 | Terminate TLS | Must |
| FR-03 | Validate credentials | Must |
| FR-04 | Enforce route authorization | Must |
| FR-05 | Apply per-client quotas | Must |
| FR-06 | Forward to healthy configured upstreams | Must |
| FR-07 | Enforce request deadlines and size limits | Must |
| FR-08 | Publish versioned route configuration | Must |
| FR-09 | Expose metrics and request IDs | Must |
| FR-10 | Cache eligible public GET responses | Could |
| FR-11 | Support safe configuration rollback | Should |
### Non-functional requirements
| ID | Requirement | Initial target / policy |
| --- | --- | --- |
| NFR-01 | Latency | p95 gateway overhead < 20 ms under baseline; excludes upstream time |
| NFR-02 | Availability | 99.9% monthly gateway availability objective |
| NFR-03 | Durability | Persist approved configuration before publication |
| NFR-04 | Consistency | Atomic per-instance snapshots; bounded propagation lag |
| NFR-05 | Scalability | Scale gateway and upstream pools independently |
| NFR-06 | Security | Default deny; TLS; tenant-scoped policy |
| NFR-07 | Observability | Latency, errors, quota decisions and config version |
| NFR-08 | Maintainability | Versioned policies and separate control/data planes |
### Out of scope for the initial version
- Billing, developer marketplace, GraphQL federation and global strict quotas.

---
## 03 — Scale
### Workload assumptions
| Metric | Illustrative value | Reasoning |
| --- | --- | --- |
| Requests/day | 86.4 million | 1,000 × 86,400 |
| Average RPS | 1,000 | Teaching workload |
| Peak RPS | 3,000 | 3× average |
| Average request size | 1 KB | Simplified decimal units |
| Average response size | 4 KB | Simplified decimal units |
| Client-facing traffic | 432 GB/day | 86.4 million × 5 KB |
| Peak client-facing bandwidth | 15 MB/sec | 3,000 × 5 KB |
| Gateway overhead target | p95 < 20 ms | Excludes upstream latency |
| Availability target | 99.9% | Illustrative objective |
### Capacity notes
- Three instances at 500 RPS each cover average traffic, not peak. At least six are needed at peak before headroom or failures.
- At 3,000 RPS and 100 ms average upstream duration, about 300 concurrent upstream requests are expected without cache.
### Questions for a production sizing exercise
- What are route mix, payload sizes, connection counts, upstream latency, quota scope and regional traffic?

---
## 04 — Naive Architecture
### Simplest viable design
One gateway process holds static route configuration, validates credentials, applies local limits and forwards requests to backend services.
**Components:** Client, single gateway, configured upstream services.
**Request flow:** Match route, authenticate, authorize, enforce quota, forward and return response.
### Naive architecture diagram
See `architecture.png`.
### Why begin here?
It demonstrates the gateway contract without distributed configuration or quota coordination.

---
## 05 — Problems
| Problem | Why it occurs | User/system impact | Likely improvement |
| --- | --- | --- | --- |
| Single gateway | One process | Outage | Replicate |
| Capacity limit | Shared CPU/connections | Latency | Scale horizontally |
| Quota inconsistency | Local counters | Uneven enforcement | Shared quota service |
| Slow upstream | Long requests | Resource exhaustion | Deadlines and isolation |
| Config drift | Manual updates | Inconsistent routing | Versioned snapshots |
| Repeated reads | Same public GET | Upstream load | Policy-controlled cache |
| Unsafe retries | Repeated writes | Duplicate effects | Retry only safe/idempotent operations |
| Poor visibility | Missing telemetry | Slow diagnosis | Metrics and traces |

---
## 06 — Architecture Evolution
These are proposed learning stages, not vendor history.
### V1 — Single gateway
- Static routes and local policies; see `architecture.png`.
### V2 — Replicated gateway behind a load balancer
- CMP-001 routes to healthy stateless gateway replicas; see `architecture-v2.png`.
### V3 — Shared quota and cache
- Introduce atomic quota decisions and optional public-response caching; see `architecture-v3.png`.
### V4 — Separate control and data planes
- Versioned configuration store and control plane distribute validated snapshots; gateways enforce policies and forward independently. See `architecture-evolved.png`.
### Global evolution
- Regional gateway pools and upstreams; distribute signed configuration, allocate regional quotas and define failover and propagation limits. See `architecture-global.png`.

---
## 07 — Components
| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Routes to healthy gateway replicas | V2 onward |
| TBD | Client | Sends API requests | Yes |
| TBD | API Gateway | Enforces policies and routes | Yes |
| TBD | Identity Provider | Issues credentials and publishes verification keys | Yes |
| TBD | Upstream Services | Execute business operations | Yes |
| TBD | Quota Store | Atomic per-client limit decisions | V3 |
| TBD | Response Cache | Stores eligible public responses | V3 optional |
| TBD | Control Plane | Validates and distributes configuration | V4 |
| TBD | Configuration Store | Persists approved versions | V4 |
| TBD | Observability | Collects metrics, logs and traces | Recommended |
**Component ID policy:** CMP-001 is the only confirmed ID; others remain TBD.
### Component interactions
- Client → Load Balancer → Gateway → Upstream Services.
- Gateway verifies credentials using trusted identity keys and consults quota store before cache lookup.
- Control Plane reads approved configuration and publishes snapshots to gateways.

---
## 08 — Data Flow
### Flow A: API request
1. Load balancer selects a healthy gateway; gateway matches host, method and path.
2. Verify credential signature, issuer, audience and expiry; enforce route authorization and quota.
3. For explicitly public cacheable GET, check cache; otherwise forward to configured healthy upstream with deadline.
4. Return response and record sanitized telemetry.
### Flow B: Configuration publication
1. Authorized operator submits route and policy version.
2. Control plane validates and commits approved snapshot; distribute to replicas.
3. Replicas atomically activate valid newer version and acknowledge; rollback by publishing a new version containing previous settings.
### Flow C: Quota enforcement
1. Derive trusted client and route key; atomically consume a token from shared quota.
2. Reject exhausted quota with 429 and appropriate retry information; do not forward.
### Example route record
```json
{"route_id":"catalog_v1","method":"GET","path":"/api/v1/catalog","upstream":"catalog-service","required_scope":"catalog:read","timeout_ms":1000,"cache_enabled":false,"version":1}
```
### Example request event
```json
{"request_id":"req_001","route_id":"catalog_v1","status":200,"duration_ms":85,"config_version":1}
```
See `data-flow.md` for sequence diagrams.

---
## 09 — Failure Scenarios
| Failure | Detection | Immediate effect | Mitigation / recovery |
| --- | --- | --- | --- |
| Gateway crash | Health check | Lost in-flight requests | Remove unhealthy replica |
| Upstream unavailable | Health/errors | 503 or 502 | Isolate and recover pool |
| Upstream timeout | Deadline | 504 | Bound concurrency |
| Quota store outage | Dependency errors | Protected requests rejected | Fail closed with 503 |
| Cache outage | Cache errors | More upstream traffic | Bypass with capacity protection |
| Control plane outage | Publish failures | No new config | Use last valid snapshot within policy |
| Invalid config | Validation | Update rejected | Keep previous snapshot |
| Region outage | Regional health | Regional disruption | Route to healthy region with capacity |

---
## 10 — Scaling
### 10× traffic
- Add gateway replicas and upstream capacity; measure TLS, connections and quota latency.
### 100× traffic
- Partition quotas by tenant, isolate route pools, use bounded cache and independent configuration distribution.
### 1,000× traffic
- Deploy regional pools; allocate quota budgets, define failover and config convergence.
### Scaling decision checklist
- Identify CPU, connection, quota, cache or upstream bottlenecks; measure saturation and cost.

---
## 11 — Trade-offs
| Decision | Option A | Option B | Considerations |
| --- | --- | --- | --- |
| Quota | Local | Shared | Speed vs. coordinated limits |
| Configuration | Static | Dynamic snapshots | Simplicity vs. agility |
| Authentication | Local token validation | Remote introspection | Latency vs. revocation freshness |
| Cache | Disabled | Public GET cache | Correctness vs. upstream savings |
| Routing | Round robin | Least connections | Simplicity vs. uneven duration |
| Regions | Single | Multiple | Cost vs. availability |
### Required decision record format
- Context, options, chosen option, justification, gains, sacrifices and revisit conditions.

---
## 12 — Cost
No provider or region specified; use a cost model rather than invented prices.
### Cost categories
| Cost item | Main cost driver | Measurement needed |
| --- | --- | --- |
| Load balancer | Hours and bytes | RPS and connections |
| Gateway compute | Instance hours | CPU and memory |
| Quota store | Operations | Decisions/sec |
| Cache | Memory | Working set |
| Configuration store | Storage and writes | Versions |
| Network | Transfer | Bytes by region |
| Observability | Telemetry | Logs and traces |
### Estimation worksheet
- Monthly compute = instance-hours × provider rate; add storage, requests, transfer, backups and telemetry. Record region, date, currency, pricing URL and assumptions.

---
## 13 — Security
### Threats and controls
| Threat | Mitigation |
| --- | --- |
| Unauthorized access | Validate credentials and enforce route scopes |
| Tenant leakage | Tenant-scoped authorization; no private response caching |
| Excessive traffic | Quotas and bounded request sizes |
| Untrusted upstream | Allow only administrator-approved destinations |
| Secret exposure | Managed secrets and redacted telemetry |
| Configuration tampering | Restricted admin access, validation and audit |
| Unsafe request forwarding | Standards-compliant HTTP parsing and trusted header handling |
### Validation rules
- Default deny unknown routes; enforce TLS, request limits and approved upstreams. Backend services enforce resource-level authorization. Never log credentials or sensitive payloads.

---
## 14 — Observability
### Metrics
| Metric | Type | Why it matters |
| --- | --- | --- |
| gateway_requests_total | Counter | Traffic |
| gateway_overhead_ms | Histogram | Gateway performance |
| upstream_latency_ms | Histogram | Backend performance |
| gateway_errors_total | Counter | Reliability |
| quota_rejections_total | Counter | Policy pressure |
| cache_hit_ratio | Gauge | Cache effectiveness |
| healthy_gateways | Gauge | Capacity |
| config_version_lag | Gauge | Propagation health |
### Logs and traces
- Correlate request ID and route; redact credentials and avoid unbounded metric labels.
### Example alerts
- High 5xx rate, overhead breach, low healthy capacity, quota dependency failure or stale configuration.

---
## 15 — Simulation
### Learning objective
Observe gateway capacity, quota enforcement, caching and upstream saturation.
### Initial simulation controls
| Control | Default | Suggested range |
| --- | --- | --- |
| Incoming RPS | 1000 | 10–100000 |
| Gateway instances | 3 | 1–20 |
| Capacity per instance | 500 | 100–5000 |
| Failed instances | 0 | 0–gateway_instances |
| Aggregate quota RPS | 1200 | 0–100000 |
| Quota store available | true | On / Off |
| Cache enabled | false | On / Off |
| Eligible cache hit ratio (%) | 30 | 0–100 |
| Upstream capacity RPS | 2000 | 0–100000 |
| Upstream latency (ms) | 80 | 1–1000 |
| Gateway overhead (ms) | 10 | 1–100 |
### Expected outputs
- Offered, admitted, rejected, successful requests, cache hits, upstream load and modeled latency.
### Experiments
- Increase traffic 10×, fail replicas, exhaust quota, enable cache, reduce upstream capacity and disable quota store.
### Simulation model warning
- See `simulation-spec.md`; deterministic teaching model, not production measurements.

---
## 16 — Design Challenge
### Challenge: Build an API gateway
**Requirements:** Route requests, authenticate and authorize, enforce quotas, forward safely and publish configuration.
**Constraints for the exercise:** Start with one gateway, add three replicas, introduce shared quotas, fail one replica and increase traffic 10×.
**Learner deliverables:** Requirements, architecture, API contracts, data model, sequence flows, capacity estimates, failure analysis, trade-offs, security and observability.
**Evaluation prompts:** How do quotas remain consistent? What happens when an upstream fails? How do you roll back configuration?

---
## 17 — AI Interview
### Interviewer question bank
| ID | Difficulty | Question | What a strong answer should cover |
| --- | --- | --- | --- |
| Q01 | Beginner | What are core gateway responsibilities? | Routing, access, quota, telemetry |
| Q02 | Beginner | How does routing work? | Host, method, path |
| Q03 | Beginner | Why use a load balancer? | Healthy replicas |
| Q04 | Intermediate | How do you enforce quotas? | Atomic decisions |
| Q05 | Intermediate | When is caching safe? | Explicit public eligibility |
| Q06 | Intermediate | How do you handle slow upstreams? | Deadlines and isolation |
| Q07 | Intermediate | How do you validate credentials? | Signature and claims |
| Q08 | Intermediate | What if quota storage fails? | Fail closed |
| Q09 | Advanced | How do you publish configuration? | Validated snapshots |
| Q10 | Advanced | How do regional quotas work? | Allocated budgets |
| Q11 | Advanced | How do you protect tenants? | Scoped access |
| Q12 | Advanced | How do you estimate cost? | Traffic, compute, dependencies |
### Follow-up evaluation dimensions
- Clarify assumptions, explain flows, identify failures, justify trade-offs and avoid unsupported vendor claims.

---
## 18 — Architecture Score

### Proposed evaluation rubric
| Dimension | Weight | Evidence |
|---|---|---|
| Requirements completeness | 10 | User journeys |
| Scalability | 15 | Capacity and bottlenecks |
| Reliability | 15 | Failover and recovery |
| Latency and performance | 10 | Gateway and upstream latency |
| Data design and consistency | 15 | Configuration and quota consistency |
| Security | 10 | Authentication and authorization |
| Cost efficiency | 10 | Cost assumptions |
| Trade-off justification | 10 | Alternatives |
| Observability and operations | 5 | Metrics and alerts |
| **Total** | **100** |  |
### Scoring guidance
- 0: Missing or incorrect; 1: Mentioned; 2: Partial; 3: Correct and justified; 4: Detailed and measurable.
Weighted score = sum of (dimension score / 4 × dimension weight). Human review and evidence required.
