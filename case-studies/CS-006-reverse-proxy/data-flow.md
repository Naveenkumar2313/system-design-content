# CS-ReverseProxy — Data Flow Specification

## Flow 1: Forward request
```mermaid
sequenceDiagram
 participant C as Client
 participant LB as Load Balancer
 participant P as Reverse Proxy
 participant O as Origin
 C->>LB: HTTPS request
 LB->>P: Forward connection
 P->>P: TLS and route validation
 P->>O: Forward HTTP request
 O-->>P: HTTP response
 P-->>C: Relay response via LB
```
## Flow 2: Cache lookup
```mermaid
sequenceDiagram
 participant C as Client
 participant P as Reverse Proxy
 participant K as HTTP Cache
 participant O as Origin
 C->>P: Eligible GET request
 P->>K: Lookup cache key
 alt Fresh hit
 K-->>P: Cached response
 else Miss or stale
 P->>O: Fetch or revalidate
 O-->>P: Response
 P->>K: Store only if permitted
 end
 P-->>C: HTTP response
```
## Flow 3: Configuration update
```mermaid
flowchart LR
 A[Operator] --> B[Configuration Service]
 B --> C[Validation and Version Store]
 C --> D[Proxy Fleet]
 D --> E[Rollout Status and Telemetry]
```
## Error and edge-case paths
- Unknown route: 404; malformed request: 400.
- No healthy origin: 503; origin timeout: 504 if headers not sent.
- Cache failure: bypass with origin load protection.
- Invalid configuration: reject; retain last known good version.
- Do not retry non-idempotent requests automatically without an explicit safe policy.
