# CS-WebHosting — Design a Web Hosting Platform

> **Content status:** Educational reference design. This document is a proposed design for a multi-tenant web hosting service. It does not claim to describe any vendor’s private production architecture.
>
> **Source alignment:** This adaptation uses Category A: Web & Internet Systems and the supplied URL-shortener template; the Web Hosting Platform catalogue number is not verified. The required structure follows the 18-component case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-WebHosting |
| System name | Web Hosting Platform |
| Real-world reference | Netlify / Vercel (reference products, not an asserted architecture source) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | To be verified |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | A platform that deploys websites, manages domains and TLS, and serves website traffic reliably. |
| Primary users | Site owners, developers, visitors, administrators |
| Key concepts | HTTP, DNS, API design, deployment orchestration, object storage, CDN, tenant isolation, rate limiting, analytics, observability |
| Related component IDs | CMP-001 (Load Balancer); additional components can be added to the component index as they are documented |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-15 |

### Scope and assumptions

- Users deploy static websites from uploaded artifacts; managed dynamic runtimes are an extension.
- Each site receives a platform subdomain and may attach a verified custom domain.
- HTTPS, deployment status, rollback and tenant isolation are required.
- Initial deployment is single-region; global delivery is an evolution stage.
- All traffic figures are illustrative, not vendor measurements.

---

## 01 — Story

### Actors

Developers need a reliable way to publish websites without managing individual servers. The platform stores versioned artifacts and routes visitors to the active deployment.

1. **Site owner:** Manages sites, domains and deployments.
2. **Developer:** Uploads artifacts and monitors deployment status.
3. **Visitor:** Requests hosted pages over HTTPS.
4. **Administrator:** Handles operations, quotas and abuse reports.

### Core user journey

1. Owner creates a site and receives `demo.host.example`.
2. Developer uploads a static website bundle.
3. Platform validates and stores immutable artifacts.
4. Deployment worker verifies readiness and activates the version atomically.
5. Visitor resolves the hostname and requests a page over HTTPS.
6. CDN or origin serves the active version.
7. Owner can roll back to a previous healthy version.

### Problem boundaries

Focus on static hosting, deployment lifecycle, custom domains, TLS and delivery. Dynamic application hosting is an optional extension, not a requirement of the baseline.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Create and manage a site | Must |
| FR-02 | Upload and validate website artifacts | Must |
| FR-03 | Deploy and serve website content | Must |
| FR-04 | Reject invalid or oversized uploads | Must |
| FR-05 | Maintain unique site hostnames | Must |
| FR-06 | Attach and verify custom domains | Should |
| FR-07 | View deployment status and roll back | Should |
| FR-08 | Provision and renew TLS certificates | Should |
| FR-09 | Collect access and deployment events | Should |
| FR-10 | Provide usage statistics | Could |
| FR-11 | Enforce tenant quotas and abuse policy | Should |

### Non-functional requirements

| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Serving latency | p95 under 100 ms at edge, excluding client network |
| NFR-02 | Availability | 99.9% monthly serving target |
| NFR-03 | Durability | Committed artifacts and metadata survive process restart |
| NFR-04 | Consistency | Atomic active-version switch; bounded edge propagation |
| NFR-05 | Scalability | Serving and deployment scale independently |
| NFR-06 | Security | Tenant isolation, HTTPS, authorization and quotas |
| NFR-07 | Observability | Track latency, errors, deployment lag and health |
| NFR-08 | Maintainability | Separate control plane and serving plane |

### Out of scope for the initial version

- Arbitrary server-side code execution, email hosting, billing, and active-active control-plane writes.

---

## 03 — Scale

All values are illustrative teaching assumptions.

### Workload assumptions

| Metric | Illustrative value | Reasoning |
|---|---|---|
| Registered users | 100,000 | Planning scenario |
| Hosted sites | 10,000 | Tenant workload |
| Deployments per day | 1,000 | Write workload |
| Page requests per day | 10,000,000 | Read-heavy workload |
| Average serving RPS | ~116 | 10,000,000 / 86,400 |
| Peak serving RPS | 1,000 | 8.6× average |
| Average deployment size | 20 MB | Static bundle |
| Artifact growth | 20 GB/day | Before retention and replication |
| Average response size | 100 KB | 1 TB/day delivery |
| Availability target | 99.9% | Illustrative |

### Capacity notes

- At 90% CDN hit ratio, origin receives about 1 million requests/day.
- Thirty days of new artifacts require about 600 GB before replication and cleanup.

### Questions for a production sizing exercise

- How large are bundles and responses? What is the cache hit ratio? How many versions are retained? What are peak deployment concurrency and regional traffic patterns?

---

## 04 — Naive Architecture

### Simplest viable design

One web server handles uploads, hostname routing and static file delivery. A local database stores site metadata and a persistent filesystem stores versioned files.

**Components**
- Client/browser
- Single hosting server
- Metadata database and persistent file storage

**Deployment flow**
1. Owner uploads a bundle.
2. Server validates ownership and files.
3. Server stores a version and atomically switches the active pointer.

**Serving flow**
1. Browser requests a hostname and path.
2. Server resolves the active site version.
3. Server returns the corresponding file or 404.

### Naive architecture diagram

See `architecture.png`.

### Why begin here?

The baseline demonstrates deployment and delivery before introducing distributed storage and edge caching.

---

## 05 — Problems

| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single-server bottleneck | Shared compute | Slow delivery | Replicate servers |
| Single point of failure | Local dependencies | Outage | Redundancy |
| Local file storage | Files tied to host | Inconsistent replicas | Object storage |
| Hot sites | Uneven traffic | Origin saturation | CDN |
| Deployment contention | Uploads share serving resources | Latency spikes | Separate workers |
| Partial deployment | Files updated in place | Broken pages | Immutable versions |
| Tenant interference | Shared resources | Noisy neighbors | Quotas and isolation |
| Stale content | Cached old versions | Delayed updates | Versioned keys and bounded TTL |
| TLS expiry | Renewal failure | HTTPS errors | Automated renewal |
| Poor visibility | Missing telemetry | Slow recovery | Metrics and alerts |

---

## 06 — Architecture Evolution

These are proposed learning stages, not vendor history.

### V1 — Single hosting server
- Local metadata and persistent files; simple but limited. See `architecture.png`.

### V2 — Replicated serving behind a load balancer
- CMP-001 routes to healthy stateless servers; shared object storage and metadata database remove local file dependency. See `architecture-v2.png`.

### V3 — CDN and cache
- CDN caches public immutable assets; origin resolves active versions. HTML uses bounded TTL; private content bypasses shared caches. See `architecture-v3.png`.

### V4 — Separate deployment and serving planes
- Management API queues deployments; isolated workers validate and publish artifacts; atomic metadata activation and rollback preserve serving availability. See `architecture-evolved.png`.

### Global evolution
- Global DNS/CDN, regional origins, replicated artifacts and read metadata. A single authoritative control-plane writer avoids conflicting activation; failover requires fencing and verified artifact readiness. See `architecture-global.png`.

---

## 07 — Components

| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Routes to healthy origins | V2 onward |
| TBD | Client | Uploads and visits sites | Yes |
| TBD | DNS | Resolves hostnames | Yes |
| TBD | Management API | Authorizes site and deployment operations | Yes |
| TBD | Serving Origin | Resolves hostname and serves files | Yes |
| TBD | Metadata Database | Stores sites, domains and active versions | Yes |
| TBD | Object Storage | Stores immutable artifacts | V2 |
| TBD | CDN | Caches public assets | V3 |
| TBD | Message Broker | Queues deployment jobs | V4 |
| TBD | Deployment Worker | Validates and publishes versions | V4 |
| TBD | TLS Manager | Issues and renews certificates | Recommended |
| TBD | Rate Limiter | Enforces quotas | Recommended |
| TBD | Monitoring / Observability | Collects telemetry | Recommended |

**Component ID policy:** CMP-001 is the only confirmed ID; others remain TBD.

### Component interactions

- Visitor → DNS → CDN → Load Balancer → Origin → metadata/object storage.
- Owner → Management API → Broker → Deployment Worker → object storage → metadata activation.
- TLS manager verifies domains and provisions certificates.

---

## 08 — Data Flow

### Flow A: Create and deploy a site
1. `POST /api/v1/sites` creates a unique hostname.
2. `POST /api/v1/sites/{id}/deployments` accepts an authorized artifact upload and returns 202 with deployment ID.
3. Worker validates bundle and stores immutable files.
4. Worker verifies readiness and atomically activates the version; status becomes ready.
5. Owner polls deployment status; failed jobs leave the previous version active.

### Flow B: Serve a website
1. Browser resolves DNS and establishes HTTPS.
2. CDN serves eligible cached assets or forwards to a healthy origin.
3. Origin resolves hostname to active version and retrieves the requested file.
4. Return 200 with appropriate content type or 404; cache public content according to policy.

### Flow C: Domain management and rollback
1. Owner submits custom domain; platform verifies ownership and uniqueness.
2. TLS manager provisions certificate before enabling HTTPS routing.
3. Rollback selects a previously ready version and atomically changes the active pointer; edge HTML is invalidated or expires within policy.

### Example site record
```json
{"site_id":"site_123","owner_id":"user_123","hostname":"demo.host.example","active_version":"v2","status":"active"}
```

### Example deployment event
```json
{"event_id":"evt_001","site_id":"site_123","deployment_id":"v2","status":"ready"}
```

See `data-flow.md` for sequence diagrams. Avoid logging secrets or uploaded content.

---

## 09 — Failure Scenarios

| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| Origin crash | Health checks | Reduced capacity | Remove unhealthy target |
| All origins unavailable | Synthetic probes | Cache misses fail | Restore capacity |
| Metadata unavailable | Query errors | Uncached routing fails | Controlled 503; failover |
| Object storage unavailable | Read errors | Uncached assets fail | Replicated storage and retries |
| CDN unavailable | Edge errors | Delivery degraded | Protected origin fallback |
| Stale edge HTML | Version probes | Old page | Purge or bounded TTL |
| Deployment fails | Job status | New version unavailable | Keep old active version |
| Broker unavailable | Queue errors | New jobs delayed | Durable job record and retry |
| Worker crash | Lease timeout | Job delayed | Idempotent retry |
| Traffic spike | Latency and RPS | Overload | Scale and rate limit |
| Certificate expiry | Expiry monitor | HTTPS failure | Renew and alert |
| Region outage | Regional probes | Regional failures | Route to ready replica |

Document trigger, outcome, recovery and recovery metrics.

---

## 10 — Scaling

### 10× traffic
- Scale origins behind CMP-001; increase CDN capacity and deployment worker count independently.

### 100× traffic
- Partition metadata by site ID, maintain hostname lookup index, distribute object storage and queue workers; enforce per-tenant quotas.

### 1,000× traffic
- Add regional origins, replicated artifacts and global CDN; define RPO/RTO, failover fencing and cost limits.

### Scaling decision checklist
- Identify network, storage, CPU, cache or queue bottlenecks; measure hot sites, consistency needs and operational cost.

---

## 11 — Trade-offs

| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| Hosting model | Static | Dynamic runtime | Simplicity vs execution flexibility |
| Storage | Local disk | Object storage | Local simplicity vs shared durability |
| Delivery | Origin only | CDN | Freshness vs latency |
| Deployment | In-place | Immutable versions | Low storage vs atomic rollback |
| Jobs | Synchronous | Asynchronous | Immediate response vs resilience |
| Domains | Platform subdomain | Custom domain | Simplicity vs verification/TLS |
| Region | Single | Multi-region | Cost vs availability |

### Required decision record format

Document context, options, chosen option, rationale, gains, sacrifices and revisit conditions.

---

## 12 — Cost

No provider or region is specified; use a cost model rather than fabricated prices.

### Cost categories

| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Load balancer | Hours and bytes | Requests and connections |
| Origin compute | Instance hours | CPU and RPS |
| Metadata database | Storage and IOPS | Queries and backups |
| Object storage | GB-month and operations | Artifact size and retention |
| CDN | Requests and egress | Delivered bytes and hit ratio |
| Deployment workers | Compute minutes | Jobs and duration |
| Broker | Throughput and retention | Jobs/sec |
| Observability | Ingestion and retention | Telemetry volume |

### Estimation worksheet
- Compute = instances × hourly price × hours.
- Storage = retained GB × monthly rate + operations.
- CDN = delivered GB × egress rate + request charges.
- Add database, queue, TLS/domain, backup and monitoring charges. Record provider, date, region and currency.

---

## 13 — Security

### Threats and controls

| Threat | Mitigation |
|---|---|
| Unauthorized deployment | Authentication and site ownership checks |
| Cross-tenant access | Tenant-scoped authorization and storage policies |
| Unsafe uploads | Size limits, safe extraction and validation |
| Domain takeover | Ownership verification and unique binding |
| Secret exposure | Managed secrets and least privilege |
| Malicious hosted content | Abuse reporting and takedown policy |
| Denial of service | Quotas and rate limits |
| TLS failure | Automated renewal and expiry monitoring |
| Cache privacy leakage | Do not shared-cache private responses |
| Dynamic runtime risk | Isolated sandbox and resource limits if supported |

### Validation rules
- Reject invalid hostnames, unsafe paths and oversized bundles.
- Verify domain ownership before binding; enforce authorization on every management operation.
- Serve correct content types; isolate hosted sites from management cookies and origins.

---

## 14 — Observability

### Metrics

| Metric | Type | Why it matters |
|---|---|---|
| `serve_requests_total` | Counter | Traffic |
| `serve_latency_ms` | Histogram | Performance |
| `serve_errors_total` | Counter | Reliability |
| `deployment_total` | Counter | Job volume |
| `deployment_failures_total` | Counter | Deployment reliability |
| `cache_hit_ratio` | Gauge | CDN effectiveness |
| `storage_latency_ms` | Histogram | Origin dependency |
| `load_balancer_healthy_targets` | Gauge | Origin health |
| `deployment_queue_depth` | Gauge | Backlog |
| `certificate_days_remaining` | Gauge | TLS expiry |

### Logs and traces
- Correlate request and deployment IDs; trace CDN, origin, storage and job stages. Avoid secrets and unbounded metric labels.

### Example alerts
- High p95 or 5xx rate, low healthy targets, growing queue, storage errors or certificate nearing expiry.

---

## 15 — Simulation

### Learning objective

Observe CDN hits, origin capacity, storage latency and deployment backlog.

### Initial simulation controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| serve_rps | Incoming website RPS | Slider | 1000 | 10 | 100000 |
| deploy_rps | Deployment jobs/sec | Slider | 1 | 0 | 100 |
| app_instances | Origin instances | Stepper | 3 | 1 | 20 |
| instance_capacity_rps | Capacity per instance | Slider | 500 | 100 | 5000 |
| cache_enabled | Enable cache | Toggle | true | — | — |
| cache_hit_ratio | Cache hit ratio | Slider | 90 | 0 | 100 |
| db_read_capacity | Origin storage read capacity/sec | Slider | 2000 | 100 | 20000 |
| db_latency_ms | Origin storage read latency | Slider | 20 | 1 | 1000 |
| lb_algorithm | Routing algorithm | Select | round_robin | — | — |
| failed_instances | Failed origin instances | Stepper | 0 | 0 | app_instances |
| worker_capacity | Deployment worker capacity/jobs per sec | Slider | 2 | 1 | 100 |
| broker_available | Broker available | Toggle | true | — | — |

### Expected outputs

- Served requests, errors, origin utilization, cache hits/misses, storage load, estimated latency and deployment backlog.

### Experiments

1. Disable CDN cache.
2. Increase traffic 10×.
3. Fail an origin.
4. Increase storage latency.
5. Disable broker.
6. Increase deployment rate above worker capacity.

### Simulation model warning

See `simulation-spec.md`; estimates are not production measurements.

---

## 16 — Design Challenge

### Challenge: Build a web hosting platform

**Requirements**
- Create sites, upload artifacts, deploy, serve HTTPS, verify domains and roll back.

**Constraints for the exercise**
- Start with one server, evolve to three origins, add CDN, inject failure and increase traffic 10×.

**Learner deliverables**
1. Requirements.
2. Architecture diagrams.
3. API contracts.
4. Data model.
5. Deployment and serving flows.
6. Capacity estimates.
7. Failure analysis.
8. Trade-off records.
9. Security and observability plan.

**Evaluation prompts**
- How is deployment atomic? How are tenants isolated? How do cache and rollback interact? How does regional failover work?

---

## 17 — AI Interview

### Interviewer question bank

| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What are core hosting requirements? | Deployment, serving, domains, TLS |
| Q02 | Beginner | How do you identify sites? | Unique hostname and site ID |
| Q03 | Beginner | Which storage would you choose? | Metadata and immutable artifacts |
| Q04 | Intermediate | Why separate serving and deployment? | Independent scaling |
| Q05 | Intermediate | How does CDN caching work? | Versioned assets and TTL |
| Q06 | Intermediate | How handle hot sites? | CDN and origin protection |
| Q07 | Intermediate | How handle failed deployments? | Keep active version, retry |
| Q08 | Intermediate | What if an origin fails? | Health checks and routing |
| Q09 | Advanced | How partition metadata? | Site ID and hostname index |
| Q10 | Advanced | How support multiple regions? | Replication and fenced failover |
| Q11 | Advanced | How isolate tenants? | Authorization, quotas, runtime isolation |
| Q12 | Advanced | How estimate cost? | Compute, storage, egress and jobs |

### Follow-up evaluation dimensions

- Clarifies assumptions, explains flows and failures, justifies trade-offs, avoids unsupported vendor claims.

---

## 18 — Architecture Score

### Proposed evaluation rubric

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements completeness | 10 | Required user journeys and constraints |
| Scalability | 15 | Traffic estimates, bottleneck analysis, scaling plan |
| Reliability | 15 | Failure handling, redundancy, recovery |
| Latency and performance | 10 | Serving path, cache strategy, measured assumptions |
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
