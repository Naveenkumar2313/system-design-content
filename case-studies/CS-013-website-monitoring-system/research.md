# CS-013 — Design a Website Monitoring System

> **Content status:** Educational reference design, not a claim about any monitoring vendor's private architecture.
>
> **Source alignment:** Category A: Web & Internet Systems; supplied 18-component template. CS-008 follows the local index; official catalogue number unverified.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-008 |
| System name | Website Monitoring System |
| Real-world reference | UptimeRobot / Pingdom (reference products) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 8 (local index; catalogue unverified) |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | Periodically check authorized websites, record availability and latency, and notify owners of incidents. |
| Primary users | Website owners, developers, operations teams, administrators |
| Key concepts | Scheduling, HTTP, DNS, TLS, queues, workers, time-series storage, incident detection, notifications, observability |
| Related component IDs | CMP-001 (Load Balancer); additional components TBD |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-19 |

### Scope and assumptions
- External HTTP/HTTPS synthetic monitoring of authorized targets; no intrusive scanning.
- Single region initially; multi-region confirmation is an extension.
- Default 60-second interval, five-second timeout, two consecutive failures to open and two successes to resolve.
- All figures are teaching assumptions, not vendor measurements.

---
## 01 — Story
Website owners need to discover outages before customers report them. A monitoring service periodically checks a configured endpoint, records the outcome and alerts the owner when sustained failures occur.
### Actors
1. **Website owner:** Registers and verifies targets.
2. **Operations engineer:** Reviews incidents and latency.
3. **Notification recipient:** Receives outage and recovery messages.
4. **Administrator:** Manages access, quotas and service health.
### Core user journey
1. Register and verify an authorized website.
2. Configure interval and expected response.
3. Scheduler dispatches a check; worker records status and duration.
4. Evaluator opens an incident after two consecutive failures.
5. Owner receives an alert and later a recovery notification.
### Problem boundaries
Synthetic availability and latency monitoring, not full application tracing or security scanning.

---
## 02 — Requirements
### Functional requirements
| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Register and verify website ownership | Must |
| FR-02 | Configure interval, timeout and expected HTTP status | Must |
| FR-03 | Schedule checks with unique IDs | Must |
| FR-04 | Execute bounded HTTP/HTTPS checks | Must |
| FR-05 | Persist results and timestamps | Must |
| FR-06 | Detect incidents and recovery | Must |
| FR-07 | Send outage and recovery notifications | Must |
| FR-08 | Show uptime, latency and incident history | Must |
| FR-09 | Support pause and maintenance windows | Should |
| FR-10 | Support multiple probe regions | Should |
| FR-11 | Export authorized reports | Should |
### Non-functional requirements
| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Latency | Dispatch p95 within 10 seconds of due time under baseline |
| NFR-02 | Availability | Illustrative 99.9% monthly monitoring control plane |
| NFR-03 | Durability | Persist results before job acknowledgment |
| NFR-04 | Consistency | Idempotent results and ordered incident evaluation |
| NFR-05 | Scalability | Scale scheduler, probes and reporting independently |
| NFR-06 | Security | Verified targets, tenant isolation and bounded checks |
| NFR-07 | Observability | Schedule lag, unknown checks, queue age and notification failures |
| NFR-08 | Maintainability | Versioned check policy and separate worker roles |
### Out of scope for the initial version
- Browser transaction recording, internal agents, billing and global consensus.

---
## 03 — Scale
### Workload assumptions
| Metric | Illustrative value | Reasoning |
|---|---|---|
| Active monitors | 10,000 | Teaching workload |
| Check interval | 60 seconds | One check/minute |
| Checks/day | 14,400,000 | 10,000 × 1,440 |
| Average checks/sec | ~167 | 10,000 / 60 |
| Peak checks/sec | 300 | Illustrative scheduling burst |
| Average result size | 500 bytes | Metadata only |
| Result growth | 7.2 GB/day | 14.4 million × 500 bytes |
| Raw retention | 30 days | 216 GB before indexes/replicas |
| Availability target | 99.9% | Control plane objective |
### Capacity notes
- At 300 checks/sec and five-second worst-case duration, up to 1,500 concurrent probes may be needed; enforce per-target and global limits.
- Spread schedules to avoid synchronized bursts. Retain rollups longer than raw results if required.
### Questions for a production sizing exercise
- What are target latency, intervals, regions, result size, retention and alert volume?

---
## 04 — Naive Architecture
### Simplest viable design
One application runs a periodic scheduler, checks websites, stores results and sends alerts using a single database.
**Components:** Client/dashboard, single monitoring server, authorized website, database and notification provider.
**Check flow:** Select due monitors, execute bounded checks, store observations, evaluate incident and notify.
### Naive architecture diagram
See `architecture.png`.
### Why begin here?
It demonstrates the complete monitoring loop with minimal infrastructure, but failures and slow targets compete with the dashboard.

---
## 05 — Problems
| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single server | Shared process | Monitoring gaps | Replicate API and workers |
| Slow targets | Long probe duration | Schedule lag | Bounded concurrency and timeouts |
| Duplicate checks | Retries or scheduler failover | Incorrect counts | Unique check IDs |
| Queue overload | Due work exceeds capacity | Missed checks | Backpressure and quotas |
| False alerts | Transient network failure | Alert fatigue | Consecutive checks and regional confirmation |
| Notification outage | Provider unavailable | Delayed alerts | Durable outbox and retries |
| Storage growth | Frequent results | Slow queries | Partition and roll up |
| Unsafe target | Untrusted destination | Internal resource exposure | Ownership and egress policy |
| Monitoring outage | Probe infrastructure fails | Misleading uptime | Unknown state and independent health checks |

---
## 06 — Architecture Evolution
These are proposed learning stages, not vendor history.
### V1 — Single application and database
- Combined scheduler, probe, dashboard and incident logic; see `architecture.png`.
### V2 — Replicated API and probe workers
- CMP-001 distributes dashboard/API traffic. A leased scheduler assigns unique jobs to workers; shared database stores configuration and results. See `architecture-v2.png`.
### V3 — Durable scheduled job pipeline
- Add durable broker, worker leases, deadlines and idempotent result writes; see `architecture-v3.png`.
### V4 — Separate monitoring, incidents and reporting
- Separate configuration, result storage, incident evaluator, notification outbox and reporting API; see `architecture-evolved.png`.
### Global evolution
- Dispatch checks to independent regions, combine results by monitor and time window, require a configured regional quorum, distinguish missing observations from target failure. See `architecture-global.png`.

---
## 07 — Components
| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Routes healthy API instances | V2 onward |
| TBD | Client | Configures monitors and views reports | Yes |
| TBD | Monitor API | Authorizes and validates configuration | Yes |
| TBD | Scheduler | Creates unique due jobs | Yes |
| TBD | Probe Worker | Executes bounded authorized checks | Yes |
| TBD | Configuration DB | Stores monitors and schedules | Yes |
| TBD | Job Broker | Durable pending jobs | V3 |
| TBD | Result Store | Stores time-stamped observations | Yes |
| TBD | Incident Evaluator | Applies ordered transition rules | Yes |
| TBD | Notification Worker | Delivers outbox messages | Yes |
| TBD | Reporting API | Queries results and incidents | Yes |
| TBD | Observability | Tracks internal service health | Recommended |
**Component ID policy:** CMP-001 is the only confirmed ID; other IDs remain TBD.
### Component interactions
- Client → Load Balancer → Monitor API → Configuration DB.
- Scheduler → Job Broker → Probe Worker → Authorized Website.
- Probe Worker → Result Store → Incident Evaluator → Outbox → Notification Worker.
- Reporting API reads results and incident state.

---
## 08 — Data Flow
### Flow A: Register monitor
1. Owner authenticates and verifies target ownership.
2. API validates interval, URL and quotas; commits configuration and next due time.
3. Return 201 Created with monitor ID.
### Flow B: Execute check
1. Scheduler publishes unique `(monitor_id, scheduled_at, region)` job using durable scheduling state.
2. Worker leases job, validates destination, executes bounded check and records result.
3. Commit idempotent result before acknowledging job; expired jobs become unknown.
### Flow C: Incident and reporting
1. Evaluator consumes results in scheduled order; missing slots are unknown and reset streaks.
2. Two consecutive failures open an incident; two successes resolve it.
3. Commit incident transition and outbox atomically; notification worker retries delivery.
4. Dashboard returns authorized uptime, latency and incidents; uptime = successful / (successful + failed) checks, with unknown coverage shown separately. Exclude maintenance slots explicitly.
### Example monitor record
```json
{"monitor_id":"mon_123","target":"https://example.com/health","interval_sec":60,"timeout_ms":5000,"expected_status":200,"owner_id":"user_123","enabled":true}
```
### Example check result
```json
{"check_id":"check_001","monitor_id":"mon_123","scheduled_at":"2026-11-19T10:00:00Z","region":"region-a","outcome":"success","http_status":200,"duration_ms":120}
```
See `data-flow.md` for sequence diagrams.

---
## 09 — Failure Scenarios
| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| API instance crash | Health check | Management request fails | Route to healthy replica |
| Scheduler crash | Schedule lag | Due jobs delayed | Lease failover and reconcile slots |
| Worker crash | Lease expiry | Incomplete check | Retry within deadline |
| Broker outage | Publish errors | Jobs delayed | Reconcile due slots; mark missed unknown |
| Result store outage | Write errors | Results delayed | Do not acknowledge until durable |
| Target timeout | Probe deadline | Failed observation | Confirm with consecutive checks |
| Region outage | Probe health | Unknown observations | Use healthy regions and report coverage |
| Notification outage | Delivery failures | Delayed alerts | Durable outbox and retry |
| Duplicate result | Unique constraint | Potential duplicate evaluation | Idempotent writes and transition IDs |

---
## 10 — Scaling
### 10× traffic
- Partition scheduler ownership, add workers and broker capacity; measure concurrency and database writes.
### 100× traffic
- Partition results by monitor/time, isolate noisy tenants, precompute rollups and scale notification workers.
### 1,000× traffic
- Deploy regional probe pools and regional queues; define quorum, residency, failover and retention.
### Scaling decision checklist
- Identify scheduling, network, worker, storage or notification bottlenecks; measure queue age, concurrency, target quotas and cost.

---
## 11 — Trade-offs
| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| Scheduling | Database polling | Durable queue | Simplicity vs. independent scaling |
| Check interval | Short | Long | Detection speed vs. cost and target load |
| Incident confirmation | Single failure | Consecutive failures | Speed vs. false positives |
| Storage | Relational | Time-series | Transactions vs. efficient time queries |
| Regions | Single | Multiple | Cost vs. independent observations |
| Notifications | Synchronous | Outbox | Immediate attempt vs. durability |
### Required decision record format
- Context, options, chosen option, justification, gains, sacrifices and revisit conditions.

---
## 12 — Cost
No provider or region specified; use a cost model rather than invented prices.
### Cost categories
| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Load balancer | Hours and traffic | API requests |
| Probe compute | Worker hours and concurrency | Checks/sec and duration |
| Broker | Operations and retention | Jobs/sec and backlog |
| Database | Storage and IOPS | Monitor count and writes |
| Result storage | Retention and query scans | Bytes/day |
| Network | Regional transfer | Request/response bytes |
| Notifications | Delivery attempts | Messages by channel |
| Observability | Telemetry volume | Logs and metrics |
### Estimation worksheet
- Monthly compute = instance-hours × provider rate; storage = retained GB × rate; add broker, network, notifications, backups and telemetry.
- Record provider, region, date, currency, pricing URL and assumptions.

---
## 13 — Security
### Threats and controls
| Threat | Mitigation |
|---|---|
| Unauthorized monitoring | Verify ownership and enforce quotas |
| Unsafe destinations | Restrict to approved public targets; enforce egress policy and revalidate redirects |
| Tenant data leakage | Authentication and tenant-scoped authorization |
| Excessive probe traffic | Bound intervals, concurrency and response size |
| Secret exposure | Managed secrets and redacted logs |
| Notification abuse | Verify recipients and rate-limit delivery |
| Injection | Parameterized queries and output encoding |
### Validation rules
- Allow HTTP/HTTPS only; validate approved destinations at registration and execution. Do not permit private or reserved destinations in this public monitoring design.
- Use bounded timeouts, response sizes and redirect counts; do not retain response bodies by default.

---
## 14 — Observability
### Metrics
| Metric | Type | Why it matters |
|---|---|---|
| checks_due_total | Counter | Expected work |
| checks_completed_total | Counter | Completed observations |
| checks_unknown_total | Counter | Coverage gaps |
| schedule_lag_seconds | Histogram | Dispatch delay |
| probe_duration_ms | Histogram | Target latency |
| job_queue_depth | Gauge | Backlog |
| incidents_open | Gauge | Active incidents |
| notification_failures_total | Counter | Delivery health |
| healthy_workers | Gauge | Capacity |
### Logs and traces
- Correlate check ID and incident ID; redact secrets and avoid unbounded metric labels.
### Example alerts
- Schedule lag exceeds target; unknown rate increases; queue age grows; notification failures persist; independent service heartbeat fails.

---
## 15 — Simulation
### Learning objective
Observe how scheduling, worker capacity, outages and incident rules affect monitoring.
### Initial simulation controls
| Control | Default | Suggested range |
|---|---|---|
| Active monitors | 10000 | 1–100000 |
| Check interval (seconds) | 60 | 30–600 |
| Probe workers | 3 | 1–20 |
| Checks/sec per worker | 100 | 10–1000 |
| Failed workers | 0 | 0–worker_instances |
| Pending job limit | 10000 | 100–100000 |
| Broker available | true | On / Off |
| Result store available | true | On / Off |
| Target failure percentage | 0 | 0–100 |
| Notification provider available | true | On / Off |
| Probe timeout (ms) | 5000 | 1000–10000 |
### Expected outputs
- Due/completed/missed checks, queue depth, worker utilization, incident transitions and notification backlog.
### Experiments
- Increase monitors 10×, fail workers, disable broker/store, inject target failures, recover targets and restore notification provider.
### Simulation model warning
- See `simulation-spec.md`; simplified deterministic model, not measured production behavior.

---
## 16 — Design Challenge
### Challenge: Build a website monitoring system
**Requirements:** Register authorized targets, schedule checks, store results, detect incidents, notify owners and display history.
**Constraints for the exercise:** Start with one server, add three workers, introduce durable jobs, fail a worker and increase monitors 10×.
**Learner deliverables:** Requirements, architecture, API contracts, data model, sequence flows, capacity estimates, failure analysis, trade-offs, security and observability.
**Evaluation prompts:** How do you avoid duplicate jobs? How do you distinguish unknown from downtime? How do you prevent false alerts and recover notification delivery?

---
## 17 — AI Interview
### Interviewer question bank
| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What are the core requirements? | Registration, checks, results, incidents, alerts |
| Q02 | Beginner | How do you schedule checks? | Due times, jitter, unique slots |
| Q03 | Beginner | How do you measure uptime? | Success denominator and coverage |
| Q04 | Intermediate | How do you handle slow targets? | Timeouts and bounded concurrency |
| Q05 | Intermediate | Why use a broker? | Durability and backpressure |
| Q06 | Intermediate | How do you avoid duplicate results? | Unique check IDs |
| Q07 | Intermediate | How do you reduce false alerts? | Consecutive failures and quorum |
| Q08 | Intermediate | What if a worker fails? | Lease expiry and deadline |
| Q09 | Advanced | How do you partition results? | Monitor/time and retention |
| Q10 | Advanced | How do multiple regions work? | Independent probes and quorum |
| Q11 | Advanced | How do you secure targets? | Ownership, egress and quotas |
| Q12 | Advanced | How do you estimate cost? | Checks, duration, storage and notifications |
### Follow-up evaluation dimensions
- Clarify assumptions, explain data flow, identify failures, justify trade-offs and avoid unsupported vendor claims.

---
## 18 — Architecture Score
### Proposed evaluation rubric
| Dimension | Weight | Evidence |
|---|---|---|
| Requirements completeness | 10 | User journeys |
| Scalability | 15 | Capacity and bottlenecks |
| Reliability | 15 | Recovery and unknown state |
| Latency and performance | 10 | Schedule and probe latency |
| Data design and consistency | 15 | Idempotency and ordering |
| Security | 10 | Target validation and access |
| Cost efficiency | 10 | Cost assumptions |
| Trade-off justification | 10 | Alternatives |
| Observability and operations | 5 | Metrics and alerts |
| **Total** | **100** |  |
### Scoring guidance
- 0: Missing or incorrect; 1: Mentioned; 2: Partial; 3: Correct and justified; 4: Detailed and measurable.
Weighted score = sum of (dimension score / 4 × dimension weight). Human review and evidence required.
