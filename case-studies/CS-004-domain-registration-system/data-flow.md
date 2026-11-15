# CS-DomainRegistration — Data Flow Specification

## Flow 1: Search domain
```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant API as Registration API
    participant Cache
    participant R as Registry Adapter
    C->>LB: GET /api/v1/domains/search
    LB->>API: Forward request
    API->>Cache: Lookup normalized name
    alt Cache hit
        Cache-->>API: Advisory availability
    else Cache miss
        API->>R: Live availability check
        R-->>API: Registry result
        API->>Cache: Store short TTL result
    end
    API-->>C: Availability and expiring quote
```

## Flow 2: Register domain
```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant API as Registration API
    participant DB as Order Database
    participant Q as Durable Broker
    participant W as Registration Worker
    participant P as Payment Provider
    participant R as Registry
    C->>LB: POST /api/v1/orders + idempotency key
    LB->>API: Forward request
    API->>DB: Commit pending order and outbox
    API-->>C: 202 Accepted + order ID
    DB-->>Q: Outbox relay publishes job
    Q->>W: Deliver job
    W->>P: Authorize payment
    W->>R: Check and create domain
    alt Confirmed creation
        R-->>W: Registry confirmation
        W->>DB: Persist domain and confirmation
        W->>P: Capture payment
        W->>DB: Record payment outcome
    else Definitive failure
        W->>P: Void authorization
        W->>DB: Record failure
    else Ambiguous outcome
        W->>DB: Mark pending reconciliation
    end
```

## Flow 3: Renewal and lifecycle
```mermaid
flowchart LR
    A[Renewal Scheduler] --> B[Durable Broker]
    B --> C[Lifecycle Worker]
    C --> D[Payment Authorization]
    D --> E[Registry Renewal]
    E --> F[Persist Confirmed Expiry]
    F --> G[Capture Payment]
    G --> H[Notify Registrant]
    E -->|Uncertain outcome| I[Reconciliation]
```

## Error and edge-case paths
- Invalid name: 400; unsupported TLD: defined validation error.
- Unavailable name: reject order; cached search is advisory.
- Duplicate idempotency key: return existing order; conflicting payload: 409.
- Registry timeout: reconcile before retrying create or renewal.
- Payment capture failure after creation: retain confirmed domain and escalate billing.
- Broker unavailable: retain durable outbox and replay; never report unconfirmed success.
