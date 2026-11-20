# CS-016 — Design a CAPTCHA Service

> **Content status:** Educational reference design, not a claim about a vendor’s private architecture.
>
> **Source alignment:** Category A: Web & Internet Systems; supplied 18-component template. Local index CS-011; official catalogue number unverified.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-011 |
| System name | CAPTCHA Service |
| Real-world reference | reCAPTCHA / hCaptcha (reference products) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 11 (local index) |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | Issues and verifies human-verification challenges for protected websites. |
| Primary users | Website visitors, site developers, administrators |
| Key concepts | Challenge lifecycle, accessibility, expiration, verification, atomic state, load balancing |
| Related component IDs | CMP-001 (Load Balancer); additional components TBD |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-19 |

---

## 01 — Story

### Actors
- Visitor completes a verification step.
- Site backend requests a verification decision.
- Administrator manages site registration and policies.
### Core user journey
A visitor submits a protected form, completes an accessible verification challenge, and the website backend checks the resulting proof before accepting the action.
### Problem boundaries
Focus on challenge issuance and server-side verification; CAPTCHA is one abuse-prevention layer, not proof of identity.

---

## 02 — Requirements

### Functional requirements
| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Register protected sites | Must |
| FR-02 | Issue a short-lived challenge | Must |
| FR-03 | Provide accessible alternatives | Must |
| FR-04 | Verify a submitted proof server-side | Must |
| FR-05 | Bind proof to site and intended action | Must |
| FR-06 | Reject expired or previously consumed proofs | Must |
| FR-07 | Support policy management | Should |
| FR-08 | Collect aggregate service metrics | Should |
### Non-functional requirements
| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Latency | p95 verification under 100 ms service-side, illustrative |
| NFR-02 | Availability | 99.9% monthly illustrative |
| NFR-03 | Consistency | Single successful consumption per proof |
| NFR-04 | Security | Authenticated site verification and protected state |
| NFR-05 | Accessibility | Alternative verification and recovery path |
| NFR-06 | Privacy | Minimize collection and retention |
### Out of scope for the initial version
Identity verification, vendor-specific detection models and global active-active state.

---

## 03 — Scale

### Workload assumptions
| Metric | Illustrative value | Reasoning |
|---|---|---|
| Challenges/day | 1,000,000 | Issuance workload |
| Verifications/day | 800,000 | 80% of issuance |
| Average issue RPS | 11.57 | 1,000,000 / 86,400 |
| Average verify RPS | 9.26 | 800,000 / 86,400 |
| Peak combined RPS | 200 | Planning assumption |
| Challenge lifetime | 300 seconds | Illustrative |
| State size | 1 KB | Excludes indexes |
| Peak live state | ~60 MB | 200 issues/sec × 300 sec × 1 KB upper bound |
### Capacity notes
Account for replicas, expired state cleanup and telemetry. Human completion time is excluded from API latency.
### Questions for a production sizing exercise
Measure completion rates, peak traffic, accessibility outcomes and retention needs.

---

## 04 — Naive Architecture

### Simplest viable design
One application issues challenges and verifies proofs against a single state database. The site backend receives the decision.
### Naive architecture diagram
See `architecture.png`.
### Why begin here?
Establish the lifecycle and verification contract before distributing state.

---

## 05 — Problems

| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single server | One process | Outage or saturation | Replicas |
| State contention | Concurrent verification | Slow decisions | Atomic state operations |
| Expired state | Retention overhead | Storage growth | TTL cleanup |
| Dependency outage | State unavailable | Verification unavailable | Redundancy and explicit failure policy |
| Accessibility barriers | Single challenge modality | Excluded visitors | Accessible alternatives |

---

## 06 — Architecture Evolution

### V1 — Single application and database
Issue and verify through one application; `architecture.png`.
### V2 — Replicated application behind a load balancer
Stateless replicas share authoritative state; `architecture-v2.png`.
### V3 — Shared atomic challenge state
Separate verification workers and use atomic lifecycle transitions; `architecture-v3.png`.
### V4 — Separate control and verification planes
Versioned site policies and asynchronous telemetry; `architecture-evolved.png`.
### Global evolution
Regional services route verification to the authoritative home region for each proof; do not accept the same proof independently in multiple regions. See `architecture-global.png`.

---

## 07 — Components

| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Route healthy instances | V2 onward |
| TBD | Site backend | Enforce verification decision | Yes |
| TBD | Challenge API | Issue challenges | Yes |
| TBD | Verification service | Validate proof and lifecycle | Yes |
| TBD | State store | Authoritative challenge state | Yes |
| TBD | Policy store | Site settings | V4 |
| TBD | Telemetry worker | Aggregate service metrics | V4 |
### Component interactions
Site backend communicates with the service; verification uses authoritative state. Only CMP-001 is confirmed.

---

## 08 — Data Flow

### Flow A: Challenge issuance
Site requests a challenge; service validates site registration and records expiry and action binding; visitor receives challenge reference.
### Flow B: Verification
Site backend submits proof; service checks validity and atomically consumes a valid unused proof. Site backend accepts the protected action only after success.
### Flow C: Telemetry
Aggregate issuance, completion and verification outcomes asynchronously without storing challenge answers in logs.
### Example challenge record
```json
{"challenge_id":"example-id","site_id":"site_123","action":"form_submit","status":"issued","expires_at":"2026-11-19T10:05:00Z"}
```
### Example decision event
```json
{"event_id":"evt_001","outcome":"verified","policy_version":1}
```

---

## 09 — Failure Scenarios

| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| Replica failure | Health checks | Reduced capacity | Remove unhealthy instance |
| State unavailable | Dependency errors | No reliable decision | Return unavailable; retry safely |
| Expired proof | Expiry check | Verification rejected | Issue fresh challenge |
| Telemetry outage | Backlog | Delayed reporting | Bounded buffering |
| Region outage | Regional health | Home-region proofs unavailable | Reissue after recovery or in healthy region |

---

## 10 — Scaling

### 10× traffic
Replicate stateless APIs and measure state throughput.
### 100× traffic
Partition state by challenge identifier; separate issuance and verification capacity.
### 1,000× traffic
Deploy regional services with explicit authoritative ownership and disaster recovery.
### Scaling decision checklist
Measure CPU, state latency, availability, completion rates and cost before scaling.

---

## 11 — Trade-offs

| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| State | Single database | Partitioned store | Simplicity versus scale |
| Verification | Interactive | Accessible alternative | Friction and inclusion |
| Deployment | Single region | Multi-region | Latency versus consistency |
| Telemetry | Synchronous | Asynchronous | Freshness versus request latency |
### Required decision record format
Context, options, chosen option, rationale, gains, sacrifices and revisit conditions.

---

## 12 — Cost

| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Load balancer | Requests and hours | RPS |
| Compute | Instance hours | CPU and memory |
| State | Operations and storage | Live challenges |
| Telemetry | Ingestion and retention | Events/day |
| Network | Transfer | Bytes/request |
### Estimation worksheet
Monthly cost = compute + load balancing + state + network + telemetry. Use selected provider prices, region and date; no fabricated currency estimate.

---

## 13 — Security

### Threats and controls
Protect site credentials, enforce authorization, use encrypted transport, short-lived proofs and single-use verification. Minimize personal data and provide accessible recovery. Treat verification as one layer rather than an identity guarantee.
### Validation rules
Validate site registration, intended action, expiry and proof status on the server. Return a controlled unavailable result when authoritative state cannot be checked.

---

## 14 — Observability

| Metric | Type | Why it matters |
|---|---|---|
| challenge_issued_total | Counter | Demand |
| verification_total | Counter | Decision volume |
| verification_latency_ms | Histogram | Performance |
| verification_errors_total | Counter | Reliability |
| completion_ratio | Gauge / derived | Usability |
| healthy_instances | Gauge | Capacity |
### Logs and traces
Use request IDs and outcome classes; exclude secrets and challenge answers.
### Example alerts
High error rate, sustained latency, low healthy capacity or sudden completion-rate changes.

---

## 15 — Simulation

### Learning objective
Observe issuance and verification capacity, state pressure and dependency failures using synthetic outcomes.
### Initial simulation controls
See `simulation-spec.md` for defaults and equations.
### Expected outputs
Offered, completed, unavailable requests and modeled latency.
### Experiments
Increase traffic, fail a worker, disable state and restore service.
### Simulation model warning
Synthetic completion outcomes are not a model of CAPTCHA effectiveness.

---

## 16 — Design Challenge

### Challenge: Build a CAPTCHA service
Design challenge issuance, accessible completion and server-side verification. Start with one server and evolve to replicated services. Deliver requirements, architecture, API contracts, state model, capacity estimates, failure analysis, security and observability plan. Explain expiration, single-use state and accessibility.

---

## 17 — AI Interview

### Interviewer question bank
| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What does CAPTCHA do? | Verification layer, limitations |
| Q02 | Beginner | Why expire challenges? | Lifecycle and retention |
| Q03 | Intermediate | Why use atomic state? | Single consumption |
| Q04 | Intermediate | How do replicas share state? | Authoritative storage |
| Q05 | Advanced | How do regions coordinate? | Ownership and failure policy |
| Q06 | Intermediate | How do you support accessibility? | Alternatives and recovery |
### Follow-up evaluation dimensions
Assumptions, data flow, failure modes, privacy and trade-offs.

---

## 18 — Architecture Score

### Proposed evaluation rubric
| Dimension | Weight | Evidence |
|---|---|---|
| Requirements completeness | 10 | User journeys |
| Scalability | 15 | Capacity |
| Reliability | 15 | Recovery |
| Latency and performance | 10 | API latency |
| Data design and consistency | 15 | Lifecycle |
| Security | 10 | Verification controls |
| Cost efficiency | 10 | Cost model |
| Trade-off justification | 10 | Decisions |
| Observability and operations | 5 | Metrics |
| **Total** | 100 |  |
### Scoring guidance
Score each dimension 0–4 (missing to detailed). Weighted score = sum(score / 4 × weight). Require reviewer evidence.
