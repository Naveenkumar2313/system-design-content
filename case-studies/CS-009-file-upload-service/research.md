# CS-009 — Design a File Upload Service

> **Content status:** Educational reference design. This document proposes a generic, scalable file-upload service; it does not claim to describe any named company's private production architecture. Workload, capacity, performance, and availability figures are illustrative assumptions.
>
> **Source alignment:** The Codorythm case-study catalogue identifies this as Case Study 9 in Category A: Web & Internet Systems. The required structure follows the 18-section case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-009 |
| System name | File Upload Service |
| Real-world reference | Google Drive, Dropbox, OneDrive, WhatsApp, Instagram, and YouTube are product examples, not asserted system designs |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 9 |
| Difficulty | Beginner to Intermediate |
| Short description | Design a secure service that authorizes uploads, transfers small or large files reliably to object storage, verifies integrity, processes files asynchronously, and exposes files only after required validation. |
| Primary users | Web/mobile clients, end users, application teams, trust and safety teams, storage and reliability operators |
| Key concepts | Direct-to-object-storage upload, pre-signed URLs, multipart/resumable transfer, metadata, checksums, asynchronous processing, scanning, quotas, lifecycle, CDN delivery |
| Related component IDs | CMP-001 (Load Balancer) for API ingress and replicated service traffic |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-14 |

### Scope and assumptions

- The service supports small single-request uploads and large multipart/resumable uploads through a client application.
- Application APIs authenticate users, authorize file ownership, create upload sessions, issue narrowly scoped short-lived upload credentials, and finalize metadata.
- File bytes normally travel directly from the client to private object storage; application servers do not proxy bulk bytes unless a workload explicitly requires inspection or protocol adaptation.
- Uploaded data first enters a quarantine/pending area. Downloads are unavailable until object integrity, policy validation, and required malware scanning finish.
- File metadata and workflow state live in a transactional database; file content lives in object storage. The database is not used to store large blobs.
- Background processing is asynchronous and idempotent. Exact storage consistency, multipart limits, checksum support, and URL behavior vary by provider.
- All capacities, retention periods, and SLOs below are illustrative teaching assumptions, not vendor benchmarks.

---

## 01 — Story

A user chooses a photo, video, document, or archive in a web or mobile application. A simple implementation sends the entire file through an application server and writes it to a local disk or database. That approach can work for small files, but long transfers consume application connections and memory, are vulnerable to network interruption, and couple upload throughput to the API tier.

A scalable file upload service separates authorization and workflow from byte transfer. The service validates a request, creates a durable upload record, and returns a short-lived URL or resumable session credential. The client transfers bytes directly to object storage, optionally as independently retryable parts. After completion, the service verifies the stored object, publishes a processing job, scans or transforms the file, and makes it available only when policy allows.

### Actors

1. **Client application:** Selects a file, requests an upload session, transfers bytes, and reports progress.
2. **API Gateway:** Terminates client TLS, enforces request-size/rate controls, and routes authenticated API calls.
3. **Authentication / authorization service:** Establishes user identity and checks upload, quota, ownership, and access permissions.
4. **Upload service:** Creates metadata and sessions, generates constrained upload credentials, verifies completion, and manages lifecycle state.
5. **Object storage:** Stores quarantined file content and processed derivatives.
6. **Message queue and workers:** Deliver durable processing jobs for scanning, transformation, and metadata extraction.
7. **Security / processing services:** Validate content, scan for malware, and generate permitted derivatives.
8. **Download client / CDN:** Retrieves a file only after authorization and availability policy permits access.
9. **Operations and trust team:** Monitors health, abuse, storage, policy outcomes, and audit events.

### Core user journey

1. User selects `vacation-video.mp4`; the client calculates size and an optional local checksum.
2. Client calls `POST /v1/uploads` with metadata and an idempotency key.
3. API authenticates the user; upload service checks allowed size/type, account quota, and destination policy.
4. Service creates a `PENDING_UPLOAD` metadata row and provider-side upload session when multipart is required.
5. Service returns a file ID, upload ID, part plan, and short-lived pre-signed URLs or a resumable session URL.
6. Client uploads directly to quarantine object storage and retries only failed parts.
7. Client submits completion with uploaded part numbers, provider validators, and expected checksum.
8. Service verifies the object/session and transitions metadata to `PENDING_SCAN`; an outbox event is delivered to the processing queue.
9. Workers inspect the object, verify integrity, scan it, and perform requested transformations.
10. A successful file becomes `AVAILABLE`; rejected content remains inaccessible and enters an audited cleanup or review path.
11. An authorized client requests metadata or a short-lived download URL; the application may deliver the object through a CDN.

### Problem boundaries

This case study designs file intake, transfer reliability, metadata, processing, and controlled retrieval. It does not design a complete identity system, object-storage implementation, malware-signature engine, image/video codec pipeline, or enterprise document editor. Object storage is the durable content system; metadata and workflow services decide who owns an object and whether it is safe to serve.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Authenticate a user and authorize upload access to a tenant, folder, or resource | Must |
| FR-02 | Validate file size, declared type, allowed extension, quota, and upload policy before transfer | Must |
| FR-03 | Create an idempotent upload session and durable metadata record | Must |
| FR-04 | Issue a short-lived, object-scoped upload URL or resumable-session credential | Must |
| FR-05 | Support single-request uploads for small files and multipart/resumable transfer for large or unreliable-network uploads | Must |
| FR-06 | Track uploaded parts/offsets and allow a client to query progress and resume | Must |
| FR-07 | Retry failed parts independently and handle duplicate part requests safely | Must |
| FR-08 | Complete or abort storage-side multipart sessions and clean up abandoned uploads | Must |
| FR-09 | Verify object existence, expected size, and cryptographic checksum before processing | Must |
| FR-10 | Store file metadata separately from object bytes and track upload/processing status | Must |
| FR-11 | Scan and process files asynchronously before marking them available | Must |
| FR-12 | Provide authorized metadata retrieval, download, and deletion APIs | Must |
| FR-13 | Support storage lifecycle, versioning/replication, and retention policy where required | Should |
| FR-14 | Emit audit events and operational metrics for upload, access, scan, and deletion | Must |
| FR-15 | Support progress reporting and cancellation from web/mobile clients | Should |

### Non-functional requirements

| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | API latency | Illustrative p95 under 200 ms for metadata/session APIs, excluding object transfer |
| NFR-02 | Upload reliability | Resume from committed part/offset after recoverable client or network interruption |
| NFR-03 | Availability | Illustrative 99.9% monthly for upload control APIs; object storage has an independent availability objective |
| NFR-04 | Integrity | Detect incomplete, corrupted, or mismatched uploads before making content available |
| NFR-05 | Scalability | Scale API, transfer, storage, queue, and workers independently |
| NFR-06 | Security | Enforce ownership, least privilege, short-lived credentials, encryption, quotas, and safe file handling |
| NFR-07 | Durability | Meet the selected storage service's durability and replication design; backup metadata and critical configuration |
| NFR-08 | Processing isolation | Treat all user files as untrusted; sandbox parsers and scanning/transcoding workloads |
| NFR-09 | Consistency | Publish an object as available only after storage completion and required processing state are durable |
| NFR-10 | Observability | Measure transfer, retries, integrity, processing backlog, storage use, and delivery outcomes |

### Out of scope for the initial version

- Collaborative editing and synchronization of file contents.
- Complete malware-signature engine internals or forensic analysis.
- Guaranteed deduplication across tenants.
- A complete CDN design; CDN delivery is a downstream option for authorized downloads.
- Provider-specific upload limits, prices, and performance guarantees.

---

## 03 — Scale

All values in this section are **illustrative workload assumptions for teaching and simulation**. They are not product disclosures or benchmarks.

### Workload assumptions

| Metric | Illustrative value | Reasoning |
|---|---:|---|
| New uploads | 1,000,000 files/day | Large consumer/work collaboration service exercise |
| Average stored file size | 10 MiB | Mixed images, documents, and small media; heavy tail handled separately |
| Daily incoming bytes | ~9.54 TiB/day | 1,000,000 × 10 MiB |
| Average upload byte rate | ~116 MiB/s (~0.97 Gbit/s) | Daily bytes / 86,400 sec, converted to decimal bits/sec |
| Peak transfer rate | ~8 Gbit/s | Approx. 8× daily average; not a hard capacity limit |
| Average upload-initiation rate | ~11.6 requests/sec | 1,000,000 / 86,400 |
| Peak upload-initiation rate | ~1,000 requests/sec | Simplified burst assumption for session API |
| Active concurrent transfers | 50,000 peak | Long transfer duration and regional demand |
| Multipart threshold | 64 MiB | Illustrative client/service policy, not a provider requirement |
| Multipart part size | 16 MiB default | Illustrative balance of retry cost, concurrency, and memory |
| Metadata per file | 2 KiB average | Excludes indexes, replicas, audit rows, and part records |
| Metadata growth | ~2 GiB/day raw | 1,000,000 × 2 KiB; database overhead is additional |
| Metadata retention | 365 days online | Product, compliance, and deletion requirements determine actual retention |
| Original object retention | 30 days after logical deletion before purge | Example grace period; legal/tenant policies may differ |

### Capacity notes

- An average upload size hides a long tail. Size bands, largest object, duration, and concurrent upload rates should be measured separately.
- Direct object-storage transfer moves bulk bytes out of application servers; the upload API still processes initiation, URL generation, completion, authorization, and status polling.
- At 50,000 simultaneous transfers, per-client connections to storage, outstanding multipart parts, URL/signature generation, and completion rates matter even when API RPS is modest.
- Storage growth must include replicas, object versions, incomplete multipart parts, derivatives, quarantine retention, lifecycle transitions, and deleted-object grace periods.
- Worker capacity depends on CPU/GPU cost, parser and malware-scan time, retry rates, and the number of derived outputs—not only file count.
- A checksum calculation is linear in file bytes; clients may compute it before transfer, while storage/provider checksum validation and server-side validation can add I/O.
- Plan for bursts from app releases, user imports, mobile reconnects, and retry storms. Backpressure should protect storage, metadata DB, and worker queues.

### Questions for a production sizing exercise

- What is the distribution of file sizes, types, and upload durations?
- What are peak simultaneous uploads, part concurrency per client, and regional network conditions?
- How much data is retained, versioned, replicated, transformed, and served per month?
- What scan/transform time and worker CPU/GPU resources are required per file type?
- Is an upload required to be available immediately after transfer, or only after asynchronous scanning?
- What are recovery-point and recovery-time objectives for metadata, objects, and processing state?

---

## 04 — Naive Architecture

### Simplest viable design

A client sends an entire file in one HTTP request to a single application server. The server authenticates the user, writes file bytes to a local disk, stores a metadata row in a database, and returns success.

**Components**
- Web/mobile client
- Single application server
- Local disk or shared filesystem
- Metadata database

**Request flow**
1. Client sends file bytes in one request.
2. Application authenticates the user and checks basic size/type limits.
3. Application streams or buffers the body to disk.
4. Application writes metadata and returns a file ID.
5. Client later retrieves the file through the same application server.

### Naive architecture diagram

See `architecture.png`.

### Why begin here?

This design makes the basic relationship between a file record and stored bytes clear. It also exposes why a single request and app-server-mediated data path fail for large files, unreliable networks, high concurrency, and horizontal scaling.

---

## 05 — Problems

| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Application server handles bulk bytes | Every upload consumes application bandwidth and connection resources | API capacity is dominated by data transfer; request timeouts rise | Issue scoped URLs for direct object-storage transfer |
| Whole-file restart after interruption | One request represents the complete transfer | Mobile or unstable-network clients lose all prior progress | Multipart/resumable sessions and per-part retries |
| Memory and file-descriptor pressure | Server buffers large bodies or keeps long-lived requests | Process exhaustion and collateral API failures | Stream with strict bounds or bypass app tier for bytes |
| Local disk is not shared durable storage | Files are tied to one host and deployments/scale change placement | Lost files, inconsistent downloads, difficult failover | Durable object storage with stable generated keys |
| Metadata/object inconsistency | DB row and file write are separate operations | Orphan records or untracked objects | Explicit state machine, completion verification, reconciliation |
| Duplicate client request | Network retry repeats initiation or completion | Duplicate metadata, object overwrite, extra billing | Idempotency keys and idempotent state transitions |
| Incomplete multipart parts | Client disappears before completion | Storage charges and orphaned parts | Session expiry, lifecycle abort rules, scheduled cleanup |
| No integrity verification | Network/client mistakes or corrupted parts go unnoticed | Invalid or incomplete content is exposed | Per-part and full-object checksums; verify size and storage metadata |
| Untrusted MIME type/name | Client controls headers and filename | Malicious content reaches parsers or active content is served | Allowlist, signature/content inspection, generated storage key, safe download headers |
| Synchronous scanning/transcoding | Upload response waits for expensive CPU/GPU work | Slow completion and poor capacity utilization | Durable queue and independently scaled workers |
| Queue delivery gaps | DB state and message publication are separate | Uploaded files never get scanned or processed | Transactional outbox, retries, dead-letter queue, reconciliation |
| Premature file visibility | Object is downloadable before scan and validation finish | Malware or invalid content is served | Quarantine object location and explicit AVAILABLE state gate |
| Quota race | Concurrent uploads check quota without reservation | Tenant exceeds storage limits | Atomic quota reservation and reconciliation against actual bytes |
| Hot metadata database | Status polling and part updates share one write path | Upload progress stalls or completion fails | Partition/index appropriately, batch updates, poll backoff, cache safe reads |
| Public storage exposure | Bucket or object ACLs are overly broad | Unauthorized file access | Private buckets, least-privilege credentials, application authorization |

---

## 06 — Architecture Evolution

These are proposed learning stages, not a historical account of any named provider's architecture.

### V1 — Application server receives full file

- Client posts file bytes to one upload server.
- Server authenticates, stores bytes on disk, writes metadata, and responds.
- Benefit: simple protocol and central validation.
- Limitation: application fleet becomes a bulk data plane; interruption restarts the whole request and local storage complicates failover.

### V2 — Durable object storage and metadata separation

- Store raw bytes in private object storage using a generated opaque key.
- Store owner, original filename, size, checksums, storage key, status, and timestamps in a relational metadata database.
- Keep original user filename as display metadata, not as the path or authorization token.
- Benefit: durable scalable byte storage and independent API scaling.
- New concern: metadata/object lifecycle is a distributed workflow requiring reconciliation.

### V3 — Direct upload with short-lived scoped credentials

- API authenticates and authorizes upload intent, reserves quota, and creates a pending metadata row.
- Service returns a short-lived pre-signed PUT URL or storage upload-session credential scoped to one random key and allowed action.
- Client transfers bytes directly to a quarantine bucket/container; API servers do not proxy the object body.
- Benefit: bulk transfer bypasses app servers and clients receive storage-native throughput.
- New concern: credential leakage, expiration, storage CORS policy, object overwrite, and the gap between object arrival and application completion must be controlled.
- The evolved architecture is shown in `architecture-evolved.png`.

### V4 — Multipart/resumable transfer and verified completion

- Initiate provider multipart/session state and persist provider upload ID and expiry.
- Issue per-part URLs or a resumable session URL; client tracks offsets, part numbers, checksums, and retries.
- Complete with a manifest of uploaded parts or provider session state; verify object size and checksum before accepting it.
- Make completion idempotent and serialize concurrent completion requests per upload.
- Benefit: recovery from network interruption without retransferring successful parts.
- New concern: part metadata, concurrent retries, orphan cleanup, provider limits, and completion races.

### V5 — Asynchronous scan and processing pipeline

- Atomically transition the file to `PENDING_SCAN` and record an outbox event.
- Publish to a durable queue; workers scan and transform in isolated environments.
- Store derivatives under versioned keys; transition metadata to `AVAILABLE` only after all required checks succeed.
- Use retries with backoff, dead-letter queues, idempotent workers, and reconciliation.
- Benefit: upload completion is decoupled from processing and unsafe content remains quarantined.
- New concern: eventual availability, duplicate deliveries, processing backlogs, and poison files.

### V6 — Multi-region, CDN, and lifecycle operations

- Replicate objects and metadata according to residency, durability, and recovery requirements.
- Use CDN or authorized object delivery for downloads; gate access on metadata state and permissions.
- Apply lifecycle rules to abandoned multipart uploads, quarantine objects, expired files, old versions, and derivatives.
- Scale upload APIs, object storage, DB partitions, workers, scanners, and delivery independently.
- Benefit: greater regional resilience and lower delivery latency.
- New concern: cross-region consistency, deletion propagation, data residency, cost, and incident coordination.

### Storage architecture

| Layer | Responsibility | Design notes |
|---|---|---|
| Upload API / control plane | Authentication, authorization, quotas, session state, URL issuance, completion | Does not proxy large file bytes in the direct-upload design |
| Metadata database | File ownership, storage key, status, size, type, checksum, retention, session and processing state | Index by file ID and owner; partition by tenant/time where scale requires |
| Quarantine object storage | Holds newly uploaded bytes before trust checks | Private access; random opaque object key; no public delivery |
| Clean object storage | Holds validated original and approved derivatives | Separate prefix/bucket or access boundary; lifecycle and versioning policy |
| Multipart/session manager | Maps application upload ID to provider session/upload ID and part state | Provider adapter handles S3 multipart, GCS resumable, Azure blocks, or equivalent |
| Message queue | Durable processing event delivery | At-least-once delivery is expected; consumers must be idempotent |
| Processing workers | Scan, inspect, extract, resize, transcode, or create thumbnails | Sandboxed, least privilege, bounded CPU/memory/time, isolated network |
| CDN / download delivery | Delivers approved files close to users | Authorization and state checks must happen before protected cache delivery |
| Monitoring / audit | Captures transfer, state transition, access, scan, deletion, and administrative events | Redact signed URLs and credentials |

### File lifecycle state machine

| State | Meaning | Allowed next states |
|---|---|---|
| `INITIATED` | Metadata and quota reservation created; bytes not yet complete | `UPLOADING`, `ABORTED`, `EXPIRED`, `FAILED` |
| `UPLOADING` | Provider upload session exists and bytes/parts are being transferred | `UPLOADED_PENDING_VERIFY`, `ABORTED`, `EXPIRED`, `FAILED` |
| `UPLOADED_PENDING_VERIFY` | Client requested completion; provider object/session is being verified | `PENDING_SCAN`, `FAILED` |
| `PENDING_SCAN` | Object is complete and quarantined; processing job is queued | `PROCESSING`, `REJECTED`, `FAILED` |
| `PROCESSING` | Scan/transform work is in progress | `AVAILABLE`, `REJECTED`, `FAILED` |
| `AVAILABLE` | Required policy checks passed; authorized downloads may be issued | `DELETING`, `REVOKED`, `FAILED` |
| `REJECTED` | Malware/policy/integrity validation failed; not downloadable | `DELETING`, `REVIEW_PENDING` |
| `DELETING` | Access revoked and object/derivatives being removed | `DELETED`, `FAILED` |
| `ABORTED` / `EXPIRED` | Session ended before accepted completion | `DELETING`, `DELETED` |
| `DELETED` | Logical and storage cleanup completed or retention policy records completion | Terminal |

State transitions use conditional updates/version checks. A file is downloadable only if its authoritative metadata state is `AVAILABLE` and the requesting principal passes authorization.

### Upload strategy comparison

| Strategy | Advantages | Disadvantages | Suitable use |
|---|---|---|---|
| Single-request upload | Simple client protocol and low metadata overhead | A failed long request may restart; proxy and timeout limits apply | Small files on reliable connections |
| Multipart upload | Independent part retries, optional parallelism, can begin before total length is known in some storage APIs | Requires session/part tracking and completion/abort cleanup | Large objects and unreliable networks |
| Chunked transfer encoding | Streams a request body without knowing total length in advance for HTTP/1.1 | Not equivalent to durable resumability; intermediaries may buffer or reject it | Streaming to a server endpoint when supported |
| Resumable upload protocol | Continues from committed offset/session after connection loss | Server/provider must maintain session state and protect session URI | Mobile networks, large files, long transfers |
| Direct-to-object-storage | Removes bulk bytes from app servers; scales with storage service | Requires scoped credentials, CORS, callbacks/completion verification, and consistency handling | Most browser/mobile large uploads |
| Upload through application server | Central inspection/control and simplified networking | App tier carries bandwidth, storage buffering, and long-lived connections | Small files or regulated workflows that require inline inspection |

Chunked HTTP transfer framing is a transport mechanism; it does not itself persist accepted chunks or let a client resume after a disconnect. Resumability requires an upload session and durable offset/part state.

### Multipart and resumable upload design

- **Chunk sizing:** Select a provider-compatible part size and balance retry cost, per-part overhead, parallelism, memory, and request timeouts. Enforce provider minimum/maximum part and count limits through the adapter.
- **Parallelism:** Allow bounded concurrent parts per upload and per user. Higher concurrency may improve throughput but increases client memory, storage/API operations, and origin pressure.
- **Session identity:** Use an unguessable application upload ID. Keep provider upload IDs private; map them to user, file, object key, expected size, checksum, expiry, and state.
- **Chunk tracking:** Persist part number/offset, expected length, observed ETag or provider checksum, upload status, and update time. Reconcile with provider `ListParts`/session status rather than blindly trusting client claims.
- **Retries:** Retry only failed/incomplete parts with exponential backoff and jitter; cap attempts and respect URL expiry. Re-issuing a URL for the same part should be safe under documented provider semantics.
- **Duplicate chunks:** Use a stable part number or offset range. Re-uploading the same part may replace that part in some provider APIs; completion must use the final provider-observed part list.
- **Resume:** Client queries session state or provider offset, verifies the source file still matches the expected file identity/checksum, then continues missing parts only.
- **Completion:** Lock or compare-and-swap session state; verify all required parts, contiguous ranges where applicable, total object size, and checksum before exposing the file.
- **Abort:** Revoke application session, request provider multipart abort/cancel, mark metadata aborted, and schedule cleanup for any in-flight parts that complete after abort.
- **Expiry:** Expire sessions after a bounded inactivity window; apply storage lifecycle rules to clean provider-side orphan parts. Exact session lifetime is provider-specific.

### File processing pipeline

1. API validates declared size, extension allowlist, and user/tenant quota before issuing upload credentials.
2. Completion verifies stored size, object key, expected metadata, and cryptographic checksum; client-supplied MIME type alone is not trusted.
3. Worker detects actual content type using signature/content inspection and applies allowlisted policy.
4. Antivirus/malware scanner inspects the quarantined object; optionally use content disarm/reconstruction for supported document formats.
5. Type-specific workers create thumbnails, normalize images, extract document metadata, or transcode video in a sandbox.
6. Derived objects are written to versioned clean keys; original retention follows policy.
7. Worker records structured results and an idempotent state transition; only passed content becomes `AVAILABLE`.
8. Failed or suspicious files remain inaccessible, are quarantined or deleted according to policy, and produce auditable events.

Workers must enforce decompression limits, parser timeouts, memory/CPU caps, archive expansion limits, and restricted network access. A valid file signature is useful evidence but not a sufficient safety guarantee.

### API design

All endpoints require authenticated TLS and object/tenant authorization. Examples use JSON for control-plane APIs; data bytes travel directly to object storage. Use a request `Idempotency-Key` for operations that may be retried.

#### Initiate an upload

`POST /v1/uploads`

Request:

```json
{
  "filename": "report.pdf",
  "size_bytes": 52428800,
  "content_type": "application/pdf",
  "checksum": {
    "algorithm": "SHA-256",
    "value": "base64-encoded-digest"
  },
  "purpose": "workspace-document",
  "multipart": true
}
```

Response (`201 Created`):

```json
{
  "file_id": "fil_01J...",
  "upload_id": "upl_01J...",
  "status": "UPLOADING",
  "storage_mode": "multipart",
  "part_size_bytes": 16777216,
  "part_count": 4,
  "expires_at": "2026-11-15T19:00:00Z"
}
```

The server generates a random object key; it does not use a client-supplied filename as a storage path. It reserves quota and persists metadata/session state before returning success.

#### Request URLs for parts (or a single PUT)

`POST /v1/uploads/{upload_id}/parts:sign`

Request:

```json
{
  "parts": [
    { "part_number": 1, "content_length": 16777216 },
    { "part_number": 2, "content_length": 16777216 }
  ]
}
```

Response (`200 OK`):

```json
{
  "upload_id": "upl_01J...",
  "urls": [
    {
      "part_number": 1,
      "url": "https://storage.example/short-lived-signature",
      "expires_at": "2026-11-14T20:00:00Z",
      "required_headers": { "Content-Length": "16777216" }
    }
  ]
}
```

For small files, initiation may return a single `PUT` URL scoped to one generated key and method. Signed URLs are bearer credentials: do not log them, expose them in analytics, or allow them to select arbitrary object keys.

#### Upload bytes

The client sends `PUT` to each signed storage URL with the required headers. For a provider resumable session, the client sends the provider-defined `PUT`/range requests to the session URI. This data-plane request does not pass through the application API. Provider responses may include ETag, part checksum, or accepted byte range; clients preserve these values for status/completion.

#### Check progress / resume

`GET /v1/uploads/{upload_id}`

Response (`200 OK`):

```json
{
  "upload_id": "upl_01J...",
  "file_id": "fil_01J...",
  "status": "UPLOADING",
  "expected_size_bytes": 52428800,
  "received_parts": [
    { "part_number": 1, "size_bytes": 16777216, "etag": "\"part-etag\"" }
  ],
  "missing_parts": [2, 3, 4],
  "expires_at": "2026-11-15T19:00:00Z"
}
```

The response contains no provider secret or reusable upload URL. The client asks the sign endpoint for new URLs only for missing parts.

#### Complete multipart upload

`POST /v1/uploads/{upload_id}:complete`

Request:

```json
{
  "parts": [
    { "part_number": 1, "etag": "\"part-1-etag\"" },
    { "part_number": 2, "etag": "\"part-2-etag\"" },
    { "part_number": 3, "etag": "\"part-3-etag\"" },
    { "part_number": 4, "etag": "\"part-4-etag\"" }
  ],
  "checksum": {
    "algorithm": "SHA-256",
    "value": "base64-encoded-full-object-digest"
  }
}
```

Response (`202 Accepted`):

```json
{
  "file_id": "fil_01J...",
  "status": "PENDING_SCAN",
  "processing_status_url": "/v1/files/fil_01J..."
}
```

Completion is idempotent. Repeated requests return current state rather than completing a different object or publishing another processing event. Provider multipart ETags are not universally equivalent to a full-object MD5/SHA checksum; use explicit supported checksum fields/algorithms and verify semantics with the selected provider.

#### Abort upload

`DELETE /v1/uploads/{upload_id}`

Response: `202 Accepted` with `{ "upload_id": "...", "status": "ABORTING" }`. The service revokes application state, asks storage to abort, releases quota only after safe reconciliation, and cleans up any late-arriving parts.

#### Retrieve metadata

`GET /v1/files/{file_id}`

Response (`200 OK`):

```json
{
  "file_id": "fil_01J...",
  "filename": "report.pdf",
  "size_bytes": 52428800,
  "content_type": "application/pdf",
  "status": "AVAILABLE",
  "created_at": "2026-11-14T19:42:00Z"
}
```

The service checks tenant and owner permissions before returning metadata. Internal storage keys, scan details, and signed URLs are not exposed by default.

#### Download a file

`POST /v1/files/{file_id}:download`

Response (`200 OK`):

```json
{
  "url": "https://download.example/short-lived-authorized-url",
  "expires_at": "2026-11-14T19:52:00Z"
}
```

Only an `AVAILABLE` file with an authorized requester receives a URL. For private CDN delivery, authorization is validated on every request or by a suitable signed token; the cache policy must not leak content between principals.

#### Delete a file

`DELETE /v1/files/{file_id}`

Response: `202 Accepted` with `{ "file_id": "...", "status": "DELETING" }`. The API first revokes download authorization, then asynchronously deletes the original, derivatives, replicas, and metadata according to retention and legal-hold policy.

#### API status codes

| Status | Meaning |
|---|---|
| `200 OK` | Status, metadata, or idempotent operation result returned |
| `201 Created` | Upload metadata/session created |
| `202 Accepted` | Completion, abort, or deletion accepted for asynchronous work |
| `400 Bad Request` | Malformed request, invalid parts, or checksum format |
| `401 Unauthorized` | Missing or invalid authentication |
| `403 Forbidden` | User lacks ownership/tenant permission or upload policy denies action |
| `404 Not Found` | File/upload absent or intentionally hidden from unauthorized caller |
| `409 Conflict` | State does not allow operation, idempotency key reused with different payload, or object completion conflict |
| `413 Content Too Large` | Declared or verified size exceeds configured limit |
| `415 Unsupported Media Type` | Type is outside the configured allowlist |
| `422 Unprocessable Content` | Uploaded object fails integrity or content validation |
| `429 Too Many Requests` | Rate limit or quota/active-session limit exceeded |
| `503 Service Unavailable` | Temporary dependency failure; include bounded retry guidance |

### Practical storage provider notes

- **Amazon S3:** Multipart upload uses initiate, upload parts, and complete/abort operations. Part numbers and ETags are supplied at completion; an object's multipart ETag should not be assumed to be a full-object MD5. S3 supports checksums and presigned URLs; constrain signed requests and clean up incomplete multipart parts.
- **Google Cloud Storage:** Resumable uploads use a session URI and allow transfer to continue after connection interruption. Treat the session URI as a sensitive bearer credential and follow the API's offset/status procedure.
- **Azure Blob Storage:** Block blobs can be staged as blocks and committed as a block list; SDK transfer behavior and limits are version-dependent. Use scoped identity/SAS and verify block/commit semantics.
- Keep provider-specific behavior behind a storage adapter. Do not assume multipart semantics, checksums, expiry, maximum object size, or consistency behavior are identical across services.

---

## 07 — Components

| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Distributes API traffic across replicated API/upload service instances | Yes |
| TBD | Client Application | Selects file, computes optional local digest, transfers bytes, tracks progress, and resumes | Yes |
| TBD | API Gateway | TLS termination, authentication integration, rate/size limits, routing, and request IDs | Yes |
| TBD | Authentication / Authorization | Validates identity, tenant, ownership, folder access, and policy | Yes |
| TBD | File Upload Service | Validates upload intent, reserves quota, creates session, issues signed storage access, and finalizes state | Yes |
| TBD | Multipart Upload Manager | Tracks upload ID, provider session, part plan, progress, expiry, retries, completion, and abort | Large/resumable files |
| TBD | Metadata Database | Stores file, owner, storage key, size/type/checksum, status, and lifecycle records | Yes |
| TBD | Object Storage | Stores quarantine originals, approved objects, and derivatives | Yes |
| TBD | Transactional Outbox | Persists processing events atomically with metadata state transition | Recommended |
| TBD | Message Queue | Delivers asynchronous scanning and transformation jobs | Yes |
| TBD | Background Workers | Perform scanning, validation, extraction, thumbnails, and transcoding | Workload-dependent |
| TBD | Virus / Malware Scanner | Detects known malware and policy-violating content | Yes for untrusted upload |
| TBD | CDN / Download Service | Delivers authorized, approved files near clients | Optional |
| TBD | Lifecycle / Reconciliation Worker | Aborts stale multipart sessions, deletes expired objects, and repairs orphan metadata | Yes |
| TBD | Monitoring / Audit Pipeline | Collects request, storage, processing, access, security, and lifecycle telemetry | Yes |

**Component ID policy:** CMP-001 is the only confirmed reusable component relevant to this case study in the starter package. Other IDs remain TBD until component specifications are created and added to the master component index. The metadata database stores file facts and workflow state; object storage stores the file bytes.

### Component interactions

- Client → API Gateway → Authentication/Authorization → File Upload Service.
- Upload Service → Metadata DB: create session, reserve quota, persist state and object key.
- Upload Service → Object Storage: initiate provider upload and sign narrowly scoped part/single-object requests.
- Client → Object Storage: transfer file bytes directly; application servers do not proxy bulk data.
- Client → Upload Service: query status, request missing-part URLs, complete, or abort.
- Upload Service → DB/outbox → Message Queue → scanner/processing workers.
- Workers → Quarantine Object Storage: read and validate; workers → clean storage: write approved original/derivatives.
- Worker → Metadata DB: record result and transition to `AVAILABLE`, `REJECTED`, or `FAILED`.
- Authorized download API → Metadata DB/access policy → CDN or short-lived object URL.
- Lifecycle/reconciliation worker → DB/Object Storage/Queue: clean expired sessions, orphan parts, rejected content, and derivatives.
- All control-plane and worker transitions → monitoring and audit pipeline.

---

## 08 — Data Flow

### Flow A: Initiate, authorize, and direct upload

1. Client selects a file, reads size/name/type, and optionally computes a checksum.
2. Client sends an authenticated `POST /v1/uploads` with an idempotency key.
3. Gateway applies request-rate and control-plane body limits; authentication identifies the principal.
4. Upload service checks tenant permission, quota, declared size, allowlisted extension/type, and policy.
5. Service generates file ID and random storage key, reserves quota, and writes `INITIATED` metadata/session state.
6. For a small file, service creates a single-object upload grant; for a large file, service initiates a storage multipart/resumable session.
7. Service returns a short-lived URL/session credential and transfer instructions.
8. Client transfers bytes directly to private quarantine storage and reports progress locally.
9. Object-storage events may be used as hints, but the API completion and authoritative provider verification determine workflow acceptance.

### Flow B: Multipart transfer, retry, and resume

1. Client requests signed URLs for a bounded set of part numbers.
2. Client uploads parts in parallel up to a configured per-upload concurrency limit.
3. Storage returns part ETags/checksums; client records them and reports progress.
4. A network failure leaves completed parts in storage; the client calls `GET /v1/uploads/{id}`.
5. Upload service reconciles provider part/offset state and returns received and missing parts.
6. Client requests new signed URLs for missing/expired parts only.
7. A duplicate upload for a part is handled according to provider semantics; completion uses the final provider-observed part list.
8. Client sends a completion request with the part manifest and expected checksum.

### Flow C: Verify, complete, and enqueue processing

1. Upload service atomically transitions session to `UPLOADED_PENDING_VERIFY` and acquires a per-session completion lock/version.
2. Service asks object storage to complete multipart upload or checks resumable session completion.
3. Service verifies object key, exact size, storage-side checksum where supported, expected metadata, and object visibility.
4. Service persists `PENDING_SCAN` metadata and an outbox event in one database transaction.
5. Outbox publisher sends a processing job to a durable queue and marks the event delivered.
6. Repeated completion requests return current state; they do not create multiple logical files or processing events.

### Flow D: Scan, transform, and publish

1. Worker consumes the job and leases it with an idempotency key and bounded visibility timeout.
2. Worker reads the quarantine object using a narrowly scoped service identity.
3. Worker checks actual file signature/content, archive expansion limits, malware, and requested processing policy.
4. Approved transforms run in a sandbox with CPU/memory/time limits and restricted network access.
5. Worker writes clean original/derivatives to versioned clean object keys and verifies writes/checksums.
6. Worker atomically transitions metadata to `AVAILABLE` and records results; rejected content remains inaccessible and enters cleanup/review.
7. Worker acknowledges the queue message only after durable state is committed; retry or dead-letter handling covers failures.

### Flow E: Progress, download, and deletion

1. Client polls upload or file status with backoff, or receives an application event when supported.
2. `GET /v1/files/{id}` returns permitted metadata and current state.
3. Download request checks identity, ownership, file status, legal hold, and access policy.
4. Service returns a short-lived signed URL or streams through an authorized download layer/CDN.
5. Deletion first marks file `DELETING` and revokes new download authorization.
6. Asynchronous cleanup removes original, derivatives, replicas, and cached copies according to retention policy, then records `DELETED`.

### Flow F: Abandoned upload cleanup

1. Lifecycle worker selects sessions past expiry that are not in a completion transition.
2. Worker conditionally marks the session `EXPIRED` using a version check.
3. Worker revokes application state, aborts provider multipart/resumable state, and deletes quarantine objects if present.
4. Reconciliation checks object storage for late parts/objects and metadata for missing or orphan records.
5. Quota reservation is released only after the actual stored bytes/session state are reconciled.
6. Cleanup errors are retried and surfaced; they are not silently treated as success.

### Error and edge-case paths

- Network interruption: resume from provider-committed parts/offset; retry only missing data.
- Signed URL expiration mid-part: request a new URL for the same part/session after ownership and state checks.
- Duplicate initiation: same idempotency key and payload returns the original session; mismatched payload returns `409 Conflict`.
- Duplicate completion: return current durable state without re-enqueueing work.
- Checksum or size mismatch: do not publish; mark failed/quarantined, release reservation after reconciliation, and log security/integrity event.
- Storage event arrives before completion API: treat it as a hint; verify application session, owner, object key, and expected size.
- Queue publishes twice: worker idempotency and conditional state transition prevent duplicate visible derivatives.
- Worker crashes after writing output: deterministic/versioned output key and job state allow safe retry or cleanup.
- Scanner unavailable: keep file pending and inaccessible; apply queue backpressure rather than bypassing scan.
- User deletes during upload: revoke session, abort provider transfer, and handle late part/object writes through cleanup.
- Metadata DB unavailable: stop creating/completing sessions; do not return success-shaped response without durable state.
- Object storage unavailable: control APIs may report dependency failure; no upload is marked complete until provider verification succeeds.

---

## 09 — Failure Scenarios

| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| Network interruption during upload | Client timeout, connection reset, incomplete range/part | Current part fails; prior parts may remain durable | Resume session, list committed parts/offset, retry only failed parts with backoff |
| Upload API instance failure | Load balancer health and request errors | Control calls fail; direct storage transfer already in progress may continue until URL expiry | Retry idempotent API calls against another instance; preserve session state in DB |
| Object storage outage/throttling | Provider errors, elevated latency, failed PUT/complete | Parts or final object unavailable | Respect retry-after, bounded retries, pause new uploads, keep state resumable |
| Metadata database outage | Connection/transaction errors and DB health alerts | Cannot safely create sessions, complete, or authorize status | Fail closed for state changes; retry with idempotency key after recovery; restore from backups/replicas |
| Multipart completion failure | Provider completion error or missing part/ETag | Object remains incomplete; parts may continue to incur storage charges | Reconcile provider part list, correct manifest, retry idempotently or abort and restart |
| Duplicate initiation request | Reused idempotency key | Potential duplicate sessions or quota reservation | Persist key + request hash; return original result for exact replay; reject mismatched replay |
| Duplicate part upload | Client retry or lost response | Part may be overwritten or duplicated depending on provider | Stable part number/offset, checksum validation, final provider-state reconciliation |
| Incomplete/abandoned upload | Session inactivity and old multipart records | Orphan parts consume storage and quota | Expiry worker, provider lifecycle abort rule, periodic reconciliation |
| Corrupted object / checksum mismatch | Provider checksum or application verification | Object must not be made available | Mark failed/quarantined, retry transfer where possible, preserve diagnostic evidence |
| Malware or prohibited content | Scanner/policy result | Content must not be served | Mark rejected, revoke URL, quarantine/delete under policy, audit and notify |
| Processing worker failure | Queue visibility timeout, heartbeat, job status | File remains pending; work may repeat | Idempotent worker, retry with backoff, dead-letter poison jobs and alert |
| Message queue failure | Publish backlog/error, consumer lag | Completed object awaits processing | Transactional outbox retries publication; maintain pending state; drain queue after recovery |
| Storage capacity/quota exhaustion | Provider quota errors, tenant usage alarms | New uploads fail or existing uploads stall | Quota reservation, lifecycle cleanup, capacity alerts, controlled rejection, tiering |
| Virus scanner unavailable | Health probes and scan queue age | Untrusted files accumulate in quarantine | Keep files unavailable; scale/restore scanner; do not skip required scanning |
| CDN/download cache stale after deletion | Access tests and purge telemetry | Deleted content may remain accessible from cache | Revoke authorization first, purge where needed, use short-lived access and immutable version keys |
| Region failure | Regional health, elevated API/storage errors | Uploads or completion may fail in affected region | Multi-region control plane/storage as required; resumable clients retry after failover; reconcile writes |
| Quota reservation race | Usage ledger differs from actual bytes; concurrent grant test | Tenant exceeds policy or is blocked prematurely | Atomic reservation/limits, idempotent accounting, periodic reconciliation |
| Signed URL leaked | Audit anomaly, unusual source/use patterns | Unauthorized actor may upload to scoped key until expiry | Very short TTL, exact method/key/headers, revoke session, rotate credentials, audit and quarantine object |

Retries must be bounded and jittered. Never retry a non-idempotent storage completion blindly without checking provider state, and never mark a file available merely because the client claims that upload succeeded.

---

## 10 — Scaling

### Control-plane and data-plane scaling

- **Horizontal API scaling:** Run stateless gateway/upload-service instances behind a load balancer. Persist session, idempotency, quota reservation, and state in shared durable systems.
- **Vertical scaling:** Increase CPU/memory for metadata APIs or scanners when profiling shows a per-node limit; it does not replace horizontal failure redundancy.
- **Direct uploads:** Let clients send file bytes to object storage using scoped credentials. This decouples bulk throughput from API server bandwidth and connection limits.
- **Admission control:** Bound active sessions, bytes in flight, parts per session, per-user concurrency, and per-tenant daily usage.
- **Regional endpoints:** Select storage region by residency and user location while keeping object key and metadata routing consistent.

### Metadata and session scaling

- Index metadata by `(tenant_id, file_id)` and owner/time for common authorization/list queries.
- Keep part/session state scoped to active uploads; expire and compact completed part rows.
- Partition large metadata tables by tenant or creation time only when measured query/write volume requires it.
- Avoid status polling hot spots with exponential backoff, push notifications where appropriate, and cached reads for non-authoritative display.
- Use atomic quota reservations and a usage ledger; asynchronous storage accounting reconciles actual size with reserved size.
- Keep storage provider IDs and signed URL material out of public metadata responses and logs.

### Storage and processing scaling

- **Object storage:** Choose storage class and region based on durability, access frequency, latency, residency, and lifecycle. Use generated keys and avoid hot prefixes where the provider's guidance requires distribution.
- **Multipart transfer:** Scale part size and bounded parallelism to network conditions, client memory, retry behavior, and provider limits.
- **Queue and workers:** Scale consumers based on oldest job age, backlog, bytes-to-process, and CPU/GPU cost; apply per-tenant fairness and poison-job isolation.
- **Scanning:** Keep scanner capacity and signatures current; shard workload by object size/type and sandbox untrusted parsers.
- **CDN:** Cache only approved/authorized downloadable content; invalidate or revoke after deletion and policy changes.
- **Lifecycle:** Abort incomplete multipart sessions, expire quarantine/rejected objects, transition cold files, and delete old versions according to policy.

### High availability and multi-region

- Replicate API instances across zones and keep session state in a replicated metadata store.
- Use storage replication/versioning based on recovery objectives and data-residency obligations.
- Design metadata backup, point-in-time recovery, and object inventory reconciliation separately.
- Decide whether an upload session is region-pinned or can resume in a failover region; provider session IDs may not work cross-region.
- Use queue replication/failover only with explicit duplicate and ordering semantics; workers remain idempotent.
- Test restore paths, deletion propagation, and scan-state recovery rather than assuming object replication alone restores service.

Thousands of concurrent uploads should primarily consume object-storage capacity and client-to-storage bandwidth, not one application worker per file. Millions of daily uploads require independent capacity planning for session APIs, object bytes, part operations, metadata writes, queue throughput, scan compute, and cleanup.

---

## 11 — Trade-offs

### Upload path and reliability

| Strategy | Strength | Limitation |
|---|---|---|
| App-server proxy | Central inspection and one client-facing endpoint | Application tier carries bulk bandwidth and long-lived connections |
| Direct single PUT | Simple and efficient for small files | Retry often repeats the entire object; URL expiration can interrupt a long transfer |
| Multipart upload | Independent part retry and bounded parallelism | Requires part state, provider-specific completion, and orphan cleanup |
| Resumable session | Continues from accepted offset after disconnect | Session URI/state is sensitive; provider protocol and expiry must be handled |
| Proxy through inspection gateway | Inline visibility/control | Expensive and may bottleneck large files; use only when policy demands it |

### Metadata/object consistency

| Design | Benefit | Cost |
|---|---|---|
| DB state machine + outbox | Durable workflow transition and eventual queue delivery | Additional outbox publisher and reconciliation |
| Object-storage event as sole trigger | Low-latency event-driven pipeline | Events can duplicate, delay, or arrive out of order; must verify against metadata |
| Completion API as sole trigger | Explicit client intent | Client may disappear after storage completion; reconciliation is still needed |
| Combined completion + storage event + reconciliation | Better recovery and duplicate tolerance | More state paths and operational complexity |

### Chunk sizing and parallelism

| Choice | Smaller parts | Larger parts |
|---|---|---|
| Retry cost | Less data retransmitted per failure | More data retransmitted per failure |
| Request overhead | More requests and part metadata | Fewer requests and less bookkeeping |
| Parallel throughput | Fine-grained concurrency | Efficient throughput if network remains stable |
| Memory/buffering | Lower per-part memory | Higher buffers and longer part retry windows |
| Constraints | Must satisfy provider minimum/count limits | Must satisfy provider maximum part/object limits |

Choose adaptive bounded part sizes rather than hardcoding one provider's limits as universal. Client-side parallelism should be limited to avoid exhausting mobile memory, radio/network capacity, or provider request budgets.

### Storage choices

| Choice | Option A | Option B | Considerations |
|---|---|---|---|
| File bytes | Relational DB BLOB | Object storage | DB transactions/joins vs. large-object cost and independent scaling |
| Object key | Original filename/path | Generated opaque key | Human readability vs. collisions, traversal, privacy, and authorization risks |
| Region | Single region | Multi-region replication | Simplicity/cost vs. RPO/RTO, consistency, residency, and replication expense |
| Deletion | Immediate physical delete | Logical revoke then async purge | Fast cleanup vs. reliability, legal hold, and residual replicas/caches |
| Integrity | Trust ETag | Explicit supported checksum | Simplicity vs. multipart/provider-specific ETag semantics |

### Required decision record format

For every major decision, interns should document:
- Context and problem.
- Options considered.
- Chosen option.
- Why it was selected.
- What is gained.
- What is sacrificed.
- Conditions that would justify revisiting it.

---

## 12 — Cost

No provider, region, storage class, egress price, request mix, or current quote is specified. Therefore, this section provides a cost model rather than a fabricated currency estimate.

### Cost categories

| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| Object storage | Stored byte-hours, object versions, storage tier | Logical and physical bytes by lifecycle/status |
| Upload requests | PUT, multipart-part, list, complete, abort, and metadata operations | Requests by method/provider/region |
| Network ingress/egress | Data transfer into storage and out to clients/workers/regions | Bytes by source, destination, and direction |
| Replication | Cross-zone/region replicated bytes and operations | Replication policy, lag, and duplicate bytes |
| Metadata database | Provisioned compute, storage, indexes, replicas, backup | Rows, write rate, index size, retention |
| Queue and messaging | Messages, payload size, retries, retention | Jobs/file, events/job, backlog, DLQ volume |
| Scan/processing workers | CPU/GPU seconds, memory, scratch space | Processing time by type/size and retry rate |
| Malware scanning | Per-byte, per-file, or scanner capacity | Files/sec, bytes/sec, signature/update tier |
| CDN/download | Request, egress, cache fill, invalidation | Delivered bytes, hit ratio, region, purge frequency |
| Logs and audit | Ingestion, query, retention | Events/sec, bytes/event, retention requirements |
| Incomplete upload cleanup | Orphan part storage and reconciliation jobs | Abandoned-session rate and age |
| Key management | KMS operations, HSM, encryption policy | Key count, request rate, envelope operations |

### Estimation worksheet

Populate using selected providers, regions, and current pricing:

- Monthly stored bytes = average daily incoming bytes × retention days × replication/version factor.
- Upload transaction cost = session API + part PUTs + complete/abort/list operations.
- Worker cost = files by type/size × average CPU/GPU seconds × worker unit price.
- Download delivery cost = viewer egress bytes × applicable regional egress/CDN pricing.
- Metadata cost = row/index/replica/backup storage plus API write/read capacity.
- Network cost = cross-region replication + scanner/worker reads + CDN fills + client egress.
- Cleanup cost = abandoned sessions × parts/list/abort operations plus retained incomplete-part byte-hours.

Use the 1,000,000 files/day, 10 MiB average, and retention assumptions in Section 03 as inputs only. Include failed/retried parts, derivatives, replication, quarantine, deleted-object grace, and restore testing. Document date, region, currency, storage tier, discounts, and egress assumptions.

---

## 13 — Security

### Threats and controls

| Threat | Mitigation |
|---|---|
| Unauthorized upload | Authenticate identity, authorize tenant/folder, use least privilege, and audit initiation |
| Pre-signed URL theft | Short expiry, one object/action, exact method/headers, avoid logs/referrers, revoke session where possible |
| Object overwrite | Generate unique random keys; use conditional writes/versioning where supported; never sign a user-controlled existing key |
| Malicious file/parser exploit | Quarantine first, scan content, sandbox parsers, patch libraries, and restrict worker network/privileges |
| Spoofed MIME type/extension | Allowlist, decode/normalize filename, inspect file signature/content; do not trust client `Content-Type` |
| Path traversal / filename attack | Use opaque storage key; store original name as sanitized metadata; safe `Content-Disposition` encoding |
| Decompression/archive bomb | Bound expanded bytes, file count, nesting, CPU, memory, and processing time |
| Cross-tenant file access | Verify ownership/tenant on every metadata/download/abort/delete operation; isolate storage credentials |
| Storage bucket exposure | Private-by-default container, block public ACLs, narrow service identity, periodic access review |
| Malware served before scan | Quarantine namespace and `AVAILABLE` state gate; download path checks authoritative state |
| Checksum manipulation | Select approved algorithm, verify provider/client checksum semantics, and bind digest to upload session |
| CSRF / confused deputy | CSRF protection for cookie-auth APIs; signed URL scope and CORS are not substitutes for authorization |
| Quota/DoS exhaustion | Per-user/tenant quotas, active-session limits, rate limits, size ceilings, and abuse monitoring |
| Encryption/key exposure | TLS in transit, encryption at rest, key rotation, secret manager, and least-privilege KMS access |
| Signed download sharing | Short validity, resource-scoped signature, optional IP/use policy where supported, and access audit |
| Queue/job injection | Publish server-generated job payloads; validate object key against metadata and tenant before processing |
| Sensitive metadata leakage | Redact names/PII where required; restrict logs; avoid credentials and signed URLs in traces |

### Validation rules

- Enforce maximum size and quota before issuing storage credentials and verify actual size after upload.
- Allow only business-required file types; validate normalized extension, declared MIME type, and file signature together.
- Generate opaque keys and keep uploads in private quarantine storage until scanning and policy checks pass.
- Do not use a multipart ETag as a universal whole-file checksum; use a documented checksum algorithm supported by storage.
- Scope signed upload access by key, method, headers, and expiry; configure CORS to approved origins and methods.
- Enforce ownership and `AVAILABLE` status before issuing downloads; CDN caching must preserve authorization semantics.
- Use malware scanning and sandboxed processing for untrusted content; fail closed if required scanning is unavailable.
- Apply lifecycle deletion to orphaned uploads and preserve audit/legal-hold policy where required.
- Limit part count, part size, concurrent uploads, retries, upload duration, and processing resource consumption.

---

## 14 — Observability

### Metrics

| Metric | Type | Why it matters |
|---|---|---|
| `upload_sessions_created_total` | Counter | Session demand by tenant, region, and storage mode |
| `upload_bytes_received_total` | Counter | Transfer throughput and storage ingress |
| `upload_duration_seconds` | Histogram | Client transfer time by size, region, and protocol |
| `upload_active_sessions` | Gauge | Concurrency and session-store pressure |
| `upload_success_total` / `upload_failure_total` | Counter | End-to-end transfer outcome |
| `upload_part_retry_total` | Counter | Network/provider instability and retry amplification |
| `upload_part_latency_seconds` | Histogram | Identifies storage or client-path bottlenecks |
| `upload_resume_total` | Counter | Resumable behavior and interruption frequency |
| `upload_checksum_mismatch_total` | Counter | Corruption, client bug, or suspicious activity |
| `upload_completion_latency_seconds` | Histogram | Time from final byte to verified object state |
| `upload_storage_bytes{state}` | Gauge | Quarantine, clean, derivative, and incomplete-part storage |
| `upload_quota_rejection_total` | Counter | Capacity protection and tenant usage policy |
| `file_processing_queue_age_seconds` | Histogram / gauge | Oldest queued work and user-visible availability delay |
| `file_scan_duration_seconds` | Histogram | Scanner capacity and file-type cost |
| `file_processing_failure_total` | Counter | Poison content, worker bugs, and dependency failures |
| `file_available_latency_seconds` | Histogram | Time from initiation to usable/approved file |
| `upload_orphan_parts_total` | Gauge | Cleanup correctness and avoidable storage charges |
| `file_download_latency_seconds` | Histogram | CDN/object delivery performance |

### Derived indicators and alerts

- Upload success rate = completed accepted uploads / initiated valid uploads, segmented by file size and protocol.
- Transfer throughput = verified bytes / transfer duration; compare client-observed and storage-observed values.
- Retry rate = retried part attempts / part attempts; sustained increases indicate path, storage, or client issues.
- Processing delay = `AVAILABLE` timestamp − verified completion timestamp; alert on queue-age SLO.
- Storage growth = quarantine + clean originals + derivatives + versions + incomplete parts; compare with retention and quota budgets.
- Alert on completion/checksum errors, rising 4xx/5xx, storage throttling, DB latency, outbox age, queue backlog, scanner health, and orphan cleanup lag.
- Track download success and CDN hit ratio where applicable, but separate delivery performance from upload completion.

### Logging and tracing

- Use request ID, upload ID, file ID, tenant pseudonym, provider region, part number, state transition, size band, checksum algorithm, and outcome.
- Never log full pre-signed URLs, session URIs, access tokens, raw file content, or unnecessary PII.
- Trace initiation, signing, storage completion, outbox, queue, scan, transform, and publish stages with correlated IDs.
- Audit ownership changes, downloads of sensitive files, administrative purges/deletions, scan decisions, and key/config changes.
- Keep operational logs distinct from content metadata and apply retention/access controls.

---

## 15 — Simulation

### Learning objective

Let learners compare application-proxied, direct, multipart, and resumable uploads and observe how file size, part size, network interruption, retry policy, storage, queue, and worker capacity affect completion time, reliability, cost, and time to availability.

### Initial simulation controls

| Control | Default | Suggested range |
|---|---:|---:|
| Concurrent clients | 1,000 | 1–100,000 |
| File size | 100 MiB | 1 KiB–100 GiB |
| Average client bandwidth | 20 Mbit/s | 0.1–1,000 |
| Network interruption probability per part | 2% | 0–50% |
| Upload strategy | Multipart direct | Single / proxy / multipart / resumable |
| Part size | 16 MiB | 1–256 MiB |
| Parallel parts per client | 4 | 1–32 |
| URL/session TTL | 15 minutes | 1 minute–24 hours |
| Retry attempts | 5 | 0–20 |
| Object storage capacity | 10 Gbit/s | 100 Mbit/s–1 Tbit/s |
| Metadata API capacity | 1,000 requests/sec | 10–100,000 |
| Metadata DB write capacity | 2,000 writes/sec | 10–100,000 |
| Queue consumer capacity | 500 jobs/sec | 1–50,000 |
| Worker processing time | 30 seconds/file | 0.1–3,600 |
| Scanner failure rate | 0% | 0–20% |
| Tenant storage quota | 100 GiB | 1 MiB–100 TiB |
| Checksum verification | Enabled | On / Off |
| Origin/CDN download cache | Disabled | Enabled / Disabled |

These are simulator defaults, not provider limits or benchmarks.

### Derived values

These are simplified teaching-model equations. The implementation must document and test them.

- Single-stream transfer time = `file_size_bytes × 8 / effective_bandwidth_bits_per_second`.
- Multipart part count = `ceil(file_size_bytes / part_size_bytes)`, subject to selected provider limits.
- Effective parallel transfer rate ≤ `min(client_bandwidth, storage_capacity, client_parallelism × per-part rate)`.
- Expected transferred bytes include retries: `file_size + sum(retransmitted failed-part bytes)`.
- Upload success rate = completed valid files / initiated valid files.
- Average active sessions estimate = arrival rate × mean session duration (Little's Law).
- Metadata write load = initiation + part/session updates + completion + status + processing transitions + cleanup.
- Queue time depends on offered job rate versus worker service capacity; when offered rate exceeds capacity, show backlog/age and do not assume an unbounded queue.
- Time to availability = transfer + verification + queue wait + scan/processing + publish, not just client transfer time.
- Storage used = incomplete parts + quarantined files + clean originals + versions + derivatives, adjusted by retention/replication factors.
- Retry backoff uses bounded exponential delay plus deterministic-seed jitter; retry count and session/URL expiry constrain resumability.
- Direct upload consumes simulated client-to-storage bandwidth; proxy upload additionally consumes application ingress/egress and connection capacity.
- Checksum mismatch, scan failure, or policy rejection never yields an available/downloadable file.
- All throughput, latency, and cost results are simulated estimates, not measured provider behavior.

### Output metrics

- Active, initiated, completed, failed, resumed, and aborted uploads.
- Per-file transfer duration and throughput.
- Parts uploaded, retried, failed, and bytes retransmitted.
- API, metadata, object-storage, queue, and worker utilization.
- Time from upload initiation to `AVAILABLE`.
- Checksum and file-policy failures.
- Quarantine, clean, derivative, and incomplete-part storage.
- Quota usage, reservations, and rejected uploads.
- Successful, rejected, pending, expired, and timed-out work.
- Direct vs proxy application-bandwidth consumption.
- Estimated download latency and CDN hit ratio when enabled.

### Required scenarios

1. Compare small single-PUT upload with multipart overhead.
2. Interrupt a single-request transfer and compare it with resumable part retry.
3. Tune part size under low bandwidth, high bandwidth, and unreliable network conditions.
4. Increase per-client parallelism and observe throughput, memory proxy, and storage throttling.
5. Expire a signed URL during transfer and request replacement credentials for missing parts.
6. Retry initiation and completion with the same and conflicting idempotency keys.
7. Deliver duplicate/out-of-order storage events and verify idempotent job handling.
8. Fail object storage during part upload and multipart completion.
9. Fail metadata DB before and after object completion and observe recovery/reconciliation.
10. Disable the scanner and verify files remain quarantined and unavailable.
11. Saturate workers and observe queue age, backpressure, and time to availability.
12. Inject checksum mismatch, oversized archive, unsupported MIME, and malware verdict.
13. Expire abandoned multipart sessions and measure cleanup/storage reclaim.
14. Compare single-region and multi-region object replication with failover delay.
15. Enable CDN download delivery and test authorization, delete/purge, and cache behavior.

### Simulation model warning

The simulator must declare whether it models actual storage-provider limits or generic assumptions. It must not equate chunked transfer encoding with resumability, claim provider ETags are always full-file MD5 hashes, count an object as available before verification/scanning, or treat a signed URL as permanent user identity. Capacity and pricing outputs are estimates.

---

## 16 — Design Challenge

### Challenge: Build a reliable large-file upload workflow

Design a service for web and mobile users uploading documents, images, and videos over reliable and intermittent networks. It must transfer bytes efficiently, resume failures, protect storage and processing systems, and expose files only after integrity and policy checks.

**Requirements**
- Support a small single-request upload and a large multipart/resumable upload.
- Keep file bytes out of the application API tier in the primary design.
- Define metadata, upload session, part, and processing state.
- Design endpoints for initiation, signing, progress/resume, completion, abort, metadata, download, and deletion.
- Specify chunk sizing, parallelism, checksums, duplicate part handling, and cleanup.
- Add a durable asynchronous queue, malware scanning, and type-specific workers.
- Separate quarantine and approved content; gate downloads on authorization and `AVAILABLE` status.
- Protect users and service with quotas, size/type controls, short-lived credentials, encryption, and audit.
- Scale API, database, object storage, queue, scanners, and workers independently.
- Define multi-region, backup, lifecycle, and recovery behavior appropriate to the workload.

**Constraints for the exercise**
- Use Section 03's assumptions: 1,000,000 uploads/day, 10 MiB average, 50,000 peak active transfers.
- Target at least 99.9% successful completion for valid uploads under the baseline network profile.
- Recover from at least one interrupted part without retransmitting completed parts.
- Keep uploaded bytes unavailable until checksum and required malware scan pass.
- Inject client disconnect, storage throttling, metadata DB failure, queue outage, worker crash, and checksum mismatch.
- Ensure retry storms and abandoned multipart sessions do not exhaust quotas or storage.

**Learner deliverables**
1. Functional and non-functional requirements with workload assumptions.
2. Naive and evolved architecture diagrams.
3. Single upload, multipart/resume, completion, scan, and download flows.
4. REST endpoint and error/idempotency design.
5. Metadata schema and explicit state machine.
6. Storage layout, lifecycle, replication, and backup plan.
7. Queue/outbox, worker, scanning, and quarantine design.
8. Security, quota, ownership, and signed URL policy.
9. Failure/reconciliation plan and cost model.
10. A simulation plan with measurable acceptance criteria.

**Evaluation prompts**
- Why should the application server avoid proxying large file bytes?
- What durable state is needed to resume a transfer after the client restarts?
- How do you know the provider completed the expected object?
- Why is a multipart ETag not necessarily a full-file checksum?
- How does a duplicate completion request avoid duplicate processing?
- What happens if object completion succeeds but the metadata DB is unavailable?
- How are orphan parts and late in-flight writes cleaned up?
- How do you prevent a file from being downloaded before scan completion?
- How do authorization, CORS, and a pre-signed URL differ?
- How does quota reservation remain correct under concurrent uploads?

---

## 17 — AI Interview

### Interviewer question bank

| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What problems does a file upload service solve? | Large data path, reliability, metadata, processing, safety |
| Q02 | Beginner | Walk through direct-to-object-storage upload. | Auth, metadata, scoped URL, byte transfer, completion |
| Q03 | Beginner | Compare a single PUT and multipart/resumable upload. | Simplicity, retry granularity, overhead, provider semantics |
| Q04 | Intermediate | How does resume work after a mobile network interruption? | Session ID, provider parts/offset, reconciliation, retry only missing data |
| Q05 | Intermediate | Why not trust a filename, MIME header, or ETag? | Client control, spoofing, storage key safety, multipart ETag semantics |
| Q06 | Intermediate | How do you verify object integrity? | Expected size, explicit checksum, provider validation, no premature publish |
| Q07 | Intermediate | How do you handle duplicate initiation and completion? | Idempotency key/payload hash, conditional state transitions, durable event |
| Q08 | Intermediate | Why use an outbox and a queue for processing? | DB/event consistency, async worker scaling, at-least-once idempotency |
| Q09 | Intermediate | How do quota reservations work with concurrent uploads? | Atomic reservation, accounting, actual-byte reconciliation |
| Q10 | Advanced | What if storage completion succeeds but the DB write fails? | Reconciliation, provider object/session check, idempotent repair |
| Q11 | Advanced | How do you ensure malicious files cannot be downloaded? | Quarantine, scan state, authorization gate, fail closed |
| Q12 | Advanced | How would you select chunk size and parallelism? | Network, retry cost, memory, operation count, provider constraints |
| Q13 | Advanced | How do you clean abandoned multipart data? | Expiry, provider lifecycle abort, reconciliation, quota release after verification |
| Q14 | Advanced | What happens when the message queue or scanner is unavailable? | Outbox/backpressure, pending state, retry/DLQ, no scan bypass |
| Q15 | Advanced | How do you scale millions of uploads across regions? | Direct storage, independent tiers, metadata routing, replication, session affinity/failover |

### Follow-up evaluation dimensions

- Separates control-plane APIs, data-plane transfer, metadata, and object bytes.
- Explains resumability as durable session/offset state, not just HTTP chunking.
- Uses explicit state transitions and recovery for cross-system partial success.
- Treats provider checksums, ETags, limits, and session expiry as product-specific.
- Keeps uploaded content quarantined until integrity and safety checks succeed.
- Includes idempotency, bounded retries, cleanup, and quota reconciliation.
- Justifies cost and capacity using bytes, concurrency, parts, worker time, and retention.

---

## 18 — Architecture Score

### Proposed evaluation rubric

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements and workload assumptions | 10 | File sizes, concurrency, transfer conditions, retention, SLO |
| Architecture and data flow | 15 | API, direct storage, metadata, queue, workers, download |
| Upload reliability and integrity | 15 | Multipart/resume, retries, checksums, completion and abort |
| Metadata and lifecycle consistency | 15 | Schema, state machine, idempotency, outbox, reconciliation |
| Scalability and high availability | 15 | Independent tier scaling, region, replication, N+1 recovery |
| Security and file safety | 10 | Authorization, signed URLs, quarantine, scanning, quotas |
| Cost and trade-off justification | 5 | Storage, requests, network, workers, retention, CDN |
| Observability and operations | 10 | Transfer, retries, queue, scan, storage, download and audit signals |
| **Total** | **100** | |

### Scoring guidance

- 0: Missing or technically incorrect.
- 1: Mentioned without explanation.
- 2: Partially explained.
- 3: Correct and reasonably justified.
- 4: Detailed, measurable, and handles edge cases.

Weighted score = sum of (dimension score / 4 × dimension weight).

The score should be accompanied by evidence and reviewer comments; it should not be treated as an objective measure of architecture quality without human review.
