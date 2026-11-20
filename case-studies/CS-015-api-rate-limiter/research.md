# CS-015 — Design an API Rate Limiter

> **Content status:** Educational reference design; not a claim about a vendor's private architecture.
>
> **Source alignment:** Category A: Web & Internet Systems; supplied 18-component template. CS-010 follows the local index; official catalogue number unverified.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-010 |
| System name | API Rate Limiter |
| Real-world reference | Envoy Rate Limit Service / Redis (reference technologies) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 10 (local index; catalogue unverified) |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | Enforces request budgets per authenticated client and route to protect API capacity and fairness. |
| Primary users | API consumers, developers, platform operators, administrators |
| Key concepts | Token bucket, fixed window, sliding window, atomic counters, distributed state, HTTP 429, load balancing, observability |
| Related component IDs | CMP-001 (Load Balancer); additional components TBD |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-19 |

### Scope and assumptions
- Protect authorized HTTP APIs; one token per request; single region initially.
- Default algorithm is token bucket; trusted tenant and route identifiers determine the policy key.
- Denied requests never reach the protected upstream. Dependency failure returns 503 for protected routes.
- Traffic and targets are illustrative, not vendor measurements.

---
## 01 — Story
Shared APIs need fair access and predictable capacity. A rate limiter decides whether a request can proceed before expensive backend work begins.
### Actors
1. **API consumer:** Sends requests and respects retry guidance.
2. **Service developer:** Defines route budgets.
3. **Platform operator:** Monitors decisions and capacity.
4. **Administrator:** Approves policy changes.
### Core user journey
1. Client sends an authenticated API request.
2. Gateway identifies the tenant and route.
3. Limiter evaluates the applicable token bucket atomically.
4. Allowed requests reach the upstream; exhausted budgets receive HTTP 429.
### Problem boundaries
Request admission, not authentication issuance, billing, business authorization or guaranteed backend capacity.

---
## 02 — Requirements
### Functional requirements
| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Identify trusted tenant and route | Must |
| FR-02 | Apply configured refill rate and burst capacity | Must |
| FR-03 | Atomically consume a token when available | Must |
| FR-04 | Forward only allowed requests | Must |
| FR-05 | Return 429 for exhausted budgets | Must |
| FR-06 | Provide Retry-After when calculable | Should |
| FR-07 | Share state across replicas | Must |
| FR-08 | Publish versioned policies | Must |
| FR-09 | Report decision metrics | Must |
| FR-10 | Support fixed and sliding window comparisons | Could |
| FR-11 | Support policy rollback | Should |
### Non-functional requirements
| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Latency | p95 decision overhead under 10 ms in region |
| NFR-02 | Availability | 99.9% monthly decision service objective |
| NFR-03 | Durability | Persist approved policies; define state recovery |
| NFR-04 | Consistency | Atomic decisions per key; no silent reset on failover |
| NFR-05 | Scalability | Scale decision workers and state partitions independently |
| NFR-06 | Security | Trusted identity, tenant isolation, fail closed |
| NFR-07 | Observability | Decisions, errors, latency and state saturation |
| NFR-08 | Maintainability | Versioned policy and documented algorithms |
### Out of scope for the initial version
- Billing quotas, weighted request costs, strict global coordination and concurrency limiting.

---
## 03 — Scale
### Workload assumptions
| Metric | Illustrative value | Reasoning |
|---|---|---|
| Requests/day | 86.4 million | 1,000 × 86,400 |
| Average RPS | 1,000 | Teaching workload |
| Peak RPS | 5,000 | 5× average |
| Active bucket keys | 100,000 | Tenant-route pairs |
| State per bucket | 128 bytes | Simplified estimate |
| Raw bucket state | 12.8 MB | 100,000 × 128 bytes |
| Decision operations | 1 per admitted request | Atomic read-modify-write |
| Decision latency target | p95 < 10 ms | In-region overhead |
| Availability target | 99.9% | Illustrative objective |
### Capacity notes
- At 500 decisions/sec per worker, ten workers cover peak before headroom; eleven tolerate one failure at that rate.
- State-store capacity must cover peak decisions, including denials. Include key, allocator, replication and persistence overhead beyond raw bytes.
### Questions for a production sizing exercise
- How many active keys, what burst sizes, how skewed is demand, and what consistency is required?

---
## 04 — Naive Architecture
### Simplest viable design
One gateway uses an in-process token bucket map and forwards allowed requests to the protected API.
**Components:** Client, single gateway with local limiter, upstream API.
**Request flow:** Identify key, refill bucket, consume token if available, forward or reject.
### Naive architecture diagram
See `architecture.png`.
### Why begin here?
It demonstrates admission decisions before introducing shared state and distributed policies.

---
## 05 — Problems
| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single gateway | One process | Outage | Replicate |
| Independent buckets | Local state per replica | Inconsistent aggregate budget | Shared atomic state |
| Race conditions | Concurrent updates | Incorrect decisions | Atomic operation |
| Restart | Volatile state | Budget reset | Conservative recovery |
| Hot keys | Concentrated traffic | State saturation | Isolate heavy keys |
| Clock skew | Multiple time sources | Incorrect refill | Authoritative state clock |
| Policy drift | Manual updates | Inconsistent limits | Versioned snapshots |
| Poor visibility | Missing metrics | Slow diagnosis | Decision telemetry |

---
## 06 — Architecture Evolution
These are proposed learning stages, not vendor history.
### V1 — Single gateway with local token buckets
- In-process state and one gateway; see `architecture.png`.
### V2 — Replicated gateways behind a load balancer
- CMP-001 routes to healthy gateways; local limits remain approximate across replicas. See `architecture-v2.png`.
### V3 — Shared atomic rate limiter
- Gateways call decision workers backed by shared bucket state; serialize each key's updates. See `architecture-v3.png`.
### V4 — Separate policy control and decision planes
- Versioned policy store and control plane publish validated snapshots; partition state by key and collect telemetry. See `architecture-evolved.png`.
### Global evolution
- Allocate regional refill rates and burst budgets whose sums do not exceed global limits; do not duplicate budgets during failover. See `architecture-global.png`.

---
## 07 — Components
| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Routes to healthy gateways | V2 onward |
| TBD | Client | Sends API requests | Yes |
| TBD | API Gateway | Authenticates and enforces decisions | Yes |
| TBD | Rate Limiter | Evaluates admission policy | Yes |
| TBD | Bucket State Store | Atomic token updates per key | V3 |
| TBD | Protected API | Executes allowed requests | Yes |
| TBD | Control Plane | Validates and distributes policies | V4 |
| TBD | Policy Store | Persists approved versions | V4 |
| TBD | Observability | Collects metrics and logs | Recommended |
**Component ID policy:** CMP-001 is the only confirmed ID; others remain TBD.
### Component interactions
- Client → Load Balancer → Gateway → Protected API (allowed only).
- Gateway → Rate Limiter → Bucket State Store; decision returns to gateway.
- Control plane distributes policies to decision workers; telemetry is asynchronous.

---
## 08 — Data Flow
### Flow A: API request
1. Gateway authenticates and authorizes the request; derives trusted tenant-route key.
2. Limiter selects policy and atomically refills and consumes bucket state.
3. Allow forwards to upstream; denial returns 429 with retry guidance; dependency failure returns 503.
### Flow B: Configuration publication
1. Authorized operator submits policy; validate positive refill rate and burst capacity of at least one.
2. Commit version and publish to workers. Apply updates atomically per key, clamp existing tokens to new capacity and avoid granting a fresh burst.
3. Acknowledge version; rollback uses a new version with prior settings.
### Flow C: Quota enforcement
1. For capacity B, rate r and elapsed time Δt, refill T = min(B, T + r × max(0, Δt)).
2. If T ≥ 1, subtract one and allow; otherwise deny. Persist tokens and timestamp atomically using authoritative time.
3. Retry-After = ceil((1 − T)/r) seconds for a denied request, an estimate rather than a reservation.
### Example policy record
```json
{"policy_id":"catalog_v1","refill_per_sec":100,"burst_capacity":200,"version":1}
```
### Example decision event
```json
{"request_id":"req_001","policy_id":"catalog_v1","decision":"deny","status":429,"policy_version":1}
```
See `data-flow.md` for sequence diagrams.

---
## 09 — Failure Scenarios
| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| Gateway crash | Health check | In-flight requests lost | Remove replica |
| Decision worker crash | Health check | Decision errors | Route to healthy worker |
| State store unavailable | Timeout | Protected requests rejected | Fail closed with 503 |
| State lost | Recovery checks | Budget uncertain | Restore or initialize conservatively |
| Hot partition | Latency and utilization | Slow decisions | Isolate workload and scale |
| Control plane outage | Publish errors | No new policy | Last valid policy within freshness bound |
| Invalid policy | Validation | Update rejected | Keep previous version |
| Region outage | Regional health | Regional disruption | Fail over only with reserved budget |

---
## 10 — Scaling
### 10× traffic
- Scale gateways, decision workers and state capacity; measure atomic operation latency.
### 100× traffic
- Partition by tenant-route key, isolate hot tenants and bound state lifetime. A single strict bucket remains serialized.
### 1,000× traffic
- Allocate regional budgets and deploy regional state; coordinate budget transfers before reuse.
### Scaling decision checklist
- Identify CPU, network, hot key, state memory or policy propagation bottlenecks; measure cost and correctness.

---
## 11 — Trade-offs
| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| Algorithm | Fixed window | Token bucket | Simple counters vs. controlled bursts |
| Window accuracy | Sliding log | Sliding counter | Exact history vs. approximate lower memory |
| State | Local | Shared | Low latency vs. coordinated enforcement |
| Failure policy | Fail open | Fail closed | Availability vs. protection; choose closed here |
| Configuration | Static | Versioned | Simplicity vs. controlled updates |
| Regions | Global coordinator | Regional allocation | Strict shared budget vs. latency and unused capacity |
### Required decision record format
- Context, options, chosen option, justification, gains, sacrifices and revisit conditions.

---
## 12 — Cost
No provider or region specified; use a cost model rather than invented prices.
### Cost categories
| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Load balancer | Hours and bytes | RPS and connections |
| Gateway compute | Instance hours | CPU and memory |
| Decision workers | Instance hours | Decisions/sec |
| State store | Operations and memory | Active keys, peak operations |
| Policy store | Storage and writes | Versions |
| Network | Transfer | Cross-region bytes |
| Observability | Telemetry | Metrics and logs |
### Estimation worksheet
- Monthly compute = instance-hours × provider rate; add state operations, storage, replication, transfer and telemetry. Record region, date, currency, pricing URL and assumptions.

---
## 13 — Security
### Threats and controls
| Threat | Mitigation |
|---|---|
| Untrusted identity | Authenticate before deriving tenant key |
| Tenant leakage | Tenant-scoped state and policy access |
| Excessive traffic | Bounded queues, edge protection and admission limits |
| Policy tampering | Restricted admin access and audit |
| State manipulation | Private authenticated state access |
| Secret exposure | Managed secrets and redacted telemetry |
| Unbounded key growth | Validated keys and safe expiry |
### Validation rules
- Validate trusted keys, positive rates, burst capacity and policy versions. Use TLS, least privilege and bounded deadlines. Never expose credentials in logs.
- Expire inactive buckets only when they would be fully refilled; conservative recovery for uncertain state.

---
## 14 — Observability
### Metrics
| Metric | Type | Why it matters |
|---|---|---|
| limiter_requests_total | Counter | Offered decisions |
| limiter_allowed_total | Counter | Admitted traffic |
| limiter_denied_total | Counter | Quota pressure |
| limiter_errors_total | Counter | Dependency failures |
| decision_latency_ms | Histogram | Decision performance |
| state_operation_latency_ms | Histogram | State bottleneck |
| active_bucket_count | Gauge | Memory demand |
| policy_version_lag | Gauge | Configuration freshness |
### Logs and traces
- Correlate request ID and policy version; avoid raw credentials and unbounded tenant labels.
### Example alerts
- High 503 rate, decision latency breach, state saturation, low healthy capacity or stale policy.

---
## 15 — Simulation
### Learning objective
Observe refill, bursts, admission capacity and shared-state failures.
### Initial simulation controls
| Control | Default | Suggested range |
|---|---|---|
| Incoming RPS | 1000 | 10–100000 |
| Decision instances | 3 | 1–20 |
| Capacity per instance | 500 | 100–5000 |
| Failed instances | 0 | 0–decision_instances |
| Refill tokens/sec | 800 | 1–100000 |
| Burst capacity | 1200 | 1–100000 |
| State capacity/sec | 2000 | 0–100000 |
| State available | true | On / Off |
| State latency (ms) | 3 | 1–1000 |
| Base overhead (ms) | 2 | 1–100 |
### Expected outputs
- Offered, allowed, 429, 503, remaining tokens, utilization and modeled latency.
### Experiments
- Increase traffic, exhaust burst, fail workers, disable state, reduce state capacity and observe refill during idle periods.
### Simulation model warning
- See `simulation-spec.md`; one-key deterministic tick model, not production benchmarks.

---
## 16 — Design Challenge
### Challenge: Build an API rate limiter
**Requirements:** Enforce token budgets, return correct status codes, coordinate shared state and publish policies.
**Constraints for the exercise:** Start with one gateway, add three replicas, introduce atomic shared state, fail one worker and increase traffic 10×.
**Learner deliverables:** Requirements, architecture, API contracts, data model, sequence flows, capacity estimates, failure analysis, trade-offs, security and observability.
**Evaluation prompts:** How do atomic updates preserve limits? How do you recover state? How do regional budgets remain bounded?

---
## 17 — AI Interview
### Interviewer question bank
| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | Why rate limit APIs? | Fairness and capacity |
| Q02 | Beginner | What is HTTP 429? | Budget exhaustion and retry guidance |
| Q03 | Beginner | How does a token bucket work? | Refill, capacity, consumption |
| Q04 | Intermediate | Compare fixed and sliding windows | Boundary effects, accuracy, memory |
| Q05 | Intermediate | Why use shared state? | Coordinated replica decisions |
| Q06 | Intermediate | How do atomic updates help? | Serialize per-key decisions |
| Q07 | Intermediate | How do you handle clock skew? | Authoritative time |
| Q08 | Intermediate | What if state is unavailable? | Explicit fail-closed policy |
| Q09 | Advanced | How do you partition state? | Key distribution and hot keys |
| Q10 | Advanced | How do regional budgets work? | Allocation and safe transfer |
| Q11 | Advanced | How do you update policies? | Versioning and conservative migration |
| Q12 | Advanced | How do you estimate cost? | Operations, memory, compute, network |
### Follow-up evaluation dimensions
- Clarify assumptions, explain algorithms, identify failures, justify trade-offs and avoid unsupported vendor claims.

---
## 18 — Architecture Score
### Proposed evaluation rubric
| Dimension | Weight | Evidence |
|---|---|---|
| Requirements completeness | 10 | User journeys |
| Scalability | 15 | Capacity and bottlenecks |
| Reliability | 15 | Failover and recovery |
| Latency and performance | 10 | Decision overhead |
| Data design and consistency | 15 | Atomic state and policy consistency |
| Security | 10 | Trusted identity and access control |
| Cost efficiency | 10 | Cost assumptions |
| Trade-off justification | 10 | Alternatives |
| Observability and operations | 5 | Metrics and alerts |
| **Total** | **100** | |
### Scoring guidance
- 0: Missing or incorrect; 1: Mentioned; 2: Partial; 3: Correct and justified; 4: Detailed and measurable.
Weighted score = sum of (dimension score / 4 × dimension weight). Human review and evidence required.
