# CS-006 — Design a Reverse Proxy

> **Content status:** Educational reference design. This document proposes a generic reverse-proxy service; it does not claim to describe any vendor's private production architecture.
>
> **Source alignment:** The Codorythm case-study catalogue identifies this as Case Study 6 in Category A: Web & Internet Systems. The required structure follows the 18-section case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-006 |
| System name | Reverse Proxy |
| Real-world reference | NGINX, HAProxy, and Envoy are implementation examples, not asserted system designs |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 6 |
| Difficulty | Beginner to Intermediate |
| Short description | Design a highly available HTTP reverse-proxy service that accepts client traffic, terminates TLS, applies request policy, routes to healthy origins, and relays responses. |
| Primary users | Web and API clients, application teams, platform operators, security and reliability engineers |
| Key concepts | HTTP, DNS, reverse proxy, routing, load balancing, TLS termination, caching, rate limiting, health checks, high availability, observability |
| Related component IDs | CMP-001 (Load Balancer), used as an optional front door for the redundant proxy tier |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-14 |

### Scope and assumptions

- The service accepts public HTTP/HTTPS traffic for registered hostnames and forwards it to configured private origins.
- The primary example is a Layer 7 HTTP reverse proxy. TCP pass-through is an optional mode and cannot apply HTTP-level policy to encrypted traffic it does not terminate.
- The proxy may terminate client TLS and may separately use TLS or mutual TLS to origins.
- Routes, origin pools, policies, and certificates are provisioned by trusted operators through a validated control/configuration workflow.
- The design includes routing, optional response caching, rate limits, health-aware failover, and telemetry.
- Workload values are illustrative design assumptions, not vendor benchmarks or real production disclosures.
- The design begins in one region; multi-region ingress and policy synchronization are later extensions.

---

## 01 — Story

An organization operates several web applications and APIs. Each service needs a public endpoint, encrypted client connections, traffic routing, and consistent edge controls. Exposing every application server directly makes certificates, routing, access policy, and operational changes difficult to manage. A reverse proxy provides a controlled entry point: clients connect to the proxy, while the proxy selects and contacts a private origin on their behalf.

The design challenge is to make that intermediary reliable without turning it into an unbounded bottleneck or a source of security and availability failures. The proxy must handle normal request/response traffic, detect failed origins, avoid unsafe retries and cache reuse, and remain available if a proxy process or host fails.

### Actors

1. **Client:** Sends an HTTP request to a public service hostname.
2. **Application owner:** Registers hostnames, routes, origins, and service policies.
3. **Platform operator:** Deploys proxy capacity, certificates, and validated configuration.
4. **Origin service:** Handles requests forwarded by the proxy from a private network.
5. **Security / reliability operator:** Monitors abuse, errors, capacity, and policy violations.

### Core user journey

1. An application owner registers `api.example.com`, an origin pool, and a route policy.
2. A validated configuration is distributed to the active proxy instances.
3. A client resolves `api.example.com` and establishes HTTPS to the public entry point.
4. The proxy validates the TLS connection, host, and request; it selects the configured route.
5. The proxy applies policy and chooses a healthy origin.
6. The origin returns a response; the proxy relays it to the client.
7. Operators inspect request outcomes, backend health, latency, and capacity through telemetry.

### Problem boundaries

This case study focuses on a reverse-proxy data path and the supporting configuration, health, scaling, and observability design. It does not design the applications behind the proxy, a complete identity provider, a CDN, or a full API-management product. Authentication may be enforced by an origin or a separately designed identity-aware gateway; basic proxy routing alone is not authorization.

### Common application scenarios

- Host multiple websites or API services behind a small set of public entry points.
- Route requests by hostname or URL path to independently deployed application services.
- Terminate client TLS centrally while maintaining a separately secured connection to private origins.
- Provide an ingress tier for containerized services, where routes and endpoint membership change over time.
- Cache approved public content and apply consistent request limits at the service edge.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Accept HTTP and HTTPS connections for registered hostnames | Must |
| FR-02 | Select routes using configured host and path rules | Must |
| FR-03 | Forward requests to configured origin pools and relay responses | Must |
| FR-04 | Support origin selection policies including Round Robin, Weighted Round Robin, Least Connections, and IP Hash where supported | Must |
| FR-05 | Remove unhealthy origins from new-request selection and restore them after recovery criteria are met | Must |
| FR-06 | Terminate client TLS and support separately configured encrypted origin connections | Must |
| FR-07 | Configure connection, request, and upstream timeouts | Must |
| FR-08 | Apply configurable per-client or per-route rate and burst limits | Should |
| FR-09 | Cache only explicitly eligible responses according to HTTP cache directives and service policy | Should |
| FR-10 | Preserve or set trusted forwarding metadata for origins | Must |
| FR-11 | Update routes and certificates through validated, observable configuration rollout | Should |
| FR-12 | Emit request, route, origin-health, and error telemetry | Must |
| FR-13 | Support safe connection draining when an origin or proxy instance is removed | Should |
| FR-14 | Provide controlled errors when no route or healthy origin is available | Must |

### Non-functional requirements

| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Proxy-added latency | Illustrative p95 under 10 ms in-region for a cache-bypass request, excluding client and origin processing |
| NFR-02 | Availability | Illustrative 99.95% monthly for the proxy service entry path, subject to upstream entry and origin availability |
| NFR-03 | Capacity | Sustain the assumed peak after loss of one proxy instance and one origin, with explicit headroom |
| NFR-04 | Correctness | Route only registered hosts and configured paths; never select an unhealthy origin after its state is observed |
| NFR-05 | Security | Encrypt client traffic, validate origin TLS when enabled, restrict management access, and reject malformed or ambiguous HTTP |
| NFR-06 | Isolation | Keep origin services private where proxy policy is required; separate tenant configuration and access |
| NFR-07 | Scalability | Scale proxy capacity independently from origin capacity and avoid global coordination on each request |
| NFR-08 | Observability | Attribute latency and failures to route, proxy, and origin without high-cardinality or sensitive labels |
| NFR-09 | Operability | Validate configuration before rollout and support health-gated rollout and rollback |

### Out of scope for the initial version

- Full API gateway features such as developer portals, quotas by subscription plan, and API lifecycle management.
- General-purpose forward proxying to arbitrary destinations.
- A complete web application firewall or bot-management product.
- Global anycast routing and active-active multi-region control-plane consistency.
- Application-specific authentication and authorization implementation.
- Guaranteed cache consistency across regions.

---

## 03 — Scale

All values in this section are **illustrative workload assumptions for teaching and simulation**. They are not vendor performance figures.

### Workload assumptions

| Metric | Illustrative value | Reasoning |
|---|---:|---|
| Registered hostnames | 500 | Several application teams and environments |
| Average request rate | ~1,157 RPS | 100,000,000 requests/day / 86,400 seconds |
| Peak request rate | 10,000 RPS | Approx. 8.6× average for bursty web traffic |
| Average request bytes | 8 KiB | Headers and small request bodies, simplified |
| Average response bytes | 32 KiB | Mixed API and web responses, simplified |
| Peak aggregate traffic | ~3.28 Gbit/s | 10,000 × (8 + 32) KiB × 8, decimal bits/sec |
| Proxy instances | 4 at 5,000 RPS tested capacity each | 20,000 RPS nominal; 15,000 RPS after losing one |
| Origin instances | 4 at 4,000 RPS tested capacity each | 16,000 RPS nominal; 12,000 RPS after losing one |
| Cacheable response share | 30% of requests | Illustrative; actual eligibility depends on route and HTTP policy |
| Proxy-added latency objective | p95 < 10 ms | Regional objective excluding origin processing |
| Availability objective | 99.95% monthly | Learning target; requires all dependencies and entry routing to be considered |

### Capacity notes

- The proxy tier's tested per-instance capacity must include TLS handshake rate, active connections, request mix, logging, and response size; an RPS-only benchmark is insufficient.
- Four proxy instances at the assumed per-instance capacity retain 15,000 RPS after one instance failure. This gives 50% capacity above the 10,000 RPS peak assumption.
- Four origins retain 12,000 RPS after one origin failure under the assumed capacity. Actual capacity is service-specific and must be measured.
- The estimated aggregate network throughput is a planning input, not a transfer-cost or network-interface guarantee.
- A cache hit reduces origin work only for eligible requests and may shift cost to proxy storage, memory, disk, and egress.
- Capacity calculations need headroom for connection churn, TLS handshakes, retries, health checks, configuration reloads, and failure recovery.

### Questions for a production sizing exercise

- What is the distribution of request methods, body sizes, response sizes, and connection lifetimes?
- What percentage of requests use HTTP/2 or HTTP/3, streaming, WebSockets, or long-lived connections?
- What TLS handshake rate, cipher policy, and client-certificate requirements are expected?
- Which routes are cacheable, and what are their freshness, privacy, and invalidation requirements?
- What failure objective and N+1 capacity must be met for each proxy and origin pool?
- What are the geographic, data-residency, and origin-network constraints?

---

## 04 — Naive Architecture

### Simplest viable design

A single reverse-proxy process accepts HTTPS traffic, terminates TLS, and forwards every registered request to one origin server.

**Components**
- Client/browser
- DNS record for the public hostname
- Single reverse-proxy instance
- Single origin application
- TLS certificate and private key

**Request flow**
1. Client resolves the service hostname to the proxy address.
2. Client establishes HTTPS to the proxy.
3. Proxy validates the hostname and forwards the HTTP request to the origin.
4. Origin returns a response to the proxy.
5. Proxy relays the response to the client.

### Naive architecture diagram

See `architecture.png`.

### Why begin here?

The single-proxy design isolates the core reverse-proxy responsibility: accepting a client request and forwarding it to an origin. It makes TLS termination, host handling, and response relay easy to demonstrate before introducing availability, traffic distribution, caching, and configuration management.

---

## 05 — Problems

| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Single proxy failure | One process or host serves all traffic | Complete service outage | Multiple proxy instances across failure domains |
| Single proxy bottleneck | TLS, network, CPU, and connection load concentrate on one host | Queueing, latency, and connection rejection | Capacity-tested horizontal proxy tier |
| Single origin failure | One backend serves every route | Requests fail even when proxy is healthy | Redundant origin pool with health-aware routing |
| Direct origin exposure | Backend accepts public traffic | Clients bypass proxy controls and reveal topology | Private origin network and explicit ingress allowlist |
| Static routing mistakes | Manual configuration has no validation or rollout policy | Wrong service receives traffic or configuration reload fails | Schema validation, staged rollout, and rollback |
| Certificate expiry | Certificate lifecycle is manual or unobserved | HTTPS handshakes fail | Automated renewal, protected key storage, expiry alerts |
| Untrusted forwarding headers | Proxy passes client-supplied identity headers unchanged | Origins may trust spoofed client IP or scheme | Strip/overwrite headers at a trusted boundary |
| Origin overload | Retries, slow clients, or unbounded buffering increase work | Cascading latency and resource exhaustion | Bounded retries, timeouts, concurrency, and buffers |
| Unsafe caching | Personalized or private responses share a cache key | Data leakage or stale content | Conservative eligibility and HTTP cache semantics |
| Rate-limit bypass | Counters are local or identity is spoofable | Limits are inconsistent or evadeable | Trusted identity and documented per-instance/distributed limits |
| Health-check mismatch | Probe tests only process liveness or is too aggressive | Broken origins remain active or healthy ones flap | Readiness-aware probes, thresholds, and alerting |
| HTTP parser disagreement | Proxy and origin interpret framing differently | Request smuggling or unexpected request boundaries | Normalize and reject malformed requests; keep parsers patched |

---

## 06 — Architecture Evolution

These are proposed design stages, not a historical account of any named product's deployment.

### V1 — Single reverse proxy and origin

- One Layer 7 proxy terminates TLS and forwards to one origin.
- Establishes virtual-host, route, timeout, and response behavior.
- Suitable for a local demonstration or low-risk prototype.
- Limitation: proxy and origin are both single points of failure.

### V2 — Redundant proxy tier and origin pools

- Run multiple stateless proxy instances across hosts or availability zones.
- Use CMP-001 Load Balancer or an equivalent resilient Layer 4 entry point to distribute client connections across proxy instances.
- Configure multiple origins per route and health-aware selection.
- Keep origin services private and accept traffic only from the proxy tier.
- Benefit: a single proxy or origin failure does not stop all service.
- New concern: configuration and certificate consistency across instances; health state may differ briefly.

### V3 — Edge policy and bounded acceleration

- Add per-route TLS and upstream TLS policy, request-size limits, timeouts, and rate limits.
- Add response caching only for approved public, cacheable responses.
- Use connection reuse and protocol-aware buffering with explicit resource limits.
- Apply controlled failover and only retry requests when method and failure semantics make it safe.
- Benefit: reduces repeated origin work and centralizes common policy.
- New concern: cache privacy/freshness, distributed rate-limit semantics, and retry amplification.

### V4 — Managed configuration and operational resilience

- Separate the request data plane from the configuration/control plane.
- Validate route, certificate, and origin-pool configuration before distribution.
- Roll changes gradually, observe health and error budgets, then complete or roll back.
- Use secrets management, certificate rotation, audit logs, and versioned configuration.
- Benefit: safer fleet-wide operations and faster recovery from bad configuration.
- New concern: control-plane availability, stale configuration, and partial rollout behavior.

The proposed evolved architecture is shown in `architecture-evolved.png`.

### Practical implementation examples

The following fragments illustrate common configuration shapes. Replace example addresses, hostnames, certificate paths, and limits; validate against the exact product release before deployment. They are not a complete production security baseline.

**NGINX HTTP reverse proxy**

```nginx
http {
    upstream api_origins {
        # Round Robin is the default when no balancing directive is set.
        server 10.0.1.11:8080 weight=1 max_fails=3 fail_timeout=10s;
        server 10.0.1.12:8080 weight=2 max_fails=3 fail_timeout=10s;
        server 10.0.1.13:8080 weight=1 max_fails=3 fail_timeout=10s;
        keepalive 64;
    }

    limit_req_zone $binary_remote_addr zone=api_per_client:10m rate=100r/s;
    limit_req_status 429;

    proxy_cache_path /var/cache/nginx/api
        levels=1:2
        keys_zone=api_public:20m
        max_size=1g
        inactive=30m
        use_temp_path=off;

    server {
        listen 80 default_server;
        server_name _;
        return 444;
    }

    server {
        listen 443 ssl default_server;
        server_name _;
        ssl_reject_handshake on;
    }

    server {
        listen 80;
        server_name api.example.com;
        return 301 https://api.example.com$request_uri;
    }

    server {
        listen 443 ssl;
        server_name api.example.com;

        ssl_certificate     /etc/nginx/tls/api-fullchain.pem;
        ssl_certificate_key /etc/nginx/tls/api-private.key;
        ssl_protocols TLSv1.2 TLSv1.3;

        limit_req zone=api_per_client burst=200 nodelay;

        location / {
            proxy_pass http://api_origins;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 2s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
            proxy_next_upstream error timeout http_502 http_503 http_504;
            proxy_next_upstream_tries 2;
        }

        location /public/ {
            proxy_pass http://api_origins;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_cache api_public;
            proxy_cache_methods GET HEAD;
            proxy_cache_key "$scheme|$request_method|$host|$request_uri";
            proxy_cache_valid 200 60s;
            proxy_cache_bypass $http_authorization $http_cookie;
            proxy_no_cache $http_authorization $http_cookie
                           $upstream_http_set_cookie;
            proxy_cache_lock on;
        }
    }
}
```

The origin pool above uses weighted Round Robin by default; weights are relative shares, not guaranteed capacity. To select Least Connections for that pool, add `least_conn;`. For client-address affinity, use `ip_hash;` in a separately configured upstream group. Do not treat IP Hash as durable session storage. `max_fails` and `fail_timeout` provide passive failure handling in the open-source upstream model, not an active readiness probe; health-check features vary by product and edition. The rate-limit zone is shared among workers of one NGINX instance, not automatically across a proxy fleet. The TLS default-server example uses `ssl_reject_handshake`, available in NGINX 1.19.4 and later.

The sample overwrites the forwarded client-address header for a direct client connection. If a trusted load balancer sits in front, configure the exact trusted source ranges and the appropriate source-address propagation before using client IP for security decisions. If origins are not on a trusted network, configure upstream TLS with certificate-chain and hostname verification instead of forwarding over HTTP. Configure caching only for routes whose methods, authorization state, cookies, `Vary` dimensions, and origin cache directives permit shared reuse.

For an encrypted origin hop, use a TLS-enabled upstream and verify the origin identity. This fragment assumes every origin certificate contains the shared internal name `api-origin.internal`:

```nginx
upstream api_origins_tls {
    server 10.0.1.11:8443;
    server 10.0.1.12:8443;
}

location / {
    proxy_pass https://api_origins_tls;
    proxy_ssl_server_name on;
    proxy_ssl_name api-origin.internal;
    proxy_ssl_verify on;
    proxy_ssl_trusted_certificate /etc/nginx/ca/origin-ca.pem;
    proxy_ssl_verify_depth 2;
}
```

**HAProxy HTTP frontend and origin pool**

```haproxy
global
    maxconn 20000

defaults
    mode http
    option httplog
    timeout connect 2s
    timeout client 30s
    timeout server 30s

frontend public_https
    bind :443 ssl crt /etc/haproxy/certs/api.pem alpn h2,http/1.1
    http-request deny deny_status 404 unless { hdr(host) -i api.example.com api.example.com:443 }
    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-For %[src]
    default_backend api_origins

backend api_origins
    balance leastconn
    option httpchk GET /readyz
    server app1 10.0.1.11:8080 check
    server app2 10.0.1.12:8080 check weight 2
    server app3 10.0.1.13:8080 check
```

This example terminates TLS at the frontend and uses an HTTP health check. For an untrusted backend network, enable and verify TLS on each server connection with the appropriate CA and hostname settings. A production deployment also needs an HTTP listener policy, default-host behavior, certificate renewal, protected management access, rate-limit policy, backend network restrictions, metrics/logging, and validated reload or rollout procedures.

Envoy expresses listener, TLS, HTTP filter, route, cluster, load-balancing, and health-check policy as structured configuration. Its HTTP connection manager handles HTTP routing and filters; cluster configuration selects upstream endpoints and can enable active health checks. Keep configuration validation and rollout controls separate from the request data path.

### Global evolution

- Place regional proxy fleets behind an appropriately designed global entry/routing layer.
- Replicate route and certificate configuration through a versioned control plane.
- Keep the data path functional with the last known-good configuration during temporary control-plane loss.
- Define regional health and failover policy, DNS/anycast convergence, origin locality, and data-residency constraints.
- Decide whether caches are regional, shared, or disposable; do not imply globally consistent cache state without a mechanism.
- Test region failure, certificate rotation, route rollout, and origin failover together.

---

## 07 — Components

| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Distributes incoming connections across healthy reverse-proxy instances in the evolved architecture | V2 onward |
| TBD | Client | Sends requests and consumes proxy responses | Yes |
| TBD | DNS | Resolves public service hostnames to the entry point | Yes |
| TBD | Reverse Proxy Data Plane | Terminates or passes through connections, routes requests, and relays responses | Yes |
| TBD | Origin / Backend Pool | Runs the applications that serve proxied requests | Yes |
| TBD | TLS Certificate / Secret Management | Issues, distributes, protects, and rotates certificates and trust roots | HTTPS |
| TBD | Health Checking / Service Discovery | Tracks origin membership and readiness | Recommended |
| TBD | Response Cache | Stores eligible responses under defined HTTP cache policy | Optional |
| TBD | Rate Limiter | Enforces route/client request budgets and burst behavior | Recommended |
| TBD | Configuration Control Plane | Validates and distributes routes, pools, and policy | Recommended |
| TBD | Monitoring / Observability | Collects request, origin, proxy, and configuration telemetry | Recommended |

**Component ID policy:** CMP-001 is the only related component with a confirmed ID in the starter package. Other IDs remain TBD until their component specifications are created and added to the master component index. The reverse proxy is the system being designed in this case study; this assignment does not create a component-library entry for it.

### Component interactions

- Client → DNS → Load Balancer / public entry point → Reverse Proxy Data Plane.
- Reverse Proxy → selected healthy origin; origin response returns through the proxy.
- Health checker / discovery → origin pool state used by the proxy.
- Configuration Control Plane → validated route and policy snapshots on proxy instances.
- Reverse Proxy ↔ Response Cache for eligible requests and responses.
- Reverse Proxy → Monitoring for metrics, logs, traces, and audit events.
- Certificate / Secret Management → proxy instances through protected distribution.

---

## 08 — Data Flow

### Flow A: Client request and origin response

1. Client resolves the public hostname and connects to the entry point.
2. Entry point selects a healthy reverse-proxy instance.
3. Proxy negotiates client TLS when TLS termination is configured and validates the requested hostname.
4. Proxy rejects malformed requests, unknown hosts, and disallowed routes according to policy.
5. Proxy applies rate limits and checks the cache only when the route and request are eligible.
6. On cache miss, proxy selects a healthy origin according to the route's routing policy.
7. Proxy forwards the request with normalized forwarding headers; upstream TLS verification is performed if configured.
8. Origin response returns to the proxy; proxy applies response/cache policy and relays the response.
9. Proxy records status, route, timing, and origin outcome without logging sensitive payloads by default.

### Flow B: Health check and origin failover

1. Health subsystem probes each configured origin or observes passive connection/response failures.
2. Health state changes only after configured thresholds and timeouts are met.
3. Proxy stops assigning new requests to an unhealthy origin.
4. Existing in-flight requests follow the configured timeout/drain behavior; they are not assumed to be safely replayable.
5. A recovered origin must satisfy recovery criteria before rejoining the pool.
6. Telemetry records state transition and remaining pool capacity.

### Flow C: Route configuration rollout

1. Operator submits a versioned route, pool, or policy change to the control plane.
2. Control plane validates schema, host ownership, route precedence, origin reachability policy, and certificate references.
3. A canary proxy receives the configuration and reports successful load and health.
4. Control plane gradually distributes the version to the fleet.
5. Error rate, route outcomes, and instance health are compared with the prior version.
6. Rollout completes or automatically/manually rolls back to the last known-good version.
7. During control-plane outage, data-plane instances continue with the last accepted configuration, subject to certificate and route expiry policy.

### Flow D: Cache lookup

1. Proxy determines whether the method, request headers, route, and authorization state permit shared caching.
2. Proxy constructs a cache key from the scheme, host, normalized URI, and any required representation dimensions.
3. On a fresh hit, proxy returns the stored response without contacting the origin.
4. On miss or revalidation requirement, proxy requests the origin and follows applicable HTTP cache directives.
5. Proxy stores a response only if policy and response directives allow it; private or authenticated responses are not shared by default.
6. Invalidation, TTL expiry, and stale serving follow the documented route policy.

### Error and edge-case paths

- Unknown hostname or route: reject with a controlled 404/421 policy; never route to an arbitrary host.
- No healthy origin: return a controlled 503 and emit an availability alert.
- Upstream connection failure: return a gateway error or make a bounded, safe retry.
- Upstream deadline exceeded: return 504; do not wait indefinitely.
- Invalid upstream response: return 502 and record protocol/error context.
- Client rate limit exceeded: delay or return 429 according to configured burst policy.
- TLS validation failure to origin: fail closed; do not disable certificate validation.
- Cache contains expired or disallowed response: revalidate or bypass cache.
- Control-plane unavailable: keep last known-good data-plane configuration and expose its version/age.
- Client disconnects during streaming: cancel upstream work when safe and release resources.

---

## 09 — Failure Scenarios

| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| One proxy instance crashes | Entry-point health check or process telemetry fails | Connections to that instance fail until removed | Drain/remove instance; remaining fleet handles traffic within N+1 capacity |
| Entire proxy tier unavailable | Synthetic checks and entry health fail | Public service unavailable | Restore proxy capacity; fail over entry point or region if designed |
| One origin fails | Active probe or passive failure threshold | Requests assigned to origin may fail during detection | Stop new selection after detection; retain adequate remaining capacity |
| All origins unhealthy | Pool health state and elevated 503s | Dynamic requests cannot be served | Return controlled 503; alert owners; restore origin or approved failover |
| Bad route configuration | Validation error, canary errors, route-level 5xx spike | Wrong service, failed requests, or exposure | Reject before rollout; canary and rollback versioned configuration |
| Control plane unavailable | Heartbeat/config age alert | New changes cannot reach fleet | Continue using last known-good snapshot; restore control plane |
| Certificate expires or key is invalid | Expiry monitor and TLS handshake failures | Client connections fail | Automated renewal, key protection, pre-expiry alerts, staged rotation |
| Origin certificate validation fails | TLS handshake error and upstream telemetry | Proxy cannot establish secure origin connection | Correct trust chain/hostname and rotate safely; never bypass verification |
| Proxy overload | CPU, connection, queue, memory, or network saturation | Higher latency, rejections, and timeouts | Shed load, enforce limits, scale fleet, reduce expensive work |
| Retry storm | Retry counters and upstream RPS rise | Origin overload worsens | Bound retries, use backoff/jitter where appropriate, retry only safe operations |
| Cache serves private/stale response | Cache audit, application report, policy mismatch | Cross-user disclosure or incorrect content | Disable affected cache route, purge entries, correct key/eligibility, investigate |
| Rate-limit state is inconsistent | Per-instance counters or fleet traffic mismatch | Limits vary with instance count | Document local limits or adopt coordinated state; tune for fleet size |
| Network partition to origin | Connect failures, packet loss, timeout rise | Requests fail despite origin process health | Route to reachable pool, avoid tight retry loops, repair network path |
| HTTP parsing discrepancy | Security testing or anomalous request behavior | Request smuggling or boundary confusion | Reject ambiguous framing, normalize, patch, and align proxy/origin parsing |
| Telemetry backend fails | Export errors and dropped telemetry | Reduced diagnosis, possible proxy overhead if blocking | Use bounded non-blocking export buffers; alert on telemetry loss |

For HTTP deployments, use 502 for unusable upstream responses, 503 for unavailable service or deliberate overload shedding, and 504 for upstream timeout when appropriate. Exact status behavior is product- and policy-specific. Track whether an error originated at the proxy, entry point, or application.

---

## 10 — Scaling

### 10× traffic

- Add proxy instances behind the resilient entry point and verify that per-instance tested capacity includes TLS, response size, and connection mix.
- Keep at least one instance of N+1 capacity after a host/zone failure.
- Scale origins independently; proxy capacity does not create application capacity.
- Use upstream keep-alive and connection pooling when compatible with origin behavior.
- Introduce bounded response caching for safe, public, frequently repeated content.
- Monitor network throughput, TLS CPU, open connections, queueing, and p95/p99 latency.

### 100× traffic

- Partition proxy fleets by region, tenant, or workload where isolation and independent scaling justify the complexity.
- Use service discovery or versioned control-plane updates to manage large dynamic origin pools.
- Protect origins from connection storms and cache stampedes with bounded concurrency, cache request coalescing, and load shedding.
- Use local rate limits for fast rejection and decide explicitly whether global quotas require distributed coordination.
- Reduce expensive per-request work and sample traces while retaining errors and tail-latency visibility.
- Keep control-plane rollout rate below the fleet's configuration processing capacity.

### 1,000× traffic

- Use regional ingress and independent proxy fleets; keep data-plane request handling local to reduce cross-region dependencies.
- Define global routing and regional failover, including DNS/anycast convergence and capacity reservation.
- Partition route/configuration distribution and observability ingestion to avoid global control-plane bottlenecks.
- Use hierarchical caching or a CDN only when cache semantics and invalidation requirements permit it.
- Test N+1 at regional scale and reserve capacity for failover rather than scaling every region to average demand.
- Establish operational limits for tenant routes, certificates, connections, request sizes, and logging volume.

### Scaling decision checklist

- Is the bottleneck TLS CPU, proxy CPU, memory, file descriptors, network, origin, cache, or entry point?
- Does measured capacity represent the real protocol, payload, and connection-duration mix?
- Can remaining instances serve peak demand during a failure?
- Will scaling proxy instances fragment cache or local rate-limit state?
- Is request coalescing or caching safe for this route?
- Does the data path depend synchronously on the control plane or telemetry system?
- What is the cost and operational complexity of the proposed scale strategy?

---

## 11 — Trade-offs

### Origin selection algorithms

The proxy selects among eligible origins only; unhealthy or administratively drained origins must not receive new requests. A given product may select per connection, request, or stream, so confirm exact semantics for the protocol and version in use.

| Algorithm | Selection behavior | Strength | Limitation |
|---|---|---|---|
| Round Robin | Cycles through eligible origins in sequence | Predictable distribution for similar targets and request cost | Ignores current load, active connections, and request cost |
| Weighted Round Robin | Assigns traffic in proportion to configured weights; weights 1:2:1 approach 25%, 50%, and 25% over a sufficiently large, similar workload | Represents origins with different configured capacity | Static shares can overload a target when its available capacity changes |
| Least Connections | Chooses an eligible origin with the fewest active connections | Can help when connection lifetimes differ | Connection count may not represent request work; long-lived streams can influence selection |
| IP Hash | Hashes a client address to select an origin and provide affinity | May keep a client's requests on the same origin without a separate affinity store | NAT, mobile IP changes, skew, and pool changes can create imbalance or remapping |

These policies are not interchangeable. Sticky routing is not a substitute for shared application session state, and a load distribution algorithm does not make an origin pool highly available without health detection and spare capacity.

| Decision | Option A | Option B | Considerations |
|---|---|---|---|
| Proxy deployment | One proxy | Redundant proxy fleet | Simplicity vs. availability, capacity, and configuration consistency |
| TLS | Terminate at proxy | Pass through TLS | Central policy and HTTP visibility vs. end-to-end TLS and reduced L7 visibility |
| Origin encryption | HTTP in trusted network | HTTPS / mTLS to origin | Operational simplicity vs. protection against internal network threats |
| Routing | Round Robin | Least Connections | Simple distribution vs. connection-state awareness and workload mismatch |
| Routing affinity | IP Hash | Stateless routing / shared session store | Affinity convenience vs. client-IP skew and remapping |
| Capacity weighting | Equal targets | Weighted Round Robin | Simpler pool vs. explicit unequal target capacity |
| Cache | Disabled | Shared or local response cache | Freshness/privacy simplicity vs. lower origin load and added consistency concerns |
| Rate limit | Per-instance state | Shared/distributed state | Fast local decision vs. global accuracy and coordination cost |
| Health checks | Passive failure observation | Active readiness probes | No probe traffic vs. faster detection and additional configuration/traffic |
| Retries | Disabled | Bounded retries for safe cases | No duplicate work vs. selected transient-failure recovery |
| Configuration | Static files | Control-plane distribution | Simple deployment vs. dynamic fleet management and rollout risks |
| Entry point | Direct proxy exposure | Separate L4 load balancer | Fewer hops vs. proxy-tier HA, address stability, and health-based instance selection |

### Required decision record format

For every major decision, interns should document:
- Context and problem.
- Options considered.
- Chosen option.
- Why it was selected.
- What is gained.
- What is sacrificed.
- Conditions that would justify revisiting it.

---

## 12 — Cost

No cloud provider, region, instance family, traffic price, or current price list is specified. Therefore, this section provides a cost model rather than a fabricated currency estimate.

### Cost categories

| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Entry load balancer / public IP | Hours, capacity units, active connections | Connections, bytes, health checks |
| Proxy compute | Instance-hours, CPU, memory | TLS rate, RPS mix, connection count, headroom |
| Proxy network | Ingress/egress and cross-zone transfer | Request and response bytes, zone placement |
| Origin compute | Service capacity and redundancy | RPS, CPU, memory, N+1 capacity |
| Cache | Memory/disk and node-hours | Working set, hit ratio, TTL, replication |
| Certificates / key management | Issuance, storage, HSM or KMS operations | Certificate count, rotation, handshake needs |
| Control plane | Compute, storage, configuration distribution | Fleet size, route count, update frequency |
| Observability | Log/metric/trace ingestion and retention | Events/sec, sampling, retention, cardinality |
| Security services | WAF, DDoS, bot control, scanning | Requests, inspected bytes, policy scope |

### Estimation worksheet

Populate using a selected provider and region:

- Monthly entry cost = entry/load-balancer quote for capacity and hours.
- Monthly proxy compute = proxy instance count × hourly price × hours.
- Network cost = measured ingress/egress and cross-zone transfer × provider rates.
- Origin cost = required normal and N+1 compute capacity × hourly price.
- Cache cost = cache nodes and storage × hourly price.
- Control-plane and key-management cost = service usage and operations quote.
- Observability cost = ingestion + retention + query charges.
- Security-service cost = inspected requests/bytes and selected protection tier.

Document date, region, currency, traffic assumptions, N+1 capacity, pricing URL, and whether discounts or reserved capacity are included. Include peak headroom and failure capacity rather than pricing only average load.

---

## 13 — Security

### Threats and controls

| Threat | Mitigation |
|---|---|
| Open-proxy or host-header abuse | Allowlist virtual hosts and routes; reject unknown hosts; never derive arbitrary upstream destinations from client input |
| Spoofed client identity | Strip or overwrite forwarding headers at the trusted edge; trust incoming identity headers only from explicitly trusted upstreams |
| Origin bypass | Restrict origin network access to proxy identities/subnets; authenticate proxy-to-origin where appropriate |
| Client-to-proxy interception | Enforce secure TLS, certificate lifecycle, and current protocol policy |
| Proxy-to-origin interception | Use TLS/mTLS when threat model requires it; verify origin CA and hostname |
| Request smuggling / parser differential | Reject ambiguous framing and malformed requests; align parser behavior; patch proxy and origin |
| Denial of service | Bound connections, request sizes, header/body buffers, timeouts, and concurrency; apply rate controls |
| Cache data disclosure | Exclude personalized/private responses; use safe cache keys; honor authorization, cookies, and cache-control policy |
| Rate-limit evasion | Base identity on trusted metadata; document local versus distributed counters |
| Malicious configuration | Authenticate and authorize operators; validate, audit, canary, and roll back changes |
| Certificate/private-key theft | Restrict access, use managed secret storage, rotate, and audit key access |
| Sensitive telemetry leakage | Redact tokens and sensitive query data; restrict log access and retention |
| Backend SSRF through dynamic routing | Do not accept client-controlled upstream URLs; validate configured origin addresses and network ranges |

### Validation rules

- Accept only explicitly configured hostnames and route patterns; use a rejecting default virtual host.
- Parse and normalize HTTP authority, path, and forwarding metadata consistently.
- Remove untrusted `Forwarded`, `X-Forwarded-*`, and identity headers before setting trusted values.
- Set request-header, request-body, response-buffer, connection, and timeout limits per route.
- Validate origin address, port, TLS identity, and allowed network before accepting configuration.
- Apply cache policy only after considering method, authorization, cookies, `Vary`, and origin cache directives.
- Do not retry non-idempotent operations unless application-level idempotency is designed and enforced.
- Keep proxy management and control-plane listeners private and authenticated.

---

## 14 — Observability

### Metrics

| Metric | Type | Why it matters |
|---|---|---|
| `proxy_requests_total` | Counter | Request volume by bounded route and outcome |
| `proxy_request_duration_ms` | Histogram | Client-facing proxy processing and tail latency |
| `proxy_upstream_duration_ms` | Histogram | Origin response time and dependency health |
| `proxy_upstream_connect_errors_total` | Counter | Origin reachability and connection failure |
| `proxy_active_connections` | Gauge | Connection pressure and capacity |
| `proxy_healthy_origins` | Gauge | Remaining origin capacity |
| `proxy_tls_handshake_errors_total` | Counter | Certificate, protocol, and client compatibility issues |
| `proxy_cache_requests_total` | Counter | Cache hits, misses, bypasses, and revalidations |
| `proxy_rate_limit_decisions_total` | Counter | Enforcement behavior and rejected traffic |
| `proxy_config_version` | Gauge / info | Fleet rollout consistency and stale instances |
| `proxy_config_reload_errors_total` | Counter | Invalid or failed configuration distribution |
| `proxy_resource_utilization` | Gauge | CPU, memory, file descriptors, and network saturation |

### Logs and traces

- Record request ID, bounded route ID, method, status, selected origin, upstream status, and timing breakdown.
- Record configuration version and health-state transitions.
- Avoid logging full bodies, credentials, cookies, raw authorization headers, or sensitive query values.
- Correlate entry point, proxy, and origin spans while preserving the original trace context only according to trust policy.
- Use bounded labels; do not use raw URL paths, client IDs, or arbitrary headers as metric dimensions.
- Make access logs non-blocking or buffered with bounded queues; expose dropped-log counters.

### Example alerts

- Client-facing 5xx rate or p95/p99 proxy latency exceeds its objective for a sustained interval.
- Healthy proxy or origin capacity falls below N+1 minimum.
- TLS handshake failure rate rises or a certificate approaches expiry.
- Upstream connect errors, timeouts, or 502/503/504 rates rise.
- Proxy CPU, network, file descriptors, active connections, or queueing approach tested limits.
- Cache hit ratio or origin RPS changes unexpectedly for a cache-enabled route.
- Configuration versions diverge across the fleet or rollout health regresses.
- Control-plane age, health-check failures, or telemetry drops exceed policy.

---

## 15 — Simulation

### Learning objective

Let learners observe how request routing, TLS overhead, cache eligibility, rate limiting, origin health, proxy capacity, and failures affect reverse-proxy throughput, availability, and estimated latency.

### Initial simulation controls

| Control | Default | Suggested range |
|---|---:|---:|
| Incoming request RPS | 1,000 | 10–100,000 |
| Proxy instances | 4 | 1–20 |
| Per-proxy tested capacity | 5,000 RPS | 100–20,000 |
| Origin instances | 4 | 1–50 |
| Per-origin capacity | 4,000 RPS | 100–20,000 |
| Routing algorithm | Round Robin | Round Robin / Weighted / Least Connections / IP Hash |
| Backend weight | 1 | 1–10 |
| Failed proxy instances | 0 | 0–N |
| Failed origin instances | 0 | 0–N |
| Average origin service time | 20 ms | 1–1,000 ms |
| TLS termination | Enabled | On / Off |
| TLS proxy overhead | 1 ms | 0–20 ms |
| Cache enabled | Off | On / Off |
| Eligible cache hit ratio | 0% | 0–100% |
| Per-client rate limit | 100 RPS | 1–100,000 |
| Burst allowance | 200 requests | 0–10,000 |
| Health detection delay | 5 sec | 0–60 sec |
| Upstream timeout | 1,000 ms | 10–60,000 ms |

These are simulator defaults, not production benchmarks. The simulator must label all latency and capacity outcomes as estimates.

### Expected outputs

- Incoming, forwarded, cache-served, rate-limited, rejected, and timed-out requests.
- Per-proxy and per-origin request distribution and utilization.
- Active connection estimate and queue depth.
- Origin health state and health-detection delay.
- Cache hits, misses, bypasses, and resulting origin request rate.
- Estimated average and p95 latency with proxy, cache, and origin components.
- Proxy-tier and origin-tier failure capacity.
- Error rate and availability indicator.

### Experiments

1. Baseline with four proxies and four equal origins.
2. Fail one proxy; verify remaining capacity and entry-point redistribution.
3. Fail one origin; compare Round Robin with health-aware removal.
4. Set origin weights to 1:2:1 and compare request shares.
5. Compare Least Connections with unequal service times.
6. Enable IP Hash with both uniform and skewed client populations.
7. Enable caching for eligible traffic; verify origin load decreases without caching private requests.
8. Apply per-client rate and burst limits; observe rejection or delay behavior.
9. Increase traffic beyond proxy capacity, then beyond origin capacity.
10. Increase health-detection delay and observe the window of failed requests.
11. Add TLS overhead and compare request throughput under the same proxy capacity budget.
12. Simulate a bad configuration rollout and roll back to the last known-good version.

### Simulation model warning

The simulator must state its queueing, cache, rate-limit, health-detection, and latency equations. A constant per-server RPS is a teaching abstraction; real capacity depends on protocol, payload size, TLS, connection lifetime, route complexity, hardware, and implementation. Do not present the model as a benchmark of NGINX, HAProxy, Envoy, or a cloud provider.

---

## 16 — Design Challenge

### Challenge: Build a highly available reverse-proxy service

Design a service that accepts HTTPS traffic for multiple application hostnames, applies explicit routing and edge policy, and forwards requests to healthy private origins.

**Requirements**
- Route requests by hostname and path.
- Terminate client TLS and define origin TLS verification.
- Support Round Robin, Weighted Round Robin, Least Connections, and IP Hash as selectable design options.
- Health-check origins and fail over without sending traffic to known-unhealthy targets.
- Apply request limits, timeouts, and safe retry rules.
- Cache only responses that are explicitly safe for shared reuse.
- Keep origins private and handle forwarding headers securely.
- Scale proxy instances independently from origin services.
- Roll out route/certificate changes safely and observe failures.

**Constraints for the exercise**
- Start with one proxy and one origin.
- Evolve to four proxy instances and four origins using the scale assumptions in Section 03.
- Maintain peak service after losing one proxy or origin.
- Add TLS termination, health checks, rate limits, and optional cache.
- Inject a failed origin, a proxy host failure, a TLS certificate problem, and a bad configuration.

**Learner deliverables**
1. Functional and non-functional requirements with workload assumptions.
2. Naive and evolved architecture diagrams.
3. Request, response, health-check, and configuration-rollout flows.
4. Routing algorithm selection and trade-off record.
5. TLS trust boundaries and header-handling policy.
6. Cache eligibility, rate-limit, timeout, and retry rules.
7. Capacity and N+1 estimates for proxy and origin tiers.
8. Failure analysis and recovery procedures.
9. Cost model, security plan, and observability plan.
10. One product-specific configuration example validated against its official versioned documentation.

**Evaluation prompts**
- What is the difference between a reverse proxy and a load balancer?
- Where does TLS terminate, and how is the proxy-to-origin connection protected?
- What happens if every origin is unhealthy or the control plane is unavailable?
- Why might Least Connections outperform Round Robin, and when might it not?
- How do you prevent spoofed forwarding headers and cache leakage?
- Which requests can be retried safely?
- How do you preserve N+1 capacity during a zone failure?
- How would you roll back a route or certificate change?

---

## 17 — AI Interview

### Interviewer question bank

| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What does a reverse proxy do, and how is it different from a forward proxy? | Server-side intermediary, client perspective, trust boundary |
| Q02 | Beginner | Walk through a client request and origin response. | DNS/entry, listener, route, origin selection, response relay |
| Q03 | Beginner | What is the difference between proxying and load balancing? | Intermediary role vs. distribution policy |
| Q04 | Intermediate | Compare Round Robin, Weighted Round Robin, Least Connections, and IP Hash. | Assumptions, workload, affinity, skew, product semantics |
| Q05 | Intermediate | What changes when TLS terminates at the proxy? | Separate connections, certificate handling, origin encryption |
| Q06 | Intermediate | How do active and passive health checks differ? | Detection delay, readiness, thresholds, probe limitations |
| Q07 | Intermediate | How do you safely cache a response? | Method, authorization, cookies, cache directives, key, invalidation |
| Q08 | Intermediate | How do rate limits behave across multiple proxy instances? | Per-instance state vs. coordinated state, identity trust |
| Q09 | Advanced | When is retrying an upstream request unsafe? | Non-idempotent operations, ambiguous completion, idempotency keys |
| Q10 | Advanced | How would you avoid a proxy or zone failure taking down the service? | Redundancy, entry routing, N+1 capacity, health, failure-domain design |
| Q11 | Advanced | How would you prevent request smuggling and spoofed client identity? | Parser alignment, framing validation, header sanitization, trusted hops |
| Q12 | Advanced | How do you distribute a route and certificate change safely? | Validation, canary, staged rollout, health observation, rollback |
| Q13 | Advanced | How would you design multi-region proxy and cache behavior? | Regional isolation, routing convergence, configuration replication, consistency |
| Q14 | Advanced | What would you measure to distinguish proxy saturation from a slow origin? | Timing breakdown, CPU/network/connections, upstream latency, errors |

### Follow-up evaluation dimensions

- Clarifies protocol, workload, and trust assumptions before choosing features.
- Explains request/response flow and distinguishes proxy from load balancer.
- Understands routing and health-check limitations.
- Separates client TLS from origin TLS.
- Identifies cache privacy, retry, and rate-limit consistency risks.
- Provides failure-capacity calculations and operational recovery.
- Supports product-specific claims with current official documentation.

---

## 18 — Architecture Score

### Proposed evaluation rubric

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements and workload assumptions | 10 | Clear routes, protocols, traffic and SLO assumptions |
| Architecture and request flow | 15 | Correct proxy boundaries, origin path, and configuration path |
| Scalability and capacity | 15 | Per-instance capacity, bottleneck analysis, N+1 plan |
| Reliability and failure handling | 15 | Proxy/origin health, failover, timeout, rollback |
| Routing and performance | 10 | Algorithm choice, connection reuse, buffering and latency |
| Security and privacy | 15 | TLS, trust headers, origin isolation, cache and parser safety |
| Cost and trade-off justification | 5 | Transparent cost drivers and alternatives |
| Observability and operations | 10 | Metrics, logs, traces, alerts, rollout evidence |
| **Total** | **100** | |

### Scoring guidance

- 0: Missing or technically incorrect.
- 1: Mentioned without explanation.
- 2: Partially explained.
- 3: Correct and reasonably justified.
- 4: Detailed, measurable, and handles edge cases.

Weighted score = sum of (dimension score / 4 × dimension weight).

The score should be accompanied by evidence and reviewer comments; it should not be treated as an objective measure of architecture quality without human review.
