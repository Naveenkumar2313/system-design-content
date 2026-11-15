# CS-WebHosting — Data Flow Specification

## Flow 1: Create and deploy a site

```mermaid
sequenceDiagram
    participant C as Owner
    participant API as Management API
    participant DB as Metadata Database
    participant Q as Event Broker
    participant W as Deployment Worker
    participant S as Object Storage
    C->>API: POST /api/v1/sites/{id}/deployments
    API->>API: Authorize and validate upload
    API->>S: Store staged bundle
    API->>DB: Persist pending job and outbox
    API-->>C: 202 Accepted + deployment ID
    DB-->>Q: Publish job via outbox relay
    Q->>W: Deliver job
    W->>S: Validate and publish immutable files
    W->>DB: Atomically activate ready version
    C->>API: GET deployment status
    API-->>C: Ready or failed
```

## Flow 2: Serve a website

```mermaid
sequenceDiagram
    participant B as Browser
    participant D as DNS
    participant E as CDN
    participant LB as Load Balancer
    participant O as Origin
    participant DB as Metadata Database
    participant S as Object Storage
    B->>D: Resolve hostname
    D-->>B: Edge address
    B->>E: HTTPS GET /path
    alt Eligible cache hit
        E-->>B: Cached response
    else Cache miss
        E->>LB: Forward request
        LB->>O: Route to healthy origin
        O->>DB: Resolve hostname and active version
        O->>S: Fetch versioned file
        S-->>O: File or not found
        O-->>E: Response and cache policy
        E-->>B: HTTP response
    end
```

## Flow 3: Domain management and rollback

```mermaid
flowchart LR
    A[Owner] --> B[Management API]
    B --> C[Domain ownership verification]
    C --> D[TLS manager]
    D --> E[Enable domain routing]
    B --> F[Select ready version]
    F --> G[Atomic active pointer update]
    G --> H[CDN purge or bounded TTL]
```

## Error and edge-case paths

- Unknown hostname or missing file: return 404.
- Unauthorized management request: return 403.
- Failed deployment: retain previous active version.
- Storage timeout: bounded retry or controlled 503.
- Broker unavailable: retain pending job in durable outbox; retry publication.
- Duplicate job: use deployment ID and conditional activation to prevent stale retries from replacing newer versions.
