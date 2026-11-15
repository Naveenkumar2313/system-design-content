# CS-Pastebin — Data Flow Specification

## Flow 1: Create paste

```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant API as Application Service
    participant ID as Paste-ID Generator
    participant DB as Paste Database

    C->>LB: POST /api/v1/pastes
    LB->>API: Forward request
    API->>API: Validate text size and policy
    API->>ID: Generate candidate paste ID
    ID-->>API: Candidate paste ID
    API->>DB: Insert paste record (unique constraint)
    DB-->>API: Commit success
    API-->>C: 201 Created + paste URL
```

## Flow 2: Read

```mermaid
sequenceDiagram
    participant B as Browser
    participant LB as Load Balancer
    participant R as Read Service
    participant Cache
    participant DB as Paste Database
    participant Q as Event Broker

    B->>LB: GET /{paste_id}
    LB->>R: Forward request
    R->>Cache: GET paste_id
    alt Cache hit
        Cache-->>R: Paste record
    else Cache miss
        R->>DB: Lookup paste_id
        DB-->>R: Paste record or not found
        R->>Cache: Populate only active public paste
    end
    R->>R: Check visibility, authorization, status and expiry
    alt Accessible active paste
        R-->>Q: Publish view event (async)
        R-->>B: 200 OK + paste content
    else Missing, expired or inaccessible
        R-->>B: 404 or defined expiry response
    end
```

## Flow 3: Analytics

```mermaid
flowchart LR
    A[Read Service] -->|View event| B[Event Broker]
    B --> C[Analytics Worker]
    C --> D[Deduplication]
    D --> E[Analytics Store]
    E --> F[Reporting API]
```

## Error and edge-case paths

- Unknown paste ID: return 404.
- Expired paste ID: return a defined expired-paste response.
- Deactivated paste ID: do not return content.
- Database timeout on cache miss: controlled 5xx or service-specific fallback.
- Event broker unavailable: apply the documented event durability policy; do not block read indefinitely.
- Duplicate event: deduplicate by event ID if exact-once aggregation is required at the application layer.
