# CS-DomainRegistration — Design a Domain Registration System

> **Content status:** Educational reference design. This document is a proposed design for a domain registrar service. It does not claim to describe any vendor’s private production architecture.
>
> **Source alignment:** This adaptation uses Category A: Web & Internet Systems and the supplied URL-shortener template; the Domain Registration System catalogue number is not verified. The required structure follows the 18-component case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-DomainRegistration |
| System name | Domain Registration System |
| Real-world reference | Namecheap / GoDaddy (reference products, not an asserted architecture source) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | To be verified |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | A service that searches, registers, renews and manages domain names through authoritative registries. |
| Primary users | Registrants, administrators, support staff |
| Key concepts | DNS, EPP, RDAP, API design, payments, idempotency, queues, consistency, observability |
| Related component IDs | CMP-001 (Load Balancer); additional components can be added to the component index as they are documented |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-15 |

### Scope and assumptions

- Registrar integrates with supported TLD registries using EPP or approved registry APIs.
- Search availability is advisory; only registry confirmation establishes registration.
- Payment authorization, registration, renewal, transfer and nameserver management are supported.
- Initial deployment is single-region; global access is an evolution stage.
- Workload figures are illustrative; TLD policies and prices vary.

---

## 01 — Story

People need unique domain names for websites and email. A registrar coordinates registration with the registry responsible for each top-level domain (TLD).

### Actors
1. **Registrant:** Searches, purchases and manages domains.
2. **Registry:** Maintains authoritative registration records for a TLD.
3. **Payment provider:** Authorizes and captures payments.
4. **Administrator:** Handles support, compliance and reconciliation.

### Core user journey
1. Search `example-shop.com`.
2. Check registry availability and display price and terms.
3. Submit contact information and authorize payment.
4. Registrar submits registration to registry.
5. Registry confirms creation; registrar records ownership and captures payment.
6. Registrant configures nameservers and renews before expiry.
7. Registrar sends renewal reminders and displays status.

### Problem boundaries
Registrar operations, not website hosting or operating the TLD registry itself. DNS resolution is separate from registration.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Search domain availability | Must |
| FR-02 | Validate and normalize domain names | Must |
| FR-03 | Register a domain through its registry | Must |
| FR-04 | Reject unsupported or invalid names | Must |
| FR-05 | Prevent duplicate local orders and registry registrations | Must |
| FR-06 | Manage nameservers and contact details | Should |
| FR-07 | Renew domains and send reminders | Should |
| FR-08 | Support transfers subject to TLD policy | Should |
| FR-09 | Track payment and registration events | Should |
| FR-10 | Provide order and domain history | Could |
| FR-11 | Enforce abuse and compliance policies | Should |

### Non-functional requirements

| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Search latency | p95 under 500 ms for cached search; live registry calls measured separately |
| NFR-02 | Availability | 99.9% monthly registrar API target |
| NFR-03 | Durability | Persist accepted orders and registry confirmations |
| NFR-04 | Consistency | Registry authoritative; reconcile ambiguous outcomes |
| NFR-05 | Scalability | Search and registration workers scale independently |
| NFR-06 | Security | MFA, ownership checks, payment tokenization |
| NFR-07 | Observability | Track latency, errors, registry and payment state |
| NFR-08 | Maintainability | Separate registry adapters and order orchestration |

### Out of scope for the initial version
- Running a TLD registry, domain auctions, web hosting and email hosting.

---

## 03 — Scale

All values are illustrative teaching assumptions.

### Workload assumptions

| Metric | Illustrative value | Reasoning |
|---|---|---|
| Registered users | 100,000 | Planning scenario |
| Managed domains | 200,000 | Portfolio |
| Searches/day | 1,000,000 | Read workload |
| Registrations/day | 5,000 | Write workload |
| Average search RPS | 11.6 | 1,000,000 / 86,400 |
| Peak search RPS | 200 | Burst assumption |
| Average registration RPS | 0.058 | 5,000 / 86,400 |
| Average record size | 4 KB | Metadata only |
| Record growth | 20 MB/day | Before indexes and replicas |
| Availability target | 99.9% | Illustrative |

### Capacity notes
- At 80% search cache hits, 200,000 live checks/day remain. At peak, 40 checks/sec may reach registries; enforce per-registry limits.
- Renewal bursts and payment retries require separate capacity.

### Questions for a production sizing exercise
- What are TLD mix, registry quotas, search popularity, renewal peaks and retention requirements?

---

## 04 — Naive Architecture

### Simplest viable design
One application handles search, orders and registry/payment calls; one database stores accounts, orders and domains.

**Components**
- Client/browser
- Single application server
- Registration database, registry and payment provider

**Registration flow**
1. Validate name and check registry.
2. Authorize payment and persist pending order.
3. Submit registry create; record confirmation and capture payment.
4. On uncertain result, mark pending reconciliation rather than retry blindly.

**Search flow**
1. Client searches name.
2. Application queries registry and returns advisory availability.

### Naive architecture diagram
See `architecture.png`.

### Why begin here?
Demonstrates authoritative registration and order state before distributed orchestration.

---

## 05 — Problems

| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single-server bottleneck | Shared compute | Slow requests | Replicate API |
| Single point of failure | One server/database | Outage | Redundancy |
| Registry pressure | Repeated searches | Quota exhaustion | Short TTL cache |
| Availability race | Concurrent buyers | Order failure | Registry create decides |
| Payment mismatch | Independent systems | Charge without domain | Reconciliation and compensation |
| Timeout ambiguity | Lost response | Unknown ownership | Query registry before retry |
| Renewal failure | Expired payment or job | Domain loss | Reminders and retries |
| Stale cache | Changing registry state | Misleading search | Live check and advisory label |
| Account takeover | Weak access controls | Unauthorized transfer | MFA and locks |
| Poor visibility | Missing telemetry | Slow recovery | Metrics and alerts |

---

## 06 — Architecture Evolution

These are proposed learning stages, not vendor history.

### V1 — Single application and database
- Synchronous registry and payment integration. See `architecture.png`.
### V2 — Replicated application behind a load balancer
- CMP-001 distributes requests to stateless APIs sharing durable orders. See `architecture-v2.png`.
### V3 — Cached domain search
- Short-lived advisory availability cache; live checks and registry create remain authoritative. See `architecture-v3.png`.
### V4 — Asynchronous registration and lifecycle orchestration
- Durable outbox, queue, workers, registry adapters, payment reconciliation and renewal scheduler. See `architecture-evolved.png`.
### Global evolution
- Regional search APIs and caches; one fenced authoritative order writer with replicated data and controlled failover. Registry-specific quotas apply globally. See `architecture-global.png`.

---

## 07 — Components

| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Routes healthy APIs | V2 onward |
| TBD | Client | Search and manage domains | Yes |
| TBD | DNS | Resolves registrar hostname | Yes |
| TBD | Registration API | Validates orders and ownership | Yes |
| TBD | Registry Adapter | EPP/API operations | Yes |
| TBD | Registration Database | Orders, domains and contacts | Yes |
| TBD | Cache | Advisory search results | V3 |
| TBD | Message Broker | Durable lifecycle jobs | V4 |
| TBD | Registration Worker | Executes and reconciles orders | V4 |
| TBD | Payment Provider | Authorization/capture/refund | Yes |
| TBD | Renewal Scheduler | Schedules renewals and reminders | V4 |
| TBD | Monitoring / Observability | Telemetry | Recommended |

**Component ID policy:** CMP-001 is the only confirmed ID; others remain TBD.

### Component interactions
- Client → DNS → Load Balancer → API → cache/registry adapter.
- API → database/outbox → broker → worker → registry and payment provider.
- Scheduler → broker; monitoring collects telemetry.

---

## 08 — Data Flow

### Flow A: Search a domain
1. `GET /api/v1/domains/search?name=example-shop.com` validates and normalizes name.
2. Read advisory cache or query registry; return availability and quote with expiry.

### Flow B: Register a domain
1. `POST /api/v1/orders` with idempotency key, quote and contact reference.
2. Persist order and outbox atomically; return 202 with order ID.
3. Worker authorizes payment, checks registry and submits create.
4. Confirm registry result, persist domain, capture payment; reconcile uncertain responses.
5. `GET /api/v1/orders/{id}` returns status. If create fails definitively, void authorization; if capture fails after creation, retain domain record and escalate billing.

### Flow C: Renewal and management
1. Scheduler finds upcoming expirations; sends reminders and queues authorized renewals.
2. Worker checks registry status, authorizes payment and submits renewal.
3. Confirm new expiry, capture payment and notify owner. Transfers and nameserver updates require ownership checks and registry policy validation.

### Example domain record
```json
{"domain_id":"dom_123","name":"example-shop.com","owner_id":"user_123","registry_status":"active","expires_at":"2027-11-15T00:00:00Z","nameservers":["ns1.example.net","ns2.example.net"]}
```
### Example registration event
```json
{"event_id":"evt_001","order_id":"ord_123","type":"registry_confirmed","occurred_at":"2026-11-15T10:00:00Z"}
```
Avoid logging contact details, payment tokens or transfer credentials.

---

## 09 — Failure Scenarios

| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| Single-server bottleneck | Health checks / state alerts | Slow requests | Replicate API |
| Single point of failure | Health checks / state alerts | Outage | Redundancy |
| Registry pressure | Health checks / state alerts | Quota exhaustion | Short TTL cache |
| Availability race | Health checks / state alerts | Order failure | Registry create decides |
| Payment mismatch | Health checks / state alerts | Charge without domain | Reconciliation and compensation |
| Timeout ambiguity | Health checks / state alerts | Unknown ownership | Query registry before retry |
| Renewal failure | Health checks / state alerts | Domain loss | Reminders and retries |
| Stale cache | Health checks / state alerts | Misleading search | Live check and advisory label |
| Account takeover | Health checks / state alerts | Unauthorized transfer | MFA and locks |
| Poor visibility | Health checks / state alerts | Slow recovery | Metrics and alerts |
| Registry timeout | Timeout | Unknown result | Reconcile via registry info and transaction records |
| Broker unavailable | Publish errors | Jobs delayed | Replay durable outbox |
| Region outage | Regional probes | API degraded | Fenced failover |

Document trigger, user outcome, recovery and metrics.

---

## 10 — Scaling

### 10× traffic
- Scale APIs behind CMP-001 and search cache; enforce registry quotas.
### 100× traffic
- Partition orders by ID and domains by canonical name; maintain unique index and per-registry queues. Scale workers independently.
### 1,000× traffic
- Regional search, replicated reads and fenced order writer failover; coordinate global registry quotas and define RPO/RTO.
### Scaling decision checklist
- Identify API, database, registry, payment or queue bottlenecks; measure cost, consistency and renewal deadlines.

---

## 11 — Trade-offs

| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| Search | Live | Cached | Freshness vs quota |
| Orders | Synchronous | Asynchronous | Immediate response vs durable recovery |
| Storage | Relational | Key-value | Transactions vs lookup scale |
| Payment | Authorize then capture | Capture then refund | Reduced refunds vs authorization expiry |
| Registry integration | Direct EPP | Reseller API | Control vs complexity |
| Renewal | Manual | Auto-renew | Control vs continuity |
| Region | Single | Multi-region | Consistency vs availability |

### Required decision record format
Document context, options, choice, rationale, gains, sacrifices and revisit conditions.

---

## 12 — Cost

No provider or region is specified; use a cost model.

### Cost categories

| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Load balancer | Hours/bytes | Requests |
| API compute | Instance hours | RPS |
| Database | Storage/IOPS | Orders and backups |
| Cache | Memory | Working set |
| Broker | Jobs/retention | Events/sec |
| Workers | Compute | Jobs and duration |
| Registry | Per-domain/TLD fees | Creates and renewals |
| Payment | Transaction fees | Successful charges |
| Observability | Ingestion | Telemetry |

### Estimation worksheet
- Compute = instances × hourly price × hours.
- Registry = sum of registration, renewal and transfer fees by TLD.
- Payment = transaction fees + percentage of processed value.
- Add storage, cache, queue, network, support and compliance. Record date, currency and provider.

---

## 13 — Security

### Threats and controls

| Threat | Mitigation |
|---|---|
| Account takeover | MFA and sensitive-action verification |
| Unauthorized transfer | Ownership checks and transfer locks |
| Payment fraud | Provider risk controls and tokenization |
| Duplicate order | Idempotency and unique constraints |
| Contact data leakage | Encryption, access controls and retention |
| Injection | Parameterized queries |
| Abusive search | Rate limits and quotas |
| Registry credential exposure | Managed secrets and least privilege |
| Domain abuse | Reporting and policy-based review |
| Webhook forgery | Verify signatures and deduplicate events |

### Validation rules
- Normalize IDNs using applicable IDNA rules and registry policy; validate label lengths and TLD.
- Validate contact requirements, quote expiry and authorization. Never treat cached availability as a reservation.

---

## 14 — Observability

### Metrics

| Metric | Type | Why it matters |
|---|---|---|
| `search_requests_total` | Counter | Traffic |
| `search_latency_ms` | Histogram | Performance |
| `search_errors_total` | Counter | Reliability |
| `registration_total` | Gauge | Volume |
| `registration_failures_total` | Gauge | Order reliability |
| `cache_hit_ratio` | Gauge | Cache effectiveness |
| `registry_latency_ms` | Histogram | Registry dependency |
| `load_balancer_healthy_targets` | Gauge | API health |
| `order_queue_depth` | Gauge | Backlog |
| `renewal_due_total` | Counter | Renewal workload |

### Logs and traces
- Correlate order and registry transaction IDs; redact personal data and secrets.
### Example alerts
- High search latency, registry failures, pending orders, payment mismatches or approaching renewal deadlines.

---

## 15 — Simulation

### Learning objective
Observe search caching, registry quotas, API failures and order backlog.

### Initial simulation controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---|---|---|
| search_rps | Incoming search RPS | Slider | 200 | 10 | 100000 |
| create_rps | Registration orders/sec | Slider | 5 | 0 | 1000 |
| app_instances | API instances | Stepper | 3 | 1 | 20 |
| instance_capacity_rps | Capacity per instance | Slider | 500 | 100 | 5000 |
| cache_enabled | Enable cache | Toggle | true | — | — |
| cache_hit_ratio | Cache hit ratio | Slider | 80 | 0 | 100 |
| registry_capacity | Registry operations/sec | Slider | 100 | 1 | 20000 |
| registry_latency_ms | Registry latency | Slider | 100 | 1 | 5000 |
| lb_algorithm | Routing algorithm | Select | round_robin | — | — |
| failed_instances | Failed API instances | Stepper | 0 | 0 | app_instances |
| worker_capacity | Worker jobs/sec | Slider | 10 | 1 | 1000 |
| broker_available | Broker available | Toggle | true | — | — |

### Expected outputs
- Search success, rejection, cache hits/misses, registry utilization, latency and order backlog.
### Experiments
1. Disable cache.
2. Increase traffic 10×.
3. Fail an API instance.
4. Increase registry latency.
5. Disable broker.
6. Increase order rate beyond worker capacity.
### Simulation model warning
See `simulation-spec.md`; results are teaching estimates.

---

## 16 — Design Challenge

### Challenge: Build a domain registration system
**Requirements**
- Search, register, renew, manage nameservers, reconcile payment and registry outcomes.
**Constraints for the exercise**
- Start with one API, evolve to three, add cache, inject failure and increase traffic 10×.
**Learner deliverables**
1. Requirements.
2. Architecture diagrams.
3. API contracts.
4. Data model.
5. Search, registration and renewal flows.
6. Capacity estimates.
7. Failure analysis.
8. Trade-off records.
9. Security and observability plan.
**Evaluation prompts**
- Who guarantees uniqueness? How do you recover ambiguous creates? How do renewals survive outages?

---

## 17 — AI Interview

### Interviewer question bank

| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | Core requirements? | Search, register, renew |
| Q02 | Beginner | Who guarantees uniqueness? | Authoritative registry |
| Q03 | Beginner | Which database? | Transactional orders |
| Q04 | Intermediate | Why cache search? | Quota and latency |
| Q05 | Intermediate | How handle stale availability? | Live check and create |
| Q06 | Intermediate | How handle hot searches? | Cache and quotas |
| Q07 | Intermediate | How reconcile payment? | State machine and compensation |
| Q08 | Intermediate | What if API fails? | Health checks |
| Q09 | Advanced | How partition data? | Canonical name and order ID |
| Q10 | Advanced | How support regions? | Fenced writer |
| Q11 | Advanced | How protect transfers? | MFA and locks |
| Q12 | Advanced | How estimate cost? | TLD fees and infrastructure |

### Follow-up evaluation dimensions
- Clarifies assumptions, explains authoritative state, failures and trade-offs; avoids unsupported vendor claims.

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
