# CS-009 — Data Flow Specification

## Flow 1: Initiate and authorize upload

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant A as Auth / Authorization
    participant U as Upload Service
    participant M as Metadata DB
    participant S as Object Storage

    C->>G: POST /v1/uploads + Idempotency-Key
    G->>A: Authenticate and authorize tenant/resource
    A-->>G: Principal and permissions
    G->>U: Validated upload intent
    U->>U: Validate size/type and reserve quota
    U->>M: Create file + upload session (INITIATED)
    U->>S: Initiate multipart/session or prepare single-object grant
    S-->>U: Provider upload ID / session state
    U->>M: Persist provider mapping and UPLOADING state
    U-->>C: File ID, upload ID, part plan, short-lived URL instructions
```

The application returns transfer credentials only after durable metadata and session state exist. Provider upload IDs and signed URLs are secrets and must not be written to ordinary logs.

## Flow 2: Direct single-object upload

```mermaid
sequenceDiagram
    participant C as Client
    participant U as Upload API
    participant O as Private Quarantine Object Storage

    C->>U: Request upload session
    U-->>C: Short-lived URL scoped to one random key and PUT
    C->>O: HTTPS PUT file bytes
    O-->>C: Storage receipt / checksum metadata
    C->>U: POST /uploads/{id}:complete
    U->>O: Verify object key, size, and checksum
    O-->>U: Verified object state
    U-->>C: 202 Accepted (PENDING_SCAN)
```

The file is not downloadable merely because object storage accepted a PUT. The upload API verifies the expected object and moves it to the processing workflow.

## Flow 3: Multipart upload and retry

```mermaid
sequenceDiagram
    participant C as Client
    participant U as Upload API
    participant O as Object Storage
    participant M as Metadata DB

    C->>U: Request signed URLs for parts 1..N
    U->>M: Verify owner/session and part range
    U-->>C: Short-lived URLs for allowed parts
    par Bounded parallel transfer
        C->>O: PUT part 1
        O-->>C: Part ETag / checksum
    and
        C->>O: PUT part 2
        O-->>C: Part ETag / checksum
    end
    Note over C,O: Connection drops while part 3 is incomplete
    C->>U: GET upload progress
    U->>O: Reconcile provider part list / offset
    O-->>U: Parts 1 and 2 committed
    U-->>C: Missing part 3; request a fresh URL
    C->>O: Retry only part 3
    O-->>C: Part 3 receipt
    C->>U: POST complete with part manifest + checksum
    U->>O: Complete multipart upload
    O-->>U: Final object metadata
    U->>M: Verify and transition to PENDING_SCAN
```

The per-part manifest and ETag requirements are storage-provider-specific. A full-object checksum must use a supported, explicitly selected checksum algorithm; multipart ETags are not universally a full-file digest.

## Flow 4: Resume after client restart

```mermaid
flowchart TD
    S[Client restarts or network returns] --> Q[GET upload status]
    Q --> A{Session authorized and active?}
    A -->|No| E[Return expired, aborted, or forbidden state]
    A -->|Yes| R[Reconcile durable app state with provider parts or offset]
    R --> V{Local source file still matches expected identity?}
    V -->|No| X[Require a new upload session]
    V -->|Yes| M[Calculate missing parts or byte range]
    M --> U[Request fresh scoped URLs for missing data]
    U --> P[Transfer missing data with bounded retries]
    P --> C[Complete and verify object]
```

The server is authoritative for committed storage state; client progress indicators alone are not proof that a part was durably stored.

## Flow 5: Outbox, queue, scan, and publish

```mermaid
sequenceDiagram
    participant U as Upload Service
    participant D as Metadata DB + Outbox
    participant P as Outbox Publisher
    participant Q as Durable Queue
    participant W as Sandboxed Worker
    participant O as Object Storage
    participant V as Virus Scanner

    U->>D: Transaction: PENDING_SCAN + outbox event
    D-->>P: Undelivered event
    P->>Q: Publish process-file job
    Q-->>W: Deliver job (at least once)
    W->>O: Read quarantine object
    W->>V: Scan/validate content
    V-->>W: Clean or rejected verdict
    alt Accepted
        W->>O: Write approved original / derivatives
        W->>D: Conditional transition to AVAILABLE
        W->>Q: Acknowledge after durable state
    else Rejected
        W->>D: Conditional transition to REJECTED
        W->>Q: Acknowledge / route review event
    end
```

Outbox publication and worker delivery can repeat. The worker must be idempotent and verify the current file state and object key before committing results.

## Flow 6: Authorized download and deletion

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Service
    participant F as File API
    participant D as Metadata DB
    participant CDN as CDN / Object Delivery
    participant O as Clean Object Storage

    C->>F: POST /v1/files/{id}:download
    F->>A: Authenticate and authorize owner/tenant
    A-->>F: Authorized principal
    F->>D: Read state and access policy
    D-->>F: AVAILABLE + storage key
    F-->>C: Short-lived authorized download URL
    C->>CDN: HTTPS GET with signed access
    CDN->>O: Fetch on eligible cache miss
    O-->>CDN: Approved object
    CDN-->>C: File bytes
    Note over F,D: Delete first revokes new download authorization
```

Deletion must revoke access before asynchronous object and derivative cleanup. Cached copies and replication have provider-specific invalidation/deletion behavior and should be included in the retention policy.

## Flow 7: Failure recovery and cleanup

```mermaid
flowchart TD
    E[Upload error or session inactivity] --> C{Can transfer resume?}
    C -->|Yes| P[Query provider committed parts / offset]
    P --> R[Retry missing parts with bounded backoff]
    R --> V[Verify and complete]
    C -->|No or expired| X[Conditionally mark EXPIRED / ABORTING]
    X --> A[Abort provider multipart session]
    A --> D[Delete orphan parts / quarantine object]
    D --> Q[Reconcile object inventory and release quota]
    V --> T{Checksum and size valid?}
    T -->|Yes| O[Persist PENDING_SCAN + outbox event]
    T -->|No| F[Mark FAILED; quarantine and alert]
```

Cleanup should be idempotent and observable. If storage is unavailable, the session remains in a retryable cleanup state rather than being recorded as successfully deleted.

## Error and edge-case paths

- Initiation response is lost: retry with the same idempotency key and receive the existing session.
- Signed URL expires: authenticate again and request a replacement for a missing part; do not broaden the grant.
- Part response is lost: reconcile the provider's committed-part list before re-uploading.
- Completion races: use a per-session lock or conditional state/version update; only one logical completion proceeds.
- Storage completes but DB is unavailable: reconcile by random object key/session ID before retrying or cleaning up.
- Outbox or queue unavailable: retain undelivered outbox rows and keep the file unavailable until processing is durably scheduled.
- Duplicate queue event: verify file state and deterministic job/output identity before work or state transition.
- Scanner unavailable: preserve quarantine state and apply backpressure; do not bypass required validation.
- Delete arrives during an upload: revoke future URL issuance, abort session, and clean late-arriving object parts.
