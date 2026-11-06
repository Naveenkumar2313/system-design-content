# CMP-001 — Load Balancer

> **Content status:** Generic educational component specification. This is not a specification for a particular vendor or product. Numerical defaults in the simulation are illustrative.
>
> **Source alignment:** Load balancing and traffic management appear in the supplied System Design Knowledge Library as a major domain. The case-study catalogue includes load balancer and cloud load balancer design exercises. The source documents do not prescribe a specific implementation or configuration.

## A. Component identity

| Field | Value |
|---|---|
| Component ID | CMP-001 |
| Component name | Load Balancer |
| Category | Traffic Management / Networking |
| Subcategory | Request Distribution |
| Difficulty | Beginner to Intermediate |
| Component type | Infrastructure / Traffic Management |
| Short description | A component that distributes incoming requests across a set of backend targets according to a routing policy and target-health state. |
| Related concepts | Client-server, HTTP, reverse proxy, health checks, routing, availability, horizontal scaling |
| Related case studies | CS-001 URL Shortener (proposed architecture evolution); other links to be added as case studies are approved |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Status | Draft reference specification |

## B. Core documentation

### 1. Definition

A load balancer distributes incoming network or application requests across multiple backend targets. Depending on its type, it may operate at the transport layer or application layer and may make routing decisions using connection, request, health, or application metadata.

### 2. Purpose

- Distribute load across multiple backend instances.
- Avoid sending new traffic to targets marked unhealthy.
- Improve capacity by allowing multiple service instances to operate together.
- Provide a stable entry point for clients.
- Support controlled routing policies and, in some designs, TLS termination.

A load balancer does not automatically make an application highly available. Backend redundancy, health checks, data-layer availability, and correct failure handling are also required.

### 3. Internal working

1. Client sends a request to the load balancer's frontend address.
2. Load balancer accepts the connection or request.
3. It identifies the applicable listener, protocol, and routing rule.
4. It selects an eligible backend target according to the configured algorithm and health state.
5. It forwards the request or connection.
6. It relays the response where operating as a proxy.
7. It records metrics and logs according to configuration.

Exact behaviour varies by layer, product, protocol, and deployment mode.

### 4. Architecture

**Logical elements**
- Frontend listener / virtual address.
- Routing policy.
- Backend target group or pool.
- Health-check subsystem.
- Connection and timeout management.
- TLS handling where supported.
- Metrics and logging.

See `architecture.png` for the logical diagram.

### 5. Types

| Type | Description | Typical use |
|---|---|---|
| Layer 4 load balancer | Routes transport connections using network/transport information | TCP/UDP service distribution |
| Layer 7 load balancer | Routes application requests using protocol/application details such as host or path | HTTP/HTTPS routing |
| Hardware load balancer | Dedicated appliance-based implementation | Specialized or high-throughput environments |
| Software load balancer | Software deployed on general-purpose compute | Flexible self-managed infrastructure |
| Managed cloud load balancer | Provider-operated service integrated with cloud networking | Cloud-hosted applications |
| Global load balancer | Directs users across regions or locations | Multi-region applications |

These categories overlap; for example, a managed cloud product may offer both L4 and L7 capabilities.

### 6. Algorithms and routing strategies

| Strategy | How it works | Strength | Limitation |
|---|---|---|---|
| Round Robin | Sends each new request/connection to the next target in sequence | Simple and predictable with similar targets | Does not account for current load |
| Weighted Round Robin | Assigns more traffic to targets with higher configured weights | Supports unequal target capacity | Static weights may not reflect changing load |
| Least Connections | Prefers the target with the fewest active connections | Useful when connection duration varies | Connection count may not equal actual workload |
| Weighted Least Connections | Combines active connections and target weights | Accounts for capacity differences | Requires accurate state and implementation-specific behaviour |
| IP Hash | Uses client address hashing to select a target | Can provide affinity | Skew and changing client IPs can create uneven distribution |
| Least Response Time | Prefers targets based on observed response time and load | Can react to latency differences | Measurement and routing behaviour vary by implementation |
| Random | Selects an eligible target randomly | Simple, low coordination | Short-term distribution can be uneven |

Do not assume every algorithm is supported by every load balancer.

### 7. Configurations

| Configuration | Purpose | Example |
|---|---|---|
| Listener protocol | Protocol accepted by frontend | HTTP, HTTPS, TCP |
| Frontend port | Port exposed to clients | 80 or 443 |
| Backend targets | Registered servers/services | app-1, app-2, app-3 |
| Routing algorithm | Target selection policy | Round Robin |
| Target weights | Relative capacity / share | 1, 2, 1 |
| Health-check path/protocol | How target health is checked | HTTP GET `/health` |
| Health-check interval | Time between checks | 5 seconds (illustrative) |
| Timeout | Connection or request timeout | Product-specific |
| TLS policy | Certificate and protocol configuration | Product-specific |
| Session persistence | Whether a client is directed to a consistent target | Enabled/disabled |
| Connection draining | How existing work is handled when removing a target | Product-specific |

### 8. Inputs

- Client connection or application request.
- Frontend listener and protocol information.
- Backend target list and target health.
- Routing rules and algorithm configuration.
- Optional request metadata, such as host/path/headers for L7 routing.

### 9. Outputs

- Forwarded connection or request to a selected target.
- Backend response returned to client when proxying.
- Routing and health telemetry.
- Error or rejection response when no eligible target or policy prevents forwarding.

### 10. Dependencies

- Network interfaces and routing.
- Backend services and their listening endpoints.
- Health-check endpoints or protocols.
- DNS or another service-discovery mechanism in many deployments.
- Certificates and key management when terminating TLS.
- Monitoring and logging infrastructure.

### 11. Interactions

- Client → Load Balancer: incoming connection/request.
- Load Balancer → Backend: forwarded request/connection.
- Health checker → Backend: periodic probe.
- Service discovery → Load Balancer: target registration/update, where supported.
- Load Balancer → Monitoring: counters, latency, health, and errors.

### 12. Use cases

- Distributing HTTP traffic across stateless web servers.
- Routing API requests to multiple application instances.
- Distributing TCP connections to service endpoints.
- Performing host/path-based routing in L7 systems.
- Routing users to healthy service instances.
- Supporting multi-region traffic management when combined with global routing.

### 13. Advantages

- Better utilization of multiple backend instances.
- Supports horizontal application scaling.
- Can reduce impact of individual backend failure.
- Provides a central point for routing policy.
- Can support TLS termination and application routing, depending on type.

### 14. Limitations

- The load balancer itself can become a bottleneck or failure point if not deployed redundantly.
- It cannot repair application bugs or unavailable dependencies.
- Health checks can be misleading if they test only process liveness rather than service readiness.
- Session affinity can reduce balancing flexibility.
- TLS termination and proxying add processing and operational complexity.
- Uneven request cost can make simple request-count algorithms ineffective.

### 15. Failure modes

| Failure | Effect | Mitigation / design consideration |
|---|---|---|
| One backend fails | Requests to that target fail until detected | Health checks and target removal |
| Health check is too permissive | Broken backend remains eligible | Readiness-aware health endpoint |
| Health check is too aggressive | Healthy target may be removed during transient delay | Tune thresholds and timeouts |
| All backends fail | No healthy target exists | Return controlled error; restore backend capacity |
| Load balancer overload | New requests queue, fail, or time out | Capacity monitoring and redundant/scalable deployment |
| Misconfigured routing rule | Requests reach wrong service or fail | Configuration validation and staged rollout |
| TLS certificate expires | HTTPS connections may fail | Certificate monitoring and renewal |
| Connection draining misconfigured | In-flight work may be interrupted | Set drain behaviour based on request duration |
| Unequal backend capacity | Some targets overload | Weights or suitable adaptive policy |

### 16. Scaling

- Add backend targets and verify that the load balancer can register and health-check them.
- Scale the load balancer tier or use a managed service with documented capacity limits.
- Separate listeners or target groups by workload where appropriate.
- Use connection reuse and keep-alive appropriately.
- Monitor new connections, active connections, requests/sec, bytes/sec, queueing, and target errors.
- For global scale, introduce global routing and regional load balancers; define health, failover, and data consistency behaviour.

### 17. Security

- Restrict management interfaces and APIs.
- Use secure TLS configuration for client-facing HTTPS.
- Protect backend traffic where the threat model requires encryption.
- Apply network access controls and least privilege.
- Validate forwarded headers and do not blindly trust client-supplied identity headers.
- Prevent direct public access to backends when traffic is intended to pass through the load balancer.
- Rate-limit and protect against connection exhaustion where supported.
- Keep software and configuration updated.

### 18. Monitoring

| Metric | Why it matters |
|---|---|
| Incoming requests/connections | Traffic volume |
| Active connections | Connection pressure |
| Requests per backend | Distribution balance |
| Healthy target count | Backend availability |
| Backend response latency | Target performance |
| 4xx / 5xx responses | Client and server error trends |
| Connection errors/timeouts | Network or backend health |
| Bytes in/out | Throughput and capacity |
| TLS handshake failures | Certificate or protocol problems |
| Queue / pending requests | Saturation signal, if exposed |

Metric names and availability depend on implementation.

---

## C. Playground and simulation specification

### Configurable parameters

| Parameter | Default | Minimum | Maximum | Unit |
|---|---:|---:|---:|---|
| Incoming request rate | 1,000 | 10 | 100,000 | requests/sec |
| Backend server count | 3 | 1 | 20 | servers |
| Per-server capacity | 500 | 100 | 5,000 | requests/sec |
| Routing algorithm | Round Robin | — | — | enum |
| Backend weight | 1 | 1 | 10 | relative weight |
| Health-check interval | 5 | 1 | 60 | seconds |
| Failed servers | 0 | 0 | server count | servers |
| Average service time | 20 | 1 | 1,000 | ms |
| Request distribution | Uniform | — | — | uniform / skewed |

All are simulator defaults, not vendor defaults or performance guarantees.

### Simulation inputs

- Offered request rate.
- Number of backend targets.
- Per-target capacity.
- Routing algorithm.
- Per-target weights.
- Failed/unhealthy targets.
- Request duration or service time.
- Optional uneven request-cost distribution.

### Expected outputs

- Requests assigned to each target.
- Successful requests.
- Rejected, queued, or timed-out requests.
- Per-target utilization.
- Estimated average and p95 latency.
- Healthy/unhealthy target state.
- Distribution fairness / imbalance indicator.

### Failure injection

- Mark one backend unavailable.
- Mark all backends unavailable.
- Delay health-check detection.
- Reduce one backend's capacity.
- Increase incoming traffic beyond total capacity.
- Simulate a health-check endpoint failure.

### Visual representation

- One frontend node.
- A target group containing backend nodes.
- Animated request flow.
- Health indicator on each backend.
- Request counter and utilization label per backend.
- Distinct visual states for healthy, overloaded, and unhealthy.

### Interaction rules

- Client can connect to the load balancer.
- Load balancer can connect to one or more backend targets.
- Backend targets expose capacity and health properties.
- Invalid connections should be rejected with an explanatory message.
- Changing the routing algorithm should change request distribution in the next simulation run.

---

## D. Practical examples

### Technology examples to research

These are examples of products/projects in the load-balancing space, not endorsements or claims of identical feature sets:

- HAProxy
- NGINX
- Envoy Proxy
- Apache HTTP Server (proxy/load-balancing modules)
- Cloud-provider application and network load-balancing services

For each example, research:
- Supported protocols and layers.
- Routing algorithms.
- Health-check capabilities.
- TLS handling.
- Configuration model.
- Metrics and observability.
- Known limitations and deployment modes.

### Example configuration (illustrative pseudoconfiguration)

```yaml
listener:
  protocol: HTTP
  port: 80

backend_pool:
  algorithm: round_robin
  targets:
    - name: app-1
      address: 10.0.0.11:8080
      weight: 1
    - name: app-2
      address: 10.0.0.12:8080
      weight: 1
    - name: app-3
      address: 10.0.0.13:8080
      weight: 1

health_check:
  path: /health
  interval_seconds: 5
  timeout_seconds: 2
  unhealthy_threshold: 3
```

This is conceptual YAML for the learning portal, not directly deployable configuration for a specific product.

### Related case study usage

- CS-001 URL Shortener: distribute create and redirect requests across application instances.
- Add more case-study relationships only after the corresponding case study is reviewed.

---

## E. References and validation

### Source documents supplied

1. `System_Design_Knowledge_Library.docx.pdf`
   - Load Balancing & Traffic Management is listed as Level 6.
   - The library includes routing, scaling, and infrastructure topics.
2. `Codorythm_250_Case_Studies.docx.pdf`
   - Includes Design a Load Balancer in the Web & Internet Systems category.
   - Includes Cloud Load Balancer and Global Load Balancer in Cloud & Infrastructure.

### Technical references to consult and verify

- HAProxy documentation: https://www.haproxy.org/
- NGINX documentation: https://nginx.org/en/docs/
- Envoy documentation: https://www.envoyproxy.io/docs
- HTTP Semantics (RFC 9110): https://www.rfc-editor.org/rfc/rfc9110

### Evidence policy

Do not claim that a named product supports a feature until its current official documentation has been checked. Record product version, documentation URL, access date, and reviewer.

### Reviewer checklist

- [ ] Distinguishes L4 and L7 behaviour.
- [ ] Explains routing algorithms accurately.
- [ ] Describes health checks and their limitations.
- [ ] Includes backend failure behaviour.
- [ ] Explains scaling and redundancy.
- [ ] Includes security and monitoring.
- [ ] Labels generic simulation values as illustrative.
- [ ] Product-specific claims have official references.
