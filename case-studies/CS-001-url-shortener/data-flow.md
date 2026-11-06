# CS-001 — Data Flow Specification

## Flow 1: Create link

```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant API as Application Service
    participant ID as Short-Code Generator
    participant DB as Mapping Database

    C->>LB: POST /api/v1/links
    LB->>API: Forward request
    API->>API: Validate URL and policy
    API->>ID: Generate candidate code
    ID-->>API: Candidate code
    API->>DB: Insert mapping (unique constraint)
    DB-->>API: Commit success
    API-->>C: 201 Created + short URL
```

## Flow 2: Redirect

```mermaid
sequenceDiagram
    participant B as Browser
    participant LB as Load Balancer
    participant R as Redirect Service
    participant Cache
    participant DB as Mapping Database
    participant Q as Event Broker

    B->>LB: GET /{code}
    LB->>R: Forward request
    R->>Cache: GET code
    alt Cache hit
        Cache-->>R: Mapping
    else Cache miss
        R->>DB: Lookup code
        DB-->>R: Mapping or not found
        R->>Cache: Populate cache
    end
    R-->>Q: Publish click event (async)
    R-->>B: 302 Location: destination
```

## Flow 3: Analytics

```mermaid
flowchart LR
    A[Redirect Service] -->|Click event| B[Event Broker]
    B --> C[Analytics Worker]
    C --> D[Deduplication]
    D --> E[Analytics Store]
    E --> F[Reporting API]
```

## Error and edge-case paths

- Unknown code: return 404.
- Expired code: return a defined expired-link response.
- Deactivated code: do not redirect.
- Database timeout on cache miss: controlled 5xx or service-specific fallback.
- Event broker unavailable: apply the documented event durability policy; do not block redirect indefinitely.
- Duplicate event: deduplicate by event ID if exact-once aggregation is required at the application layer.
