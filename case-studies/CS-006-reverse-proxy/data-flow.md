# CS-006 — Data Flow Specification

## Flow 1: Client request and origin response

```mermaid
sequenceDiagram
    participant C as Client
    participant DNS as DNS
    participant LB as CMP-001 Load Balancer
    participant RP as Reverse Proxy
    participant Cache as Optional Cache
    participant HC as Health State
    participant O as Origin Service

    C->>DNS: Resolve api.example.com
    DNS-->>C: Public entry address
    C->>LB: HTTPS request
    LB->>RP: Select healthy proxy instance
    RP->>RP: TLS handshake, validate host and route
    RP->>RP: Apply request limits and policy
    opt Route allows shared caching
        RP->>Cache: Lookup normalized cache key
        alt Fresh eligible cache hit
            Cache-->>RP: Cached response
            RP-->>C: Relay cached response
        else Miss or revalidation required
            RP->>HC: Read eligible origin state
            HC-->>RP: Healthy origin candidates
            RP->>O: Forward request with normalized headers
            O-->>RP: Origin response
            opt Response is cacheable
                RP->>Cache: Store according to cache policy
            end
            RP-->>C: Relay origin response
        end
    end
```

## Flow 2: Origin health and failover

```mermaid
sequenceDiagram
    participant H as Health Checker
    participant O1 as Origin 1
    participant O2 as Origin 2
    participant S as Proxy Pool State
    participant RP as Reverse Proxy

    H->>O1: Readiness probe
    O1-->>H: Probe result
    H->>S: Update health after configured thresholds
    RP->>S: Get eligible origins
    S-->>RP: Healthy origins only
    RP->>O2: Route new request
    Note over RP,O1: In-flight requests follow timeout/drain policy; they are not automatically safe to replay
```

## Flow 3: Configuration rollout

```mermaid
flowchart LR
    A[Operator submits versioned route] --> B[Validate host, route, pool, TLS and policy]
    B -->|Invalid| X[Reject and report validation errors]
    B -->|Valid| C[Apply to canary proxy]
    C --> D{Health and request outcomes acceptable?}
    D -->|No| R[Rollback to last known-good version]
    D -->|Yes| E[Gradually distribute to proxy fleet]
    E --> F[Observe route errors and fleet version]
    F --> G{Rollout healthy?}
    G -->|No| R
    G -->|Yes| H[Mark version active]
```

## Error and edge-case paths

- Unknown hostname or unconfigured route: reject without forwarding to a default arbitrary origin.
- No healthy origin: return a controlled 503 and emit an alert.
- Upstream connection failure: return 502 or bounded retry only when the request is safe to retry.
- Upstream timeout: return 504; enforce a bounded deadline.
- Invalid upstream response: return 502 and record the origin and protocol failure.
- Rate limit exceeded: delay within configured burst or return 429.
- TLS validation failure to origin: fail closed; do not disable certificate verification.
- Private or authenticated response: bypass shared cache unless an explicit safe policy exists.
- Control plane unavailable: continue with last known-good configuration and expose its version/age.
- Client disconnects during streaming: cancel upstream work when safe and release resources.
