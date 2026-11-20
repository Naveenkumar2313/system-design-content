# CS-020 — Design an HTTP Proxy

> **Content status:** Educational reference design. This document proposes a generic HTTP proxy architecture and does not claim to describe a named vendor's private implementation. Workload, latency, and capacity figures are illustrative assumptions.
>
> **Source alignment:** The catalogue identifies this as Case 20 in Category A: Web & Internet Systems. The required structure follows the repository's 18-section case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-020 |
| System name | HTTP Proxy |
| Real-world reference | Squid, Envoy, HAProxy, NGINX, enterprise proxies, privacy proxies, and content filters are examples, not asserted architectures |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 20 |
| Difficulty | Intermediate |
| Short description | A scalable intermediary that accepts client HTTP(S) traffic, authenticates and authorizes it, resolves upstream destinations, forwards or tunnels traffic, applies policy/caching, and exposes operational controls. |
| Primary users | Enterprise clients, browser/application users, network operators, security teams, developers |
| Key concepts | Forward/explicit/transparent proxy, CONNECT tunneling, TLS pass-through/interception, HTTP framing, DNS, connection pools, caching, access control, SSRF, backpressure |
| Related component IDs | CMP-001 (Load Balancer) |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-20 |

### Scope and assumptions

- The primary design is an explicit forward proxy for outbound HTTP/HTTPS traffic. Transparent and reverse proxy behavior are compared, not combined as one ambiguous mode.
- Plain HTTP can be parsed, filtered, cached, and forwarded. HTTPS can be tunneled as opaque bytes with `CONNECT` or intentionally intercepted under enterprise policy.
- The proposed design supports HTTP/1.1 completely and HTTP/2 for selected client/upstream paths where the implementation library permits independent protocol negotiation. HTTP/3/QUIC is future-compatible rather than assumed as the baseline.
- Policy, authentication, routing, and configuration are control-plane services; active connections and pools remain in node memory.
- Upstream destinations are untrusted. DNS, IP, redirect, TLS, content, and rate policy apply.
- All capacity, latency, and cache values are teaching assumptions.

### Proxy type comparison

| Type | Client awareness | Typical use |
|---|---|---|
| Forward/explicit | Client is configured with proxy host/port | Enterprise, privacy, filtering, access control |
| Forward/transparent | Client is not proxy-aware; traffic is redirected at network edge | Network-level inspection/enforcement |
| Reverse proxy | Client believes it contacts the origin, proxy fronts backend | Web tiers, TLS termination, load balancing |
| API gateway | Client uses application API; proxy applies API policy | API auth, routing, quotas |
| VPN | Creates encrypted network tunnel for network traffic | Broad private network access |

---

## 01 — Story

A client application wants to call `https://example.com/products` through an enterprise network. A direct connection bypasses corporate authentication, logging, filtering, and security controls. An HTTP proxy receives the outbound request, determines whether the client may access the destination, resolves and connects to it, applies protocol and cache policy, and forwards the response.

For plain HTTP, the proxy parses the complete request and may inspect or cache it. For HTTPS, the proxy ordinarily receives `CONNECT example.com:443`, connects to the destination, confirms `200 Connection Established`, and then relays encrypted bytes end-to-end. TLS interception is a separate enterprise feature that terminates the client connection and creates a second upstream TLS connection.

### Actors

1. **Client/browser/application:** Sends explicit proxy requests or uses network-transparent redirection.
2. **Proxy listener:** Accepts client TCP/TLS connections and enforces connection limits.
3. **Request parser/protocol handler:** Parses HTTP/1.x or supported HTTP/2 frames.
4. **Authentication/authorization:** Validates user/service credentials and destination policy.
5. **Routing/DNS engine:** Resolves destination, checks IP policy, and selects upstream.
6. **Connection pool manager:** Maintains and reuses safe upstream connections.
7. **Cache:** Stores eligible shared HTTP representations.
8. **Response handler:** Processes status, headers, body, streaming, and cache insertion.
9. **Logging/metrics:** Emits bounded access and operational telemetry.
10. **Configuration/audit:** Applies versioned policy and records changes.

### Core user journey

1. Client sends an HTTP absolute-form request or HTTPS `CONNECT` to the proxy.
2. Listener accepts a bounded connection and starts an idle/request timeout.
3. Parser validates method, target, headers, framing, and protocol limits.
4. Authentication extracts credentials; authorization evaluates destination/user/group policy.
5. Routing engine performs DNS resolution and IP/policy validation.
6. Connection pool selects or creates an upstream TCP/TLS connection.
7. Proxy forwards a method-compatible request and applies hop-by-hop header policy.
8. Response is received, parsed, filtered/cached if eligible, and streamed to client.
9. Persistent connection state is updated or closed according to protocol rules.
10. Access and performance telemetry is emitted with sensitive values redacted.

### Problem boundaries

The design covers proxy listeners, protocol handling, access policy, DNS/upstream connections, caching, TLS tunnel/interception policy, management APIs, performance, and failure. It does not implement a VPN, reverse-load-balancer implementation, identity provider, DNS server, or every HTTP extension.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Accept explicit/transparent client connections | Must |
| FR-02 | Forward HTTP GET/POST/PUT/PATCH/DELETE and supported methods | Must |
| FR-03 | Support HTTPS CONNECT tunneling | Must |
| FR-04 | Support HTTP/1.1 and selected HTTP/2 paths | Must |
| FR-05 | Parse and validate request/response headers and framing | Must |
| FR-06 | Authenticate clients and authorize destinations | Must |
| FR-07 | Resolve DNS and maintain upstream connection pools | Must |
| FR-08 | Apply URL/domain filtering and access policy | Must |
| FR-09 | Cache only explicitly eligible shared responses | Should |
| FR-10 | Apply timeouts, size limits, rate limits, and connection limits | Must |
| FR-11 | Stream bodies with bounded buffers | Must |
| FR-12 | Expose management/configuration APIs | Must |
| FR-13 | Log bounded access/audit/error data with redaction | Must |
| FR-14 | Support transparent TLS pass-through and controlled interception | Configurable |

### Non-functional requirements

| ID | Requirement | Initial target |
|---|---|---|
| NFR-01 | Availability | Illustrative 99.95% monthly per region |
| NFR-02 | p95 latency | Proxy overhead < 20 ms for local policy/cache paths excluding upstream |
| NFR-03 | Throughput | Horizontal scaling from Gbit/s and concurrent connection needs |
| NFR-04 | HTTP efficiency | Reuse safe upstream connections; enforce keep-alive limits |
| NFR-05 | Concurrency | Support tens/hundreds thousands of connections per node based on OS capacity |
| NFR-06 | Security | Strict parsing, TLS validation, SSRF/DNS policy, redaction |
| NFR-07 | Backpressure | Shed or buffer bounded work rather than unbounded memory growth |
| NFR-08 | Cache correctness | Shared cache respects private/public, validator, and `Vary` semantics |
| NFR-09 | Observability | Metrics, structured logs, traces, dashboard and alerts |
| NFR-10 | HA | Stateless nodes, shared config/cache as needed, zone/region recovery |

### Out of scope

VPN tunneling, general reverse-proxy behavior, a complete content security engine, browser trust lifecycle, or support for every HTTP extension.

---

## 03 — Scale

### Workload assumptions

| Input | Assumption |
|---|---:|
| Client nodes | 100,000 |
| Active connections/node fleet | 500,000 |
| Requests/second | 100,000 |
| Average response | 100 KiB |
| Average payload bandwidth | ≈ 80 Gbit/s |
| Peak-to-average factor | 5× |
| Cache hit ratio | 40% for eligible assets |
| DNS cache hit ratio | 95% |
| Pool reuse ratio | 80% |

At 100,000 RPS × 100 KiB, approximate payload throughput is `100,000 × 100 KiB × 8 ≈ 81 Gbit/s`. Five-times peak may approach 400 Gbit/s before protocol overhead and connection efficiency.

Node sizing depends on active sockets, events/second, TLS handshake rate, DNS, cache hit, policy evaluation, logging, and upstream body streaming. Throughput is not a safe proxy for memory/connection capacity.

### Questions for production sizing

- Plain HTTP, CONNECT, or intercepted TLS percentage?
- TLS handshake and session-reuse rates?
- Cacheable response distribution and payload size?
- DNS TTL/availability and upstream pool behavior?
- HTTP/1.1 versus HTTP/2 client/upstream traffic?
- Number of concurrent connections and idle timeout?
- Log volume and retention?
- Required RPO/RTO and regional policy consistency?

---

## 04 — Naive Architecture

### Simplest viable design

A single server listens on a TCP port, accepts a request, parses it, opens a socket to the destination, copies bytes, and closes both connections.

**Components**
- Proxy listener
- HTTP parser
- TCP/TLS upstream client
- Byte copy loop
- Access log

**Request flow**
1. Accept client.
2. Parse request.
3. Resolve destination.
4. Open upstream connection.
5. Send request and stream response.

### Naive architecture diagram

See `architecture.png`.

### Why begin here?

This design demonstrates proxy forwarding and exposes resource limits: no pool, auth, policy, caching, DNS rebinding protection, TLS policy, request framing safety, backpressure, or failover.

---

## 05 — Problems

| Problem | Impact | Improvement |
|---|---|---|
| Open proxy abuse | Unauthorized egress and bandwidth | Authentication, ACL, rate/connections |
| Request smuggling | Parser ambiguity across hops | Strict parser and conflict rejection |
| DNS rebinding | Destination changes after validation | Resolve/connect policy and recheck |
| TLS interception privacy | Endpoint sees enterprise-issued cert | Policy disclosure, scoped trust, audit |
| No connection pool | Excess TCP/TLS cost | Safe upstream pool and keep-alive |
| Incorrect hop headers | Protocol/connection corruption | Strip hop-by-hop headers per RFC |
| CONNECT bypass | Hidden HTTPS traffic | Policy, audit, allow/deny ports |
| Unsafe cache | Private or auth data leakage | Shared/private cache key and `no-store` rules |
| Slow upstream | Client and memory pressure | Timeout, streaming, circuit breaker |
| Resource exhaustion | Proxy node crash | Bounds, backpressure, connection/byte limits |
| Redirect SSRF | Rebinding/private destination | Revalidate follow or response redirect |
| Header bloat | CPU/memory exhaustion | Header count/byte limits |
| No auth revocation | Former clients retain access | Token/config version checks and rotation |

---

## 06 — Architecture Evolution

### V1 — Basic TCP/HTTP forwarder

A node accepts and forwards one request. It does not scale or protect untrusted targets.

### V2 — Protocol parser and connection manager

Strict HTTP parsing, method/header limits, persistent connections, idle timeout, and upstream pool are added.

### V3 — Authentication and authorization

Proxy credentials and destination policies decide whether a request is allowed.

### V4 — DNS, routing, caching, and filtering

Routing engine resolves destination, selects upstream, checks policy, and uses a shared/in-memory cache for eligible content.

### V5 — CONNECT and controlled TLS interception

Pass-through tunnels are supported. Explicit interception is isolated behind enterprise policy and trust configuration.

### V6 — Regional scale and observability

Multiple proxy nodes use external load balancing, shared policy/cache/config, health checks, metrics, and zone failover.

### Proposed evolved architecture

1. Client opens a TCP connection to proxy listener.
2. Listener applies connection/idle limits and protocol identification.
3. Parser validates request line, headers, Host, framing, and method.
4. Auth service validates `Proxy-Authorization`/Bearer credentials.
5. Authorization engine evaluates user/group/IP/domain/URL policy.
6. Routing/DNS engine resolves destination and validates IP.
7. Pool manager obtains/reuses safe upstream connection.
8. Header processor removes hop-by-hop fields and applies proxy headers.
9. Response handler streams status/body and cache updates.
10. Access/metrics/audit services receive redacted telemetry.

### HTTP request target forms

- Forward proxy plain HTTP: absolute-form `GET http://example.com/products HTTP/1.1`.
- Origin server request: origin-form `/products HTTP/1.1`.
- Authority form: `CONNECT example.com:443`.
- Astrisk form: `OPTIONS * HTTP/1.1` for server options.

### Header handling

Strip connection-specific/hop-by-hop fields according to negotiated protocol. Typical hop-by-hop fields include `Connection`, `Proxy-Connection`, `Keep-Alive`, `Transfer-Encoding`, `TE`, and `Upgrade`; actual behavior must follow protocol rules. End-to-end headers such as `Host`, content metadata, and application headers require proxy-specific preservation.

Forwarded identity headers may include `Via`, `Forwarded`, or `X-Forwarded-*`. Define trusted-proxy behavior and prevent clients from spoofing untrusted original values. Never forward credentials such as `Proxy-Authorization` upstream.

---

## 07 — Components

| Component ID | Component | Responsibility |
|---|---|---|
| CMP-001 | Load Balancer | Distributes management/control traffic or fronts proxy pools where applicable |
| TBD | Client Listener | Accepts explicit/transparent proxy connections |
| TBD | DNS Resolver | Resolves destinations with cache/TTL |
| TBD | Request Parser | Strict HTTP parsing and framing |
| TBD | Protocol Handler | HTTP/1.1, HTTP/2, CONNECT behavior |
| TBD | Authentication | Validates proxy credentials |
| TBD | Authorization/Policy | Destination/user/group allow-deny rules |
| TBD | Routing Engine | Destination route and upstream selection |
| TBD | Header Processor | Hop-by-hop/end-to-end handling |
| TBD | TLS Handler | Pass-through or enterprise interception |
| TBD | Connection Pool | Upstream TCP/TLS reuse |
| TBD | Cache | Eligible shared response storage |
| TBD | Response Handler | Status/header/body streaming |
| TBD | Configuration Manager | Versioned policy/config rollout |
| TBD | Monitoring/Audit | Request, TLS, cache, DNS, pool, and access telemetry |

---

## 08 — Data Flow

### Flow A: Plain HTTP forward request

1. Client sends absolute-form request to proxy.
2. Parser extracts method, absolute URI, Host, headers, and body framing.
3. Auth/policy checks user and destination.
4. DNS resolves allowed host and IP.
5. Pool opens/reuses upstream.
6. Proxy sends origin-style request to upstream.
7. Response is parsed and streamed.
8. Cache eligibility is evaluated and headers are forwarded safely.

### Flow B: HTTPS CONNECT

1. Client sends `CONNECT example.com:443`.
2. Proxy authenticates and authorizes host/port.
3. Proxy resolves and validates destination.
4. Proxy opens upstream TCP/TLS as appropriate.
5. Proxy returns `HTTP/1.1 200 Connection Established`.
6. Proxy relays encrypted bytes in both directions without decrypting.
7. Tunnel closes on either side timeout/disconnect.

### Flow C: TLS interception

1. Client sends CONNECT.
2. Enterprise proxy opens upstream TLS and verifies origin certificate.
3. Proxy generates/serves a certificate chain for the destination under enterprise trust.
4. Client terminates TLS to proxy only where the enterprise CA is trusted.
5. Proxy decrypts, applies policy, re-encrypts, and forwards.
6. Limit interception to policy-approved users/networks and disclose/log per governance.

### Flow D: Cache lookup

1. Cache key includes normalized method, scheme/authority, URI, and relevant `Vary`.
2. Fresh entry is returned.
3. Stale entry may revalidate with conditional request.
4. Miss fetches upstream and inserts eligible representation.
5. Private/authorization/error responses are excluded by policy.

### Flow E: Failure

1. DNS/TCP/TLS/connect failure is classified.
2. Circuit state and health metrics are updated.
3. Retry only safe methods/operations or after a policy retry decision.
4. Client receives a proxy 502/504/503 category without leak of upstream secrets.

---

## 09 — Failure Scenarios

| Failure | Detection | Recovery |
|---|---|---|
| Destination downtime | TCP/TLS/HTTP errors | Circuit breaker, alternate IP, bounded retry |
| DNS failure | Resolver timeout/error | Cached TTL, retries, fail error |
| TCP timeout | Connect deadline | Close, retry safe, circuit state |
| TLS handshake failure | TLS alert/certificate | Do not blindly retry; alert and error |
| Proxy node crash | Health/load balancer | Draining, replacement, client reconnect |
| Database outage | Config/API health | In-memory/config fallback policy; fail safe |
| Cache failure | Redis/memory timeout | Origin path; bounded cache misses |
| Network interruption | TCP reset/idle timeout | Close or resume only where protocol safe |
| Resource exhaustion | FD/socket/memory/CPU | Admission control and connection limits |
| Traffic spike | Queue/latency/RPS | Backpressure, autoscale, deny excess |
| Slow upstream | TTFB/read timeout | Stream/cancel and circuit state |
| Invalid framing | Parser conflict | Reject 400/close to prevent smuggling |

Do not automatically retry POST/PUT/PATCH unless an explicit idempotency policy is available.

---

## 10 — Scaling

- Run stateless proxy nodes across zones behind or beside the network load-balancing layer.
- Keep connection pools and active state local.
- Share policy/config and cache only with bounded consistency.
- Scale from active connections, RPS, TLS handshake, upstream sockets, and bandwidth—not RPS alone.
- Partition cache by key/region and avoid a hot-cache dependency.
- Use non-blocking I/O and bounded memory buffers.
- Spread traffic by DNS/ECMP/load balancer/node health.
- Drain gracefully and reject new work before process shutdown.

Traffic spikes require admission control, per-user/domain limits, upstream pool sizing, and autoscaling.

---

## 11 — Trade-offs

| Choice | Benefit | Limitation |
|---|---|---|
| Forward proxy | Central policy/filter | Client/config and endpoint complexity |
| Reverse proxy | Origin shielding/load balancing | Not a general outbound forward proxy |
| Transparent proxy | No client configuration | Routing/identity/TCP redirection complexity |
| CONNECT tunnel | End-to-end encrypted TLS | Proxy cannot inspect application content |
| TLS interception | Policy inspection | Certificate trust/privacy/regulatory risk |
| Shared cache | Better origin offload | Authorization/Vary correctness |
| In-memory cache | Lowest lookup latency | Node affinity/eviction |
| Redis cache | Distributed reuse | Network latency/failure boundary |
| Thread-per-connection | Simple runtime model | Memory/context cost |
| Event loop | High connection density | Blocking operations are dangerous |

### Required decision record format

Document context, alternatives, selected option, security/performance consequences, and revisit trigger.

---

## 12 — Cost

### Cost categories

Proxy compute, upstream bandwidth, DNS, TLS certificates/inspection, cache memory/storage, database/config, logs/traces, load balancing, and multi-zone capacity.

### Estimation worksheet

Let `R` = requests/second, `B` = average response bytes, `C` = concurrent connections, `H` = TLS handshake RPS.

`payload_throughput_gbit_s ≈ R × B × 8 / 10^9`

`monthly_proxy_cost = compute + bandwidth + DNS + cache + TLS tooling + logs + load balancing`

Include capacity for peak `C`, idle connection memory, TLS handshake CPU, and buffer overhead. Provider values are not vendor benchmarks.

---

## 13 — Security

### Open proxy protection

Reject unauthenticated use unless policy intentionally permits public restricted traffic. Authenticate clients and apply IP/user/domain rate and connection limits.

### SSRF and DNS

- Block private/reserved destinations for enterprise fetch modes where policy requires.
- Validate DNS answers and recheck every redirect or socket hop.
- Protect loopback, RFC1918, link-local, metadata, multicast, unspecified, and private IPv6 ranges.
- Pin validated resolution or use resolver/socket integration to reduce rebinding risk.

### Request smuggling

Reject conflicting `Content-Length` and `Transfer-Encoding`, ambiguous chunk extensions, whitespace/framing anomalies, and protocol-specific parser divergence. Configure a consistent parser on every proxy tier.

### Additional controls

- TLS certificate validation by default.
- SAN/hostname checks and controlled interception trust.
- Header count/size limits and sensitive-header redaction.
- Request/response byte limits and decompression limits.
- Connection and bandwidth quotas.
- Secure management API and audit logs.
- DoS protections, health isolation, and graceful failure.

---

## 14 — Observability

### Database and configuration design

### Active/in-memory data

Active connections, upstream pools, route caches, in-flight buffers, DNS cache, and transient circuit state stay in memory or distributed cache with bounded TTL.

### Proxy users

| Field | Purpose |
|---|---|
| user_id | Stable user identity |
| username | Human identifier |
| password_hash / credential_reference | Auth credential |
| role | Privilege class |
| status | Active/disabled |
| created_at | Lifecycle |

### Access policies

| Field | Purpose |
|---|---|
| policy_id | Policy identity |
| user_id/group_id | Scope |
| destination_pattern | Host/domain/URL pattern |
| action | allow/deny/intercept |
| priority | Ordered evaluation |
| created_at | Audit |

### Configurations/logs

Persist configuration name/value/version/status and access log metadata. Partition access logs by time/tenant and archive them. Index by user/destination/time/status.

---

### API and management interface design

### Create proxy user

`POST /api/v1/proxy/users`

```json
{"username":"analyst","role":"enterprise_user","status":"active"}
```

`201 Created`.

### List/update user

`GET /api/v1/proxy/users`

`PUT /api/v1/proxy/users/{userId}` changes role/status.

### Create policy

`POST /api/v1/proxy/policies`

```json
{
  "subject": "group:engineering",
  "destination_pattern": "*.example.com",
  "action": "allow",
  "priority": 100
}
```

### Update configuration

`PUT /api/v1/proxy/config`

```json
{
  "config_name":"upstream_timeouts",
  "configuration_value":{"connect_ms":3000,"read_ms":15000},
  "expected_version":42
}
```

### Metrics/log APIs

- `GET /api/v1/proxy/metrics`
- `GET /api/v1/proxy/logs`

Management APIs are TLS-authenticated, tenant-scoped, and separate from the proxy traffic listener. Status codes include `200/201/400/401/403/404/409/429/503`.

---

### Connection management and performance

- Use non-blocking/event-loop I/O for high connection density.
- Use thread-per-connection only where CPU/complexity characteristics justify memory cost.
- Pool upstream TCP/TLS connections where protocol/authority/credentials are compatible.
- Remove stale/idle connections; set connect/read/idle limits.
- Stream bodies and use bounded buffers; avoid proxying complete large bodies in memory.
- Use `sendfile`/zero-copy only where TLS and protocol transforms permit.
- Tune TCP, socket buffers, file descriptors, and keep-alive.
- Apply backpressure and bounded queues for parser/cache/log work.
- Monitor memory per connection and GC/allocator behavior.

Thread-per-connection favors simplicity but consumes more memory. Event-driven proxies scale connections but require blocking operations to be isolated.

---

### Load balancing and horizontal scalability

- Place a load balancer or ECMP/DNS routing in front of explicit or transparent proxy nodes.
- Scale nodes horizontally across zones.
- Use shared configuration and distributed cache only where consistency is explicitly designed.
- Health check listener, DNS, upstream, and policy dependencies.
- Drain connections during deploy/shutdown.
- Use affinity only where protocol or cache-local behavior requires it.
- Route DNS/configured proxy endpoints regionally.
- Autoscale from active connections, RPS, bandwidth, CPU, and memory.

Include a horizontally scalable diagram for proxy nodes, shared policy, cache, monitoring, and upstream access.

---

### Monitoring and observability

### Metrics

Requests RPS, active connections, successful/failed requests, p50/p95/p99 latency, upstream failures, DNS failures, cache hit ratio, bandwidth, CPU, memory, TLS handshake failures, pool reuse, 400/502/504 rates, and connection limits.

### Logs/traces

Propagate request/trace IDs. Record user/tenant, method, destination host, status, duration, bytes, cache, upstream, circuit state, and parser outcome. Redact authorization, cookies, passwords, URLs containing secrets, and body contents.

### Alerts

Alert on active connection saturation, upstream/DNS/TLS failure, p95/p99 latency, cache collapse, circuit-open state, request-smuggling rejections, memory/FD exhaustion, auth/config failures, and traffic spikes.

---

### Failure handling and disaster recovery

### High availability

- Destination failover only after health verification.
- Retry GET/HEAD or explicitly idempotent methods; avoid blind POST retry.
- Circuit breakers avoid continuously calling failed upstream hosts.
- Graceful shutdown stops new requests and drains accepted work.
- Deploy across zones; restore database/config from versioned backup.
- Cache and DNS failures fall back without becoming a security bypass.

### Disaster recovery

Document RPO/RTO for policy and audit state. Active connections are transient and are not restored. Configuration and credentials need independent backup/rotation and recovery.

---

### Real-world applications

- Enterprise internet access and outbound policy.
- Corporate content filtering.
- Privacy-focused outbound routing.
- Web scraping infrastructure with controlled origin access.
- Content caching for repeated public assets.
- Network monitoring and auditing.
- Security inspection for approved enterprise TLS.
- Development/testing of browser or application proxy settings.
- Outbound traffic control and tenant isolation.

---

### Technology stack

| Area | Options | Fit |
|---|---|---|
| Language | Go, Rust, Java, C++ | Go/Rust strong for concurrency; Java/C++ mature ecosystems |
| HTTP network | Envoy, NGINX, Squid, custom library | Select by feature and operational fit |
| Database | PostgreSQL/MySQL | Policies, users, config, audit |
| Cache | Redis/in-memory | Redis distributed; memory lowest latency |
| Metrics | Prometheus/Grafana | Dashboard and alerting |
| Deployment | Docker/Kubernetes | Node scaling and rollout |
| External load | HAProxy/NGINX/cloud LB | Listener distribution |

**Squid:** caching and classic forward proxy operations. **Envoy:** modern HTTP filtering/L7 extensions. **HAProxy:** TCP/HTTP load balancing and proxying. **NGINX:** reverse proxy/content delivery plus forwarding with limits.

---

### Architecture diagrams

Create these as separate PNG/Mermaid files or diagram sections:

1. High-level HTTP proxy architecture.
2. Forward proxy absolute-form request flow.
3. HTTP request lifecycle.
4. HTTPS CONNECT tunnel.
5. TLS interception.
6. DNS resolution/upstream pool.
7. Cache HIT/MISS.
8. Authentication/access-control flow.
9. Connection pooling.
10. Horizontal scaling and high availability.

Each diagram should have a title, labeled components, directional arrows, and a short explanation.

---

## 15 — Simulation

### Learning objective

Demonstrate forward proxy forwarding, HTTPS CONNECT, TLS interception policy, DNS, caching, connection pooling, authentication, request-smuggling rejection, traffic spikes, and node failover.

### Initial simulation controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| requests_per_second | Requests/s | Slider | 100000 | 1 | 1000000 |
| active_connections | Active connections | Slider | 500000 | 1 | 10000000 |
| average_response_kib | Average response | Slider | 100 | 1 | 102400 |
| pool_reuse_ratio | Upstream pool reuse | Slider | 0.80 | 0 | 1 |
| cache_hit_ratio | Cache hit ratio | Slider | 0.40 | 0 | 1 |
| dns_cache_hit_ratio | DNS cache hit ratio | Slider | 0.95 | 0 | 1 |
| connect_request_share | CONNECT share | Slider | 0.30 | 0 | 1 |
| tls_interception_share | Interception share | Slider | 0.10 | 0 | 1 |
| upstream_failure_probability | Upstream failure | Slider | 0.01 | 0 | 1 |
| max_connections_per_node | Node connection limit | Slider | 100000 | 100 | 1000000 |

### Required scenarios

1. Forward an absolute-form HTTP GET.
2. Establish a pass-through HTTPS CONNECT tunnel.
3. Show enterprise TLS interception and re-encryption.
4. Compare cache HIT, MISS, revalidation, and `304`.
5. Measure pool reuse under cold versus warm upstreams.
6. Inject DNS failure and rebinding.
7. Reject conflicting `Content-Length` and `Transfer-Encoding`.
8. Block private/metadata destinations.
9. Fail an upstream and observe circuit-breaker state.
10. Saturate connections and apply backpressure.
11. Scale proxy nodes and drain a node.

### Acceptance checks

- Plain HTTP destination comes from the absolute URI/Host consistently.
- Pass-through CONNECT does not decrypt TLS.
- Interception is explicitly enabled and trust-controlled.
- Private destinations and rebinding are blocked.
- Smuggling-conflict messages are rejected.
- Cache excludes private/auth/error responses by policy.
- Pool reuse does not mix authority, TLS credentials, or protocol context.
- Simulation values are estimates, not measured provider results.

---

## 16 — Design Challenge

### Challenge: Build a secure and scalable HTTP proxy

Design an explicit forward proxy for an enterprise network that handles HTTP/1.1 and supported HTTP/2 traffic, HTTPS CONNECT, optional controlled TLS interception, access policy, DNS safety, caching, connection pools, and horizontal scale.

**Requirements**
- Support absolute-form HTTP and CONNECT.
- Authenticate clients and authorize destinations.
- Apply strict HTTP framing and header handling.
- Resolve DNS and prevent rebinding/private-target access.
- Reuse upstream connections and stream response bodies.
- Support cacheable public responses without leaking private data.
- Implement management APIs for users, policies, configuration, metrics, and logs.
- Add TLS, smuggling, open-proxy, DoS, and sensitive-log defenses.
- Deploy multiple nodes with health checks, failover, backpressure, and graceful drain.

**Constraints**
- Do not pass proxy credentials upstream.
- Do not decrypt CONNECT traffic unless interception is explicitly configured.
- Reject conflicting request framing.
- Retry only safe or explicitly idempotent operations.
- Keep active connections and pools in memory or bounded distributed state.
- Preserve exact protocol semantics for streaming, chunked, and keep-alive responses.

**Learner deliverables**
1. Functional/non-functional requirements and workload assumptions.
2. Naive and evolved architecture diagrams.
3. HTTP, CONNECT, and cache flows.
4. Access policy and TLS trust model.
5. Database/configuration/API design.
6. Connection pool, pool reuse, and backpressure design.
7. Failure, HA, observability, and cost plans.
8. A measurable simulation.

**Evaluation prompts**
- How is absolute-form forwarding different from origin-form?
- Why is CONNECT a tunnel rather than a normal HTTP request?
- Which headers are hop-by-hop, and why are they removed?
- How does strict framing prevent request smuggling?
- How does pool reuse avoid protocol/authority mixing?
- When is TLS interception appropriate, and what are its privacy implications?
- How does DNS rebinding affect destination validation?
- How do cache authorization and `Vary` prevent data leakage?

---

## 17 — AI Interview

| ID | Difficulty | Question | Strong answer |
|---|---|---|---|
| Q01 | Beginner | What is an HTTP proxy? | Client-to-upstream intermediary for HTTP or CONNECT |
| Q02 | Beginner | Forward proxy vs reverse proxy? | Outbound client proxy vs origin-facing frontend |
| Q03 | Intermediate | What is absolute-form? | Proxy receives full URI in plain HTTP request |
| Q04 | Intermediate | How does CONNECT work? | Tunnel TCP to host:port and return 200 |
| Q05 | Intermediate | Pass-through vs interception? | Relay opaque bytes vs terminate/re-encrypt |
| Q06 | Advanced | How do hop-by-hop headers work? | Strip and consume per protocol negotiation |
| Q07 | Advanced | How does pooling preserve safety? | Reuse compatible authority/TLS/client context |
| Q08 | Advanced | How can request smuggling occur? | Conflicting framing causes parser divergence |
| Q09 | Advanced | How does DNS rebinding defeat validation? | Answer changes after validation or before socket |
| Q10 | Advanced | What cache responses are private? | Authorization, Set-Cookie, no-store, or policy-sensitive |

---

## 18 — Architecture Score

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements and scale | 10 | RPS, connections, bandwidth, protocols |
| Architecture and data flow | 15 | Listener, parser, policy, DNS, upstream |
| Protocol correctness | 15 | Framing, headers, pooling, HTTP versions |
| HTTPS and TLS | 15 | CONNECT, pass-through, interception |
| Security | 15 | SSRF, smuggling, auth, redaction |
| Scaling/performance | 15 | Event loop, pool, cache, horizontal |
| Observability/recovery | 10 | Metrics, logs, failover |
| Cost/trade-offs | 5 | Bandwidth, nodes, cache, TLS |
| **Total** | **100** | |

Weighted score = sum of (dimension score / 4 × dimension weight).
