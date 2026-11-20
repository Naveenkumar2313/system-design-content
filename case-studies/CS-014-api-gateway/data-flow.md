# CS-014 — Data Flow Specification

## Flow 1: Route API request
```mermaid
sequenceDiagram
 participant C as Client
 participant LB as Load Balancer
 participant G as API Gateway
 participant Q as Quota Store
 participant K as Response Cache
 participant U as Upstream Service
 C->>LB: HTTPS API request
 LB->>G: Forward request
 G->>G: Match route, authenticate and authorize
 G->>Q: Atomic quota decision
 alt Quota denied
 Q-->>G: Denied
 G-->>C: 429 via LB
 else Allowed
 Q-->>G: Allowed
 opt Eligible public GET
 G->>K: Lookup response
 K-->>G: Hit or miss
 end
 alt Cache hit
 G-->>C: Cached response via LB
 else Miss or not cacheable
 G->>U: Forward with deadline
 U-->>G: Response
 G-->>C: Response via LB
 end
 end
```

## Flow 2: Publish configuration
```mermaid
sequenceDiagram
 participant O as Operator
 participant P as Control Plane
 participant D as Configuration Store
 participant G as Gateway Replicas
 O->>P: Submit authorized route version
 P->>P: Validate and approve
 P->>D: Commit snapshot
 D-->>P: Durable commit
 P->>G: Publish validated snapshot
 G->>G: Atomically activate version
 G-->>P: Acknowledge version
```

## Flow 3: Observability
```mermaid
flowchart LR
 A[Gateway] --> B[Telemetry Collector]
 C[Upstream Services] --> B
 D[Control Plane] --> B
 B --> E[Metrics and Logs]
 E --> F[Dashboard and Alerts]
```

## Error and edge-case paths
- Unknown route: 404; invalid credentials: 401; insufficient permission: 403.
- Exhausted quota: 429; quota dependency unavailable: 503, fail closed.
- Upstream connection failure: 502 or 503 according to policy; deadline exceeded: 504.
- Cache unavailable: bypass with upstream capacity protection.
- Invalid configuration: reject update and retain last valid version.
- Never automatically retry non-idempotent operations without an explicit safe contract.
