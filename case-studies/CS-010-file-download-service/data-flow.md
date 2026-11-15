# CS-010 — File Download Service Data Flows

These flows describe a proposed design. Provider-specific URL, range, cache, and revocation semantics must be confirmed against the selected service.

## Authorized download and URL issuance

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant G as API Gateway / Load Balancer
    participant A as Auth + Access Control
    participant D as Download Service
    participant M as Metadata DB
    participant S as URL Signer
    participant L as Audit Pipeline

    C->>G: GET /v1/files/{file_id}/download
    G->>A: Validate identity and policy
    A-->>G: Principal + decision
    G->>D: Authorized request + request ID
    D->>M: Resolve file/version/status/object metadata
    M-->>D: Immutable version, size, ETag, policy class
    D->>A: Confirm access to requested version
    A-->>D: Allow / deny
    alt Denied, deleted, or unavailable
        D-->>C: 403 / 404 / 409
    else Allowed
        D->>S: Sign GET for one object version with bounded expiry
        S-->>D: Short-lived URL or cookie
        D->>L: Record decision and grant issuance (redacted)
        D-->>C: 302 Location or 200 JSON delivery descriptor
    end
```

## CDN cache HIT

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant E as CDN Edge
    participant P as Access Policy / Signature Validator
    participant L as CDN Logs

    C->>E: GET /objects/{version} (+ optional Range)
    E->>P: Validate signed request/cookie and cache policy
    P-->>E: Allow
    E->>E: Lookup canonical key for immutable version
    Note over E: Valid cached representation found
    alt No Range / full representation
        E-->>C: 200 OK + validators + file bytes
    else Satisfiable Range
        E-->>C: 206 + Content-Range + requested bytes
    else Unsatisfiable Range
        E-->>C: 416 + Content-Range: bytes */length
    end
    E->>L: Cache HIT, status, bytes, latency (no secret URL)
```

## CDN cache MISS and origin shield

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant E as CDN Edge
    participant P as Access Policy / Signature Validator
    participant S as Origin Shield
    participant O as Private Object Storage

    C->>E: GET /objects/{version} (+ optional Range)
    E->>P: Validate grant / policy
    P-->>E: Allow
    E->>E: Cache MISS or revalidation required
    E->>S: Fetch exact immutable object/version
    S->>S: Collapse concurrent misses for same cache key
    S->>O: Authenticated GET / Range GET
    O-->>S: 200 / 206 + validators + bytes
    S-->>E: Origin response
    E->>E: Cache only if class and response policy allow
    E-->>C: Stream response with correct HTTP status and headers
```

## Safe range resume after interruption

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant E as CDN / Delivery Endpoint
    participant O as Origin

    C->>E: Initial GET
    E->>O: GET immutable object
    O-->>E: 200 + ETag "v17" + Content-Length
    E-->>C: 200 + ETag "v17" + bytes
    Note over C: Connection interrupted; persist local length N and ETag
    C->>E: GET + Range: bytes=N- + If-Range: "v17"
    E->>O: Range GET with validator (cache miss)
    alt Validator still matches
        O-->>E: 206 + Content-Range: bytes N-(L-1)/L + ETag "v17"
        E-->>C: 206 + matching Content-Range and remaining bytes
        Note over C: Append only after validating offset, total length, and ETag
    else Object version changed
        O-->>E: 200 full current representation
        E-->>C: 200 full body; client replaces/restarts partial file
    else Range cannot be satisfied
        O-->>E: 416 + Content-Range: bytes */L
        E-->>C: 416; client reconciles local offset or restarts
    end
```

## Revocation and cache purge

```mermaid
flowchart TD
    A[Owner revokes access or deletes version] --> B[Metadata state becomes unavailable]
    B --> C[Stop issuing new download grants]
    B --> D[Restrict origin read / revoke capability where supported]
    B --> E[Request CDN purge or rely on bounded immutable URL expiry]
    E --> F{Purge confirmed?}
    F -- Yes --> G[Record completion and retain audit]
    F -- No --> H[Keep authorization gate and origin denial active]
    H --> I[Retry purge and alert on revocation SLO]
    C --> J[Existing in-flight transfer may continue until provider/request boundary]
```

## Notes

- The client appends a partial response only if the requested start offset, `Content-Range`, total length, and validator agree with its saved state.
- A CDN cache hit must still validate a private-content grant or use a correctly scoped signed capability.
- CDN purge completion and URL expiry are not universal instantaneous revocation mechanisms.
- Log object/file/version IDs and decisions, but redact signed URL query strings, cookies, credentials, and authorization headers.
