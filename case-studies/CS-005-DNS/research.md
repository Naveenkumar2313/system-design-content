# CS-DNSService — Design a DNS Service

> **Content status:** Educational reference design for a DNS service, not a claim about any provider's private architecture.
>
> **Source alignment:** Category A: Web & Internet Systems; supplied 18-component template. Official case number unverified.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-DNSService |
| System name | DNS Service |
| Real-world reference | Google Public DNS / Cloudflare DNS (reference products) |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | To be verified |
| Difficulty | Beginner (with intermediate extension topics) |
| Short description | Resolve domain names into DNS records using recursive resolution and caching; optionally host authoritative zones. |
| Primary users | Clients, network operators, zone administrators |
| Key concepts | DNS, UDP/TCP, recursion, delegation, TTL, DNSSEC, anycast, caching |
| Related component IDs | CMP-001 (Load Balancer); others TBD |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-15 |

### Scope and assumptions
- Baseline is a recursive caching resolver; managed authoritative DNS is a separate optional plane.
- Support UDP and TCP port 53; encrypted DNS is an extension.
- Public recursive service validates DNSSEC; private deployment restricts recursion to authorized networks.
- All capacity figures are illustrative, not provider disclosures.

---

## 01 — Story
Clients need to translate human-readable names into network addresses and other records. A recursive resolver answers from cache or follows DNS delegations to authoritative servers.
### Actors
1. **Client:** Requests DNS records through a stub resolver.
2. **Recursive resolver:** Resolves and caches answers.
3. **Authoritative server:** Serves records for its zones.
4. **Administrator:** Manages zones and service health.
### Core user journey
1. Client asks for `www.example.com` A records.
2. Resolver checks its cache.
3. On a miss, it follows root, TLD and authoritative referrals as needed.
4. Resolver validates DNSSEC where applicable, caches the answer and returns it with remaining TTL.
### Problem boundaries
DNS resolution and optional authoritative zone hosting; not domain registration or website hosting.

---

## 02 — Requirements
### Functional requirements
| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Resolve A, AAAA and other supported record types | Must |
| FR-02 | Follow delegations and CNAME chains with bounded work | Must |
| FR-03 | Cache positive and negative answers with correct TTL | Must |
| FR-04 | Return NXDOMAIN, NODATA or SERVFAIL correctly | Must |
| FR-05 | Support UDP and TCP DNS | Must |
| FR-06 | Validate DNSSEC | Must |
| FR-07 | Manage authoritative zones in optional hosting plane | Should |
| FR-08 | Publish zone changes reliably | Should |
| FR-09 | Support DoT/DoH | Could |
| FR-10 | Expose aggregate operational metrics | Should |
| FR-11 | Apply abuse and resource limits | Should |

### Non-functional requirements
| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Latency | Cached p95 < 20 ms service-side; misses measured separately |
| NFR-02 | Availability | 99.9% monthly illustrative target |
| NFR-03 | Durability | Persist accepted zone changes; cache is disposable |
| NFR-04 | Consistency | Respect TTL and zone serials; no instant global propagation guarantee |
| NFR-05 | Scalability | Scale resolver and authoritative pools separately |
| NFR-06 | Security | DNSSEC validation, isolation and resource limits |
| NFR-07 | Observability | Track QPS, latency, errors and cache effectiveness |
| NFR-08 | Maintainability | Separate resolution, zone management and serving |

### Out of scope for the initial version
- Registrar operations, custom root infrastructure and global active-active zone editing.

---

## 03 — Scale
All figures are illustrative teaching assumptions.
### Workload assumptions
| Metric | Illustrative value | Reasoning |
|---|---|---|
| Clients | 100,000 | Learning scenario |
| Queries/day | 86,400,000 | 1,000 average QPS |
| Average QPS | 1,000 | Queries / 86,400 |
| Peak QPS | 10,000 | 10× average |
| Cache hit ratio | 90% | Assumption |
| Peak misses/sec | 1,000 | 10,000 × 10% |
| Upstream requests/miss | 3 | Simplified average |
| Peak upstream QPS | 3,000 | 1,000 × 3 |
| Cached RRsets | 1,000,000 | Illustrative working set |
| Average cached RRset | 1 KB | ~1 GB raw cache |
| Availability | 99.9% | Target |

### Capacity notes
- Include cache overhead, TCP connections, validation CPU and network bandwidth.
- At 500 bytes average response, peak response traffic is ~5 MB/s before overhead.
### Questions for a production sizing exercise
- What is the query-type mix, TTL distribution, negative-cache ratio and upstream latency?

---

## 04 — Naive Architecture
### Simplest viable design
One recursive resolver with local cache queries external authoritative DNS hierarchy.
**Components**
- Client/stub resolver
- Single recursive resolver and local cache
- Root, TLD and authoritative servers
**Query flow**
1. Receive DNS query.
2. Check local cache; on miss follow referrals.
3. Validate and cache answer, then respond.
### Naive architecture diagram
See `architecture.png`.
### Why begin here?
Establish DNS correctness before adding replication and global routing.

---

## 05 — Problems
| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single resolver | One process | Outage or saturation | Replicated pool |
| Cold cache | New instance or expiry | Higher latency | Cache-aware routing |
| Upstream timeout | Dependency failure | Slow or failed resolution | Bounded retries and alternate servers |
| Hot names | Popular records expire together | Burst upstream load | Coalesce identical requests |
| Stale answers | TTL mishandling | Incorrect resolution | Track remaining TTL |
| Invalid DNSSEC | Bad signatures or trust chain | SERVFAIL | Validation and alerts |
| Abuse | Excessive requests | Resource exhaustion | Limits and monitoring |

---

## 06 — Architecture Evolution
Proposed teaching stages, not vendor history.
### V1 — Single resolver
- Local cache and iterative resolution; single point of failure.
### V2 — Replicated resolvers behind a load balancer
- CMP-001 distributes UDP/TCP queries to healthy resolver instances; preserve TCP connection affinity.
- Each resolver has local cache; health checks and capacity limits protect service.
### V3 — Cache-optimized resolver pool
- Local TTL-aware caches, bounded negative caching and request coalescing.
- Optional shared cache must preserve validation state and remaining TTL; avoid introducing a mandatory single dependency.
### V4 — Separate resolution and authoritative management
- Independent recursive serving pool and authoritative servers.
- Zone API persists versioned zone data; publisher distributes validated snapshots to authoritative replicas.
- Telemetry is asynchronous; authoritative servers do not offer recursion.
### Global evolution
- Anycast advertises resolver service addresses from multiple regions; routing selects reachable sites, not necessarily nearest geographically.
- Replicate authoritative zones with serial/version tracking; withdraw unhealthy routes and monitor convergence.
- See `architecture-v2.png`, `architecture-v3.png`, `architecture-evolved.png`, `architecture-global.png`.

---

## 07 — Components
| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Distribute DNS queries | V2 onward |
| TBD | Stub resolver | Submit queries | Yes |
| TBD | Recursive resolver | Follow delegations and validate | Yes |
| TBD | Cache | Store RRsets and negative answers | Yes |
| TBD | Root/TLD/authoritative DNS | Provide referrals and answers | External |
| TBD | Zone API | Manage hosted zones | V4 |
| TBD | Zone store | Persist versioned records | V4 |
| TBD | Publisher | Distribute zone snapshots | V4 |
| TBD | Authoritative replicas | Serve hosted zones | V4 |
| TBD | Monitoring | Collect operational telemetry | Recommended |

**Component ID policy:** Only CMP-001 is confirmed.
### Component interactions
- Client → load balancer → resolver → cache or external DNS hierarchy.
- Zone API → zone store → publisher → authoritative replicas.

---

## 08 — Data Flow
### Flow A: Resolve a name
1. Receive query with QNAME, QTYPE and QCLASS.
2. Check cache and remaining TTL.
3. On miss, follow referrals, resolve aliases and validate DNSSEC.
4. Cache eligible answer or negative result; return response. Use TCP when UDP response is truncated.
### Flow B: Publish a zone change
1. Authenticate zone owner and validate records.
2. Commit new zone version and serial.
3. Publish snapshot to authoritative replicas and monitor acknowledgments.
4. Existing recursive caches expire according to TTL.
### Flow C: Analytics
Collect aggregate query counts, latency and response codes asynchronously; avoid unnecessary query-name retention.
### Example cache record
```json
{"name":"www.example.com.","type":"A","class":"IN","records":["192.0.2.10"],"ttl_seconds":300,"validation":"secure"}
```
### Example telemetry event
```json
{"event_id":"evt_001","rcode":"NOERROR","cache_hit":true,"latency_ms":2}
```
See `data-flow.md`.

---

## 09 — Failure Scenarios
| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| Resolver crash | Health probe | Lost capacity | Remove instance |
| Upstream timeout | Timeout metric | Miss delayed | Bounded retry / alternate server |
| Cache loss | Hit ratio falls | More upstream queries | Warm gradually and limit concurrency |
| DNSSEC failure | Validation metric | SERVFAIL | Check trust anchors and signatures |
| Zone publish failure | Version lag | Old authoritative data | Retry and alert |
| Region outage | Regional probes | Reduced capacity | Withdraw route / shift traffic |

Document trigger, outcome, recovery and recovery metrics.

---

## 10 — Scaling
### 10× traffic
- Add resolver instances, cache capacity and UDP/TCP-aware balancing.
### 100× traffic
- Distribute resolver pools, coalesce duplicate misses and limit upstream concurrency.
- Scale authoritative serving separately from zone writes.
### 1,000× traffic
- Introduce regional anycast sites, redundant transit and regional capacity headroom.
- Monitor route convergence, zone replication and disaster recovery.
### Scaling decision checklist
- Measure CPU, memory, network, cache hit ratio, validation cost and upstream saturation before adding capacity.

---

## 11 — Trade-offs
| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| Cache | Local | Shared | Latency and independence vs reuse |
| TTL | Short | Long | Freshness vs upstream load |
| Transport | UDP | TCP | Low overhead vs reliable larger responses |
| Routing | Regional unicast | Anycast | Simplicity vs global reach |
| Zone distribution | Push | Pull/transfer | Speed vs operational complexity |
| Telemetry | Aggregate | Detailed | Privacy vs diagnostic depth |

### Required decision record format
- Context, options, choice, rationale, gains, sacrifices and revisit conditions.

---

## 12 — Cost
No provider pricing specified; use a cost model.
### Cost categories
| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Resolver compute | Instance-hours | QPS, CPU |
| Cache | Memory | RRsets, overhead |
| Network | Traffic and transit | Bytes, packets |
| Load balancer | Capacity and hours | QPS, connections |
| Zone store | Storage and writes | Zones, versions |
| Observability | Telemetry volume | Metrics and logs |

### Estimation worksheet
- Compute = instances × hourly price × hours.
- Add network, storage, load balancing and telemetry charges using provider quotes. Record region, date, currency and assumptions.

---

## 13 — Security
### Threats and controls
| Threat | Mitigation |
|---|---|
| Forged DNS answers | DNSSEC validation and resolver response matching |
| Resource exhaustion | Rate and concurrency limits |
| Unauthorized zone edits | MFA, authorization, audit logs |
| Privacy leakage | Minimize query logging; encrypted client transport where supported |
| Key compromise | Managed keys and rotation |
| Cross-tenant access | Zone ownership checks |

### Validation rules
- Validate message structure, record types, name lengths and zone ownership.
- Bound recursion, alias depth and retries; never treat bogus DNSSEC data as secure.

---

## 14 — Observability
### Metrics
| Metric | Type | Why it matters |
|---|---|---|
| dns_queries_total | Counter | Traffic |
| dns_latency_ms | Histogram | Performance |
| dns_rcode_total | Counter | Errors and outcomes |
| cache_hit_ratio | Gauge | Cache effectiveness |
| upstream_timeout_total | Counter | Dependency health |
| dnssec_bogus_total | Counter | Validation failures |
| healthy_resolvers | Gauge | Capacity |
| zone_publish_lag | Gauge | Replication health |

### Logs and traces
- Use sampled, privacy-aware telemetry; avoid full query names as metric labels.
### Example alerts
- Elevated SERVFAIL, p95 latency, upstream timeout or zone version lag.

---

## 15 — Simulation
### Learning objective
Observe how query load, cache hits, resolver failures and upstream capacity affect resolution.
### Initial simulation controls
| Control | Default | Suggested range |
|---|---:|---:|
| Incoming query QPS | 1000 | 10–100000 |
| Resolver instances | 3 | 1–20 |
| Per-instance capacity | 500 QPS | 100–5000 |
| Cache enabled | Yes | On / Off |
| Cache hit ratio | 90% | 0–100% |
| Upstream capacity | 2000 QPS | 100–20000 |
| Upstream latency | 20 ms | 1–1000 |
| Routing algorithm | Round Robin | Supported algorithms |
| Failed instances | 0 | 0–N |
| Upstream available | Yes | On / Off |
### Expected outputs
Served/rejected queries, cache hits/misses, upstream demand, latency, health and errors.
### Experiments
Disable cache, increase traffic 10×, fail instances, raise upstream latency, disable upstream and compare routing algorithms.
### Simulation model warning
Simplified model, not measured production performance. See `simulation-spec.md`.

---

## 16 — Design Challenge
### Challenge: Build a DNS service
**Requirements**
- Resolve records, follow delegations, cache with TTL and validate DNSSEC.
- Handle failures and scale query traffic.
**Constraints for the exercise**
- Start with one resolver; evolve to three, inject failure and increase traffic 10×.
**Learner deliverables**
1. Requirements.
2. Architecture diagram.
3. DNS and zone API contracts.
4. Cache and zone data model.
5. Resolution and publishing flows.
6. Capacity estimates.
7. Failure analysis.
8. Trade-off records.
9. Security and observability plan.
**Evaluation prompts**
- How are referrals followed? How does TTL work? What happens on upstream failure? How do regions and zone replicas stay healthy?

---

## 17 — AI Interview
### Interviewer question bank
| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What is DNS? | Names and resource records |
| Q02 | Beginner | Recursive vs authoritative? | Distinct roles |
| Q03 | Beginner | What is TTL? | Cache lifetime |
| Q04 | Intermediate | How do referrals work? | Root, TLD, authoritative |
| Q05 | Intermediate | How is NXDOMAIN cached? | Negative TTL |
| Q06 | Intermediate | When is TCP used? | Truncation and transport |
| Q07 | Intermediate | How does DNSSEC work? | Trust chain and signatures |
| Q08 | Intermediate | What if a resolver fails? | Health checks and redundancy |
| Q09 | Advanced | How do you scale caches? | Locality and coalescing |
| Q10 | Advanced | How does anycast work? | Routing and failover |
| Q11 | Advanced | How do zone updates propagate? | Versions and TTL |
| Q12 | Advanced | How estimate cost? | QPS, memory, transit |

### Follow-up evaluation dimensions
- Correct protocol semantics, clear assumptions, trade-offs, failure handling and no unsupported vendor claims.

---

## 18 — Architecture Score

### Proposed evaluation rubric

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements completeness | 10 | Required user journeys and constraints |
| Scalability | 15 | Traffic estimates, bottleneck analysis, scaling plan |
| Reliability | 15 | Failure handling, redundancy, recovery |
| Latency and performance | 10 | Resolution path, cache strategy, measured assumptions |
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
