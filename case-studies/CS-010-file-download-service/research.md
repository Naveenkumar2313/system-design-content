# CS-010 — Design a File Download Service

> **Content status:** Educational reference design. This document proposes a generic, scalable file-download service; it does not claim to describe any named company's private production architecture. Workload, capacity, performance, and availability figures are illustrative assumptions.
>
> **Source alignment:** The Codorythm case-study catalogue identifies this as Case Study 10 in Category A: Web & Internet Systems. The required structure follows the 18-section case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-010 |
| System name | File Download Service |
| Real-world reference | Google Drive, Dropbox, OneDrive, Google Photos, and software-distribution platforms are product examples, not asserted system designs |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 10 |
| Difficulty | Beginner to Intermediate |
| Short description | Design a secure service that authorizes file access, locates immutable object versions, and delivers small or large files efficiently through object storage and a CDN, with HTTP range support and resumable downloads. |
| Primary users | Web/mobile clients, end users, application teams, content owners, reliability and security operators |
| Key concepts | Metadata lookup, authorization, object storage, pre-signed URLs, CDN caching, HTTP Range, conditional requests, resumability, streaming, observability |
| Related component IDs | CMP-001 (Load Balancer) for API ingress and replicated download-service traffic |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-14 |

### Scope and assumptions

- The service returns metadata and an authorized delivery capability; large file bytes normally flow from object storage or a CDN directly to the client rather than through application servers.
- The control plane authenticates the principal, authorizes access to a file/version, checks policy and quota, and issues a short-lived URL or streams through a trusted delivery layer.
- File metadata and access policy are separate from file content. Object keys are generated and are not derived from user-supplied filenames.
- The baseline supports complete downloads, single byte ranges, conditional requests, and resume with validators. Multipart byte ranges may be supported selectively; clients may issue multiple range requests for parallel download.
- Public immutable assets may be cached broadly. Private or revocable files use private-cache rules, authorization-aware cache keys, short TTLs, signed cookies/URLs, or an authenticated origin path. A CDN cache hit must never bypass authorization.
- Origin storage, metadata DB, identity system, CDN, DNS, and client networks are independent failure domains. Their specific consistency, limits, and failover behavior vary by provider.
- All capacities, cache ratios, SLOs, and transfer volumes below are illustrative teaching assumptions, not vendor benchmarks.

---

## 01 — Story

A user clicks a file in a web or mobile application and expects the download to start quickly, continue over an unreliable connection, and never expose content they are not allowed to see. A basic implementation reads a file from a server disk and streams it through an application process. That can work for a small deployment, but each byte consumes application bandwidth and connection capacity, distant users see high latency, and interrupted large downloads may have to start over.

A scalable file download service separates the control plane from the data plane. A client first asks for metadata and access authorization. The service then streams the file or returns a short-lived, narrowly scoped delivery URL. Object storage holds durable content; a CDN may cache eligible content near clients. HTTP validators and byte-range requests let clients continue an interrupted transfer without downloading bytes they already have.

### Actors

1. **Client application:** Requests file metadata, starts a download, follows redirects or signed URLs, tracks progress, and resumes with a byte range and validator.
2. **API Gateway / Load Balancer:** Terminates TLS for control APIs, applies request limits, routes calls to healthy service instances, and assigns request IDs.
3. **Authentication and authorization services:** Validate identity and evaluate ownership, tenant, role, ACL, policy, legal hold, and file state.
4. **File Download Service:** Resolves metadata and immutable object version, enforces download policy, issues delivery capabilities, and records an audit event.
5. **Metadata database:** Stores file identity, ownership, permissions, object location, version, status, size, checksum, and lifecycle facts.
6. **Object storage:** Stores durable file bytes and serves origin reads, including byte ranges where supported.
7. **CDN / edge cache:** Delivers eligible content near clients and retrieves cache misses from a protected origin.
8. **Monitoring and audit pipeline:** Records access decisions and aggregates download, origin, CDN, and failure telemetry without recording bearer secrets.
9. **Operations and content owners:** Manage retention, revocation, deletion, regional policy, and delivery health.

### Core user journey

1. User selects a file and an optional version in the client.
2. Client calls `GET /v1/files/{file_id}` or `POST /v1/downloads` using its authenticated session.
3. Gateway applies control-plane rate limits; identity is verified.
4. Download service checks file state and the caller's access to the requested version and purpose.
5. Service fetches minimal metadata and resolves the immutable object key, size, content type, and validators.
6. Service records an authorization decision and returns a short-lived signed CDN/object URL or a controlled streaming endpoint.
7. Client requests the file, optionally with `Range` and `If-Range`.
8. CDN serves an authorized cache hit or retrieves the exact object version from origin; origin access is restricted to the CDN/service identity.
9. Bytes stream to the client with correct `Content-Length`, `Content-Range`, and validator headers.
10. Client persists progress and validator; after interruption, it resumes from the local byte offset only if the remote representation still matches.
11. Completion or failure telemetry is recorded asynchronously; security-relevant access decisions remain auditable.

### Problem boundaries

This case study designs authorization, metadata resolution, delivery routing, caching, range/resume behavior, and operations. It does not design a complete identity provider, storage engine, CDN product, client filesystem, or the upload/processing pipeline that makes objects available. The service assumes a file becomes downloadable only after its owning system marks the appropriate immutable version available.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Authenticate callers and authorize access to a file, tenant, purpose, and version | Must |
| FR-02 | Retrieve file metadata and lifecycle status without exposing internal storage credentials | Must |
| FR-03 | Deliver complete files from an immutable object version through CDN or object storage | Must |
| FR-04 | Support HTTP byte ranges and correct `206`, `200`, and `416` semantics | Must |
| FR-05 | Support conditional requests using ETag, Last-Modified, If-Range, and cache validators | Must |
| FR-06 | Issue short-lived, narrowly scoped signed URLs or equivalent delivery capabilities | Must |
| FR-07 | Resume an interrupted download without refetching already received bytes when the representation is unchanged | Must |
| FR-08 | Restrict cache behavior according to public/private classification and revocation requirements | Must |
| FR-09 | Support file versions and ensure metadata, validator, cache key, and object key refer to the same version | Must |
| FR-10 | Record access decisions, download initiation, completion/failure signals, and administrative actions | Must |
| FR-11 | Enforce per-user/tenant quotas, rate limits, concurrency limits, and optional bandwidth policy | Should |
| FR-12 | Support deletion/revocation, retention, lifecycle, and cache purge or expiry workflows | Must |
| FR-13 | Support optional bulk/archive delivery for authorized multi-file requests | Could |
| FR-14 | Expose operational metrics for latency, throughput, cache, origin, errors, and completion | Must |

### Non-functional requirements

| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Control-plane availability | Illustrative 99.95% monthly for metadata and URL issuance, excluding client, DNS, and third-party delivery outages |
| NFR-02 | Download authorization | Every new download capability is tied to an authenticated policy decision; CDN hits cannot bypass it |
| NFR-03 | Metadata lookup latency | Illustrative p95 < 200 ms, excluding identity-provider and network latency |
| NFR-04 | Time to first byte | Illustrative p95 < 500 ms for warm, nearby CDN content; origin and client path vary |
| NFR-05 | Resumability | A valid resume request transfers only the requested remaining byte range when the object validator still matches |
| NFR-06 | Integrity | Returned bytes map to a specific immutable object version and length; optional client checksum verifies end-to-end content |
| NFR-07 | Confidentiality | TLS in transit, encryption at rest, least-privilege origin access, and no secret-bearing URL in logs |
| NFR-08 | Durability | Determined by selected object store, replication, backup, and recovery policy; no unsupported durability guarantee is assumed |
| NFR-09 | Scalability | API, metadata, CDN, and origin capacity scale independently; large byte streams do not consume app-server memory proportional to file size |
| NFR-10 | Cache correctness | Private content is not shared across principals unless policy explicitly authorizes shared caching |
| NFR-11 | Observability | Record request IDs, authorization outcomes, status codes, bytes, cache status, and bounded-latency distributions |
| NFR-12 | Fairness | Enforce configurable tenant concurrency and bandwidth controls without letting one tenant exhaust shared origin capacity |

### Out of scope for the initial version

- Uploading, malware scanning, transcoding, and content approval.
- Peer-to-peer delivery or client-side decryption key management.
- Real-time collaborative editing or content mutation during a download.
- Torrent protocols and public mirror management.
- Guarantees that an internet client will sustain a specific throughput.
- A custom CDN, DNS provider, identity provider, or object-storage implementation.

---

## 03 — Scale

### Workload assumptions

The following values create a sizing exercise; they are not claims about a real product.

| Input | Assumption | Why it matters |
|---|---:|---|
| Download requests | 1,000,000 per day | Metadata/API and access-audit volume |
| Average object size served | 25 MiB | Approximate data egress and origin/cache sizing |
| Daily bytes requested | 25,000,000 MiB ≈ 23.8 TiB | Egress cost and storage-read planning |
| Average delivered bandwidth | ≈ 2.4 Gbit/s over 24 hours | Baseline network sizing |
| Peak-to-average factor | 8× (assumed) | Illustrative peak egress ≈ 19 Gbit/s before CDN effects |
| Concurrent active transfers at peak | 50,000 | Connection, range, origin, and fairness planning |
| CDN hit ratio | 85% (assumed target for eligible content) | Origin offload; private/unique content may have lower hit rate |
| Metadata lookup per download | 1–2 reads | Database and cache pressure |
| Range/resume share | 15% (assumed) | Additional request count and validator correctness |
| Retention | 1–5 years by policy | Metadata, object lifecycle, deletion, and legal requirements |

### Capacity notes

- `1,000,000 downloads/day ÷ 86,400 ≈ 11.6` average download starts per second. A 20× burst would be about 232 starts/second, while long-lived transfers dominate concurrent connections and bytes.
- `25 MiB × 1,000,000 ≈ 25,000,000 MiB/day ≈ 23.8 TiB/day`. Dividing by 86,400 seconds gives about 2.4 Gbit/s average payload throughput (decimal bits and approximate units).
- An assumed 8× peak-to-average multiplier yields about 19 Gbit/s of peak payload delivery before protocol overhead. This is a planning input, not a measured traffic pattern.
- At an 85% byte hit ratio, a simplified origin payload estimate is 15% of delivered bytes. Real origin load also depends on object popularity, range behavior, cache key fragmentation, TTL, and revalidation.
- `50,000` concurrent transfers at average `1 MiB/s` would imply about `50,000 MiB/s` (roughly 419 Gbit/s) if every client sustained that rate. This illustrates why global CDN and client/network ceilings matter; concurrency is not the same as throughput.
- A 25 MiB file is not necessarily transferred all at once in one request. Range requests, retries, and parallel connections can increase request count while retaining similar payload bytes.
- Metadata read capacity is much smaller than data-plane throughput, but authorization, audit, and listing workloads can become bottlenecks if every byte-range request repeats expensive policy evaluation.
- Cache hit ratio must be measured both by request count and by bytes. A high request hit ratio can still leave large, unpopular objects on origin.

### Questions for a production sizing exercise

- What is the object-size distribution (p50/p95/max), and how many downloads are public versus private?
- What are regional user distribution, cacheability, popularity skew, and acceptable origin egress?
- What are required revocation time, URL lifetime, cache TTL, and deletion propagation?
- Are files immutable and versioned, or may an object change under the same public identity?
- How many concurrent transfers and parallel range requests are expected per user and per object?
- What are the RPO/RTO, residency, retention, legal-hold, and disaster-recovery requirements?
- Which clients support range requests, validators, parallel chunks, and persistent resume state?
- How should bandwidth, download counts, and abuse limits apply to tenants and shared links?

---

## 04 — Naive Architecture

### Simplest viable design

A client requests a file from one application server. The server authenticates and authorizes the user, reads the file from local disk or object storage, and streams bytes over the same application connection. Metadata lives in a database.

**Components**
- Web/mobile client
- Single application server
- Metadata database
- Local disk or object storage

**Request flow**
1. Client sends an authenticated `GET /files/{id}`.
2. Application checks permission and reads file metadata.
3. Application opens the file and streams bytes to the client.
4. Client receives the body; if the connection fails, it retries the full request.

### Naive architecture diagram

See `architecture.png`.

### Why begin here?

The simple design clearly identifies the trust decision and the relationship between metadata and bytes. It also exposes why a single application tier becomes a bandwidth, connection, availability, and geographic-latency bottleneck, and why full-file retries waste bandwidth.

---

## 05 — Problems

| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Application server proxies every byte | The API tier is in the data path | High egress cost, long-lived connections, CPU/socket pressure | Deliver via CDN/object storage after authorization |
| One server is a failure boundary | No redundant service or delivery path | A host failure stops downloads | Replicate stateless APIs and use durable storage/CDN |
| Distant users see high latency | Content comes from one origin region | Slow TTFB and poor throughput | Cache eligible objects at geographically close edges |
| Interrupted transfer restarts from zero | No Range/validator resume protocol | Wasted time and bandwidth for large files | Persist local offset and resume using Range + If-Range |
| Object changes under same URL | Mutable keys/weak cache identity | Clients combine bytes from different file versions | Use immutable versioned keys and strong validators |
| Authorization only at API, not CDN | Shared cache does not know the caller's entitlement | Private content leakage | Signed delivery, private cache key, origin authorization, short TTL |
| Signed URL is too broad or long-lived | URL permits arbitrary key/method or persists too long | URL sharing or leakage creates unauthorized access | Scope to one object/version/method and keep expiry short |
| Cache key includes irrelevant query parameters | Unique URLs fragment cache | Low hit ratio and higher origin traffic | Define a canonical cache key; include only security/content dimensions |
| Cache ignores security-relevant headers | CDN caches response across different identities | Cross-user data exposure | Separate public/private policies; explicitly configure cache key and headers |
| Revocation is slower than expected | Edge TTL and purge propagation are independent | Deleted/restricted content remains retrievable | Short TTL, versioned URLs, authorization gating, purge and origin denial |
| Range requests use stale object metadata | Range offset applies to a changed representation | Corrupt file or invalid chunk assembly | If-Range validator and immutable object version |
| Multiple ranges are not constrained | Client can request many overlapping ranges | CPU/response amplification and cache inefficiency | Bound range count/size or ignore unsupported multiple ranges |
| Slow clients consume proxy resources | Application holds a stream open | Connection exhaustion and noisy-neighbor impact | Direct delivery, bounded buffering, timeouts, fairness limits |
| Origin stampede on popular miss | Many simultaneous misses fetch same large object | Origin saturation and elevated latency | Origin shield, request collapsing, prewarm, stale serving where safe |
| Metadata lookup is hot | Every request repeats DB and ACL checks | DB overload delays URL issue | Cache safe metadata, batch policy evaluation, indexes, short-lived grants |
| CDN bypass is possible | Origin bucket or URL is public | Attackers avoid edge controls and increase egress | Private origin, OAC/service identity, network restrictions |
| Download analytics block response | Synchronous write in hot path | Added latency and failed downloads during analytics outage | Durable asynchronous events; separate audit-critical and aggregate signals |
| Large archive generation is synchronous | One request waits for expensive work | Timeouts and memory/disk exhaustion | Prebuild bounded archives asynchronously, then deliver immutable artifact |
| Cache purge is treated as authorization | Purge propagation may be delayed or incomplete | Security revocation depends on control-plane timing | Reauthorize private requests or use short-lived scoped delivery capability |

---

## 06 — Architecture Evolution

These are proposed learning stages, not a historical account of any named provider's architecture.

### V1 — Single application server streams content

- Client downloads through one application process.
- **Pressure:** large byte volume and long connections dominate application capacity.
- **Change:** keep authorization and metadata in the application, move durable bytes to object storage.
- **Trade-off:** object storage endpoint and credentials must be protected; direct download may expose a signed URL.

### V2 — Object storage origin with direct authorized delivery

- Download service checks identity and access, then returns a short-lived URL scoped to one object/version.
- Client transfers bytes directly from storage; app servers no longer proxy large bodies.
- **Benefit:** data-plane throughput scales independently from API instances.
- **Trade-off:** bearer URLs can be shared until expiry, and storage URL behavior is provider-specific.

### V3 — CDN for cacheable content

- Place a CDN in front of a private origin; configure cache key, TTL, range support, and origin authentication.
- Public immutable assets can be cached longer; private content requires signed delivery or authorization-aware policy.
- **Benefit:** lower geographic latency and reduced origin bandwidth.
- **Trade-off:** cache invalidation, stale copies, query-string fragmentation, and private cache isolation need explicit policy.

### V4 — Range requests, validators, and resume

- Serve `Accept-Ranges: bytes`, correct `206 Partial Content`/`416 Range Not Satisfiable`, ETag/Last-Modified, and `If-Range`.
- Clients persist local file identity, byte offset, and validator; resume only if the object version matches.
- **Benefit:** avoids retransferring already downloaded bytes after a disconnect.
- **Trade-off:** representation encoding and object mutation must preserve byte-offset semantics; validators and CDN behavior must be tested end to end.

### V5 — Replicated control plane, policy-aware delivery, and observability

- Replicate API instances across zones behind a load balancer; use database replicas/caches for metadata reads and robust policy evaluation.
- Record authorization, URL issuance, bytes, cache result, and completion signals; enforce tenant quotas and abuse limits.
- **Benefit:** improves availability and provides operational control without putting the entire byte path through the app.
- **Trade-off:** policy cache consistency, audit volume, URL revocation, and regional failover need defined guarantees.

### V6 — Multi-region origin strategy and lifecycle governance

- Use regional object replicas or origin failover according to RPO/RTO, residency, cost, and object-version semantics.
- Restrict origins to CDN/service identity; use versioned keys, lifecycle, purge workflows, and recovery tests.
- **Benefit:** better regional resilience and predictable recovery.
- **Trade-off:** replication lag, failover correctness, cache consistency, and duplicate egress/capacity cost.

### Proposed evolved architecture

1. Client uses DNS and TLS to reach an API endpoint or CDN.
2. Gateway/load balancer routes metadata and download-intent requests to stateless service instances.
3. Identity and access control are evaluated before returning metadata or a delivery capability.
4. Download service resolves an immutable object version from the metadata DB and issues a short-lived URL/cookie or authenticated redirect.
5. Client requests the object from CDN; edge cache applies the content-specific policy.
6. On a miss, CDN retrieves from an origin shield or private object-storage origin using authenticated origin access.
7. CDN/object storage streams bytes and supports ranges; client records offset and validator for resume.
8. Access/audit and aggregate delivery telemetry are emitted without logging secrets or body content.
9. Operations revoke grants, update metadata, purge or expire cache, and reconcile object/version state.

### Storage and cache identity

- Use generated immutable object keys, such as `tenant/{opaque-id}/file/{file-id}/version/{version-id}`; do not make a client filename the object path.
- Store a stable file ID separately from object key, version ID, checksum, size, MIME type, owner, ACL/policy reference, status, and timestamps.
- A file update creates a new object version/key. A mutable alias resolves to a version through metadata; it is not the byte cache identity.
- Derive a strong ETag from the stored representation or a trusted content digest when possible. Do not treat multipart provider ETags as a universal whole-object checksum.
- Cache key must include the immutable object version and all dimensions that change response bytes or authorization. Avoid identity-independent shared caching for private data.
- For compressed/transformed variants, the validator and range offsets must apply to the selected representation. Disable on-the-fly transformation for range responses unless the CDN explicitly guarantees consistent range semantics.

### HTTP Range and resume semantics

- A byte range request uses `Range: bytes=start-end`; `bytes=1000-` requests from offset 1000 to the representation end; `bytes=-500` requests the final 500 bytes.
- If the range is satisfiable and supported, return `206 Partial Content`, `Content-Range: bytes start-end/total`, and the selected representation's `Content-Length`.
- A request with no Range or an ignored range returns the full representation with `200 OK`.
- If no requested range is satisfiable, return `416 Range Not Satisfiable` and normally include `Content-Range: bytes */total`.
- `Accept-Ranges: bytes` advertises support but is not a guarantee that every intermediary/client honors ranges.
- `If-Range` contains a validator. If it still matches, the server may return the requested partial representation; if it does not match, return the full current representation (`200`) rather than bytes that cannot safely be combined.
- ETag identifies a representation; Last-Modified is a coarser time validator. Prefer a strong ETag for byte-level resume where available.
- A safe client stores the file ID/version, local size (resume offset), validator, and expected total length. It appends a `206` body only when the returned `Content-Range` begins at that offset and the validator still matches.
- If an ETag/version changes, the client discards or quarantines the partial file and restarts with a full `200` response. Never concatenate bytes from different versions.
- Multi-range requests can return `multipart/byteranges`; clients and caches must parse boundaries correctly. A simpler service may reject or ignore multiple ranges and document that policy.

### API design

All control-plane endpoints require authenticated TLS and enforce tenant-level authorization. The delivery endpoint is either a signed CDN/object URL or a protected streaming URL; it never exposes an internal storage credential. Signed URLs are bearer capabilities and must be redacted from logs, traces, referrers, and analytics.

#### Retrieve file metadata

`GET /v1/files/{file_id}`

Response (`200 OK`):

```json
{
  "file_id": "fil_01J...",
  "name": "manual.pdf",
  "size_bytes": 52428800,
  "mime_type": "application/pdf",
  "current_version": "ver_17",
  "etag": "\"sha256-opaque-validator\"",
  "last_modified": "Sat, 14 Nov 2026 10:15:00 GMT",
  "status": "AVAILABLE"
}
```

Return only metadata the caller is allowed to see. Do not include an internal object key, signing secret, or permanent storage URL.

#### Check access (internal service-to-service API)

`POST /internal/v1/access:check`

Request:

```json
{
  "subject_id": "usr_01J...",
  "tenant_id": "ten_01J...",
  "resource": {
    "file_id": "fil_01J...",
    "version_id": "ver_17"
  },
  "action": "read",
  "purpose": "user-download"
}
```

Response (`200 OK`):

```json
{
  "decision": "ALLOW",
  "policy_version": "pol_42",
  "decision_id": "dec_01J...",
  "valid_until": "2026-11-14T20:05:00Z"
}
```

This endpoint is private to trusted services and is not an unauthenticated permission oracle. A policy decision cache must respect the decision's scope and expiry; sensitive revocations may require immediate invalidation or per-request authorization.

#### Initiate an authorized download

`POST /v1/files/{file_id}/downloads`

Request:

```json
{
  "version_id": "ver_17",
  "purpose": "user-download",
  "delivery": "cdn"
}
```

Response (`201 Created`):

```json
{
  "download_id": "dwn_01J...",
  "file_id": "fil_01J...",
  "version_id": "ver_17",
  "status": "ISSUED",
  "delivery_url": "https://cdn.example/objects/ver_17?signature=REDACTED",
  "expires_at": "2026-11-14T20:05:00Z",
  "size_bytes": 52428800,
  "etag": "\"sha256-opaque-validator\"",
  "last_modified": "Sat, 14 Nov 2026 10:15:00 GMT",
  "accept_ranges": "bytes"
}
```

Use an `Idempotency-Key` when clients may retry creation after a timeout; exact replay returns the same logical download grant/result, while a conflicting payload returns `409 Conflict`. The URL is short-lived, read-only, scoped to this file version, and is never persisted as durable metadata.

#### Renew an expired or expiring delivery URL

`POST /v1/downloads/{download_id}/url:refresh`

Response (`200 OK`):

```json
{
  "download_id": "dwn_01J...",
  "delivery_url": "https://cdn.example/objects/ver_17?signature=REDACTED",
  "expires_at": "2026-11-14T20:10:00Z",
  "version_id": "ver_17",
  "etag": "\"sha256-opaque-validator\""
}
```

The service re-authenticates and re-authorizes before issuing a replacement. A client preserves partial bytes only when the new grant resolves to the same immutable version and validator.

#### Download a complete file or a byte range

The client sends `GET` to the delivery URL (or, for small/strictly controlled responses, `GET /v1/files/{file_id}/content` through an authenticated streaming endpoint). A full response has no `Range` header:

```http
GET /objects/ver_17 HTTP/1.1
Host: cdn.example
```

Response (`200 OK`):

```http
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Length: 52428800
Accept-Ranges: bytes
ETag: "sha256-opaque-validator"
Last-Modified: Sat, 14 Nov 2026 10:15:00 GMT
Content-Disposition: attachment; filename="manual.pdf"
X-Content-Type-Options: nosniff
```

For resume, the client requests the remaining bytes and supplies the saved validator:

```http
GET /objects/ver_17 HTTP/1.1
Host: cdn.example
Range: bytes=1048576-
If-Range: "sha256-opaque-validator"
```

Response (`206 Partial Content`):

```http
HTTP/1.1 206 Partial Content
Content-Type: application/pdf
Content-Length: 51380224
Content-Range: bytes 1048576-52428799/52428800
Accept-Ranges: bytes
ETag: "sha256-opaque-validator"
```

If the range is unsatisfiable, return `416 Range Not Satisfiable` and `Content-Range: bytes */52428800`. If `If-Range` no longer matches, return the full current representation (`200 OK`), not a partial body that a client could append to stale bytes. A `304 Not Modified` may be returned for a valid conditional GET when no body is needed. A server may ignore a Range request and return `200`; clients must verify the status and headers before appending.

#### Retrieve download status

`GET /v1/downloads/{download_id}`

Response (`200 OK`):

```json
{
  "download_id": "dwn_01J...",
  "file_id": "fil_01J...",
  "version_id": "ver_17",
  "status": "RESPONSE_COMPLETE",
  "bytes_sent_observed": 52428800,
  "last_observed_at": "2026-11-14T20:01:14Z"
}
```

Possible states include `ISSUED`, `IN_PROGRESS`, `RESPONSE_COMPLETE`, `FAILED`, `EXPIRED`, and `REVOKED`. `RESPONSE_COMPLETE` means the delivery service observed the response finish; it cannot prove the client persisted bytes to disk. A client may separately report completion, but that report is advisory.

#### Retrieve available versions

`GET /v1/files/{file_id}/versions`

Response (`200 OK`):

```json
{
  "file_id": "fil_01J...",
  "versions": [
    {
      "version_id": "ver_17",
      "size_bytes": 52428800,
      "etag": "\"sha256-opaque-validator\"",
      "created_at": "2026-11-14T10:15:00Z",
      "status": "AVAILABLE"
    }
  ]
}
```

Only expose versions allowed by the caller's policy and retention rules. A version ID is an identity, not an authorization token.

#### Create a multi-file archive (optional)

`POST /v1/download-bundles`

Request:

```json
{
  "file_ids": ["fil_01J...", "fil_02J..."],
  "archive_format": "zip"
}
```

Response (`202 Accepted`):

```json
{
  "bundle_id": "bun_01J...",
  "status": "QUEUED",
  "status_url": "/v1/download-bundles/bun_01J..."
}
```

The service rechecks permission for every item, bounds item count and total bytes, builds the archive asynchronously, and issues a short-lived URL only after the immutable archive is ready. `GET /v1/download-bundles/{bundle_id}` retrieves job status.

#### API status codes

| Status | Meaning |
|---|---|
| `200 OK` | Metadata/status returned, URL refreshed, or full content streamed |
| `201 Created` | Authorized download session created |
| `202 Accepted` | Optional archive job queued |
| `206 Partial Content` | Requested byte range returned |
| `301` / `302` / `307` | Redirect to a delivery endpoint; preserve method only where the chosen status and client behavior require it |
| `304 Not Modified` | Conditional request is current; no body |
| `400 Bad Request` | Malformed request or invalid Range syntax where the service elects to reject it |
| `401 Unauthorized` | Missing or invalid authentication |
| `403 Forbidden` | Caller is authenticated but policy denies access or grant is invalid |
| `404 Not Found` | File/version absent or intentionally hidden from an unauthorized caller |
| `409 Conflict` | Version/state conflict or conflicting idempotency replay |
| `416 Range Not Satisfiable` | No requested byte range can be served |
| `429 Too Many Requests` | Rate, concurrency, quota, or abuse limit exceeded |
| `500 Internal Server Error` | Unexpected service failure |
| `502 Bad Gateway` / `503 Service Unavailable` / `504 Gateway Timeout` | Temporary CDN/origin/dependency failure; provide bounded retry guidance when safe |

### Metadata model

| Field | Purpose | Notes |
|---|---|---|
| `file_id` | Stable public identifier | Opaque, non-sequential ID |
| `tenant_id` / `owner_id` | Ownership and tenant scope | Indexed with file ID and policy queries |
| `original_filename` | User-facing display name | Sanitized for headers; never used as storage path |
| `object_key` | Internal object-storage location | Secret/internal; not directly returned unless intentionally exposed |
| `version_id` | Immutable content revision | Key for stable URL, cache, and resume identity |
| `size_bytes` | Expected representation length | Must match stored object metadata |
| `mime_type` | Response media type | Validated/normalized; use safe Content-Disposition policy |
| `checksum_algorithm` / `checksum` | Integrity metadata | Explicit algorithm and provider semantics |
| `etag` | HTTP representation validator | Preserve strong/weak semantics; not automatically a content hash |
| `last_modified_at` | HTTP conditional validator and audit time | HTTP date has second granularity in common use |
| `status` | Availability/lifecycle | Only eligible states can be delivered |
| `access_policy_id` / ACL | Authorization reference | Avoid copying large ACL into every object row |
| `cache_class` | Public/private/no-store classification | Drives TTL and cache-key policy |
| `created_at` / `updated_at` | Lifecycle and audit timestamps | UTC |
| `retention_until` / `deleted_at` | Lifecycle and deletion semantics | Account for legal hold and version retention |

Index common reads by `(tenant_id, file_id)` and relevant owner/listing dimensions. Use unique constraints for `(file_id, version_id)`. Keep access policy evaluation separate from object lookup; a known file ID is not proof of permission. Store signed URL tokens only at delivery edges, not in the metadata record.

### Practical storage and CDN provider notes

- **Amazon S3 + CloudFront:** The download service can issue a short-lived S3 pre-signed `GetObject` URL or use CloudFront signed URLs/cookies. For a private S3 origin, configure CloudFront Origin Access Control and deny public bucket access. Verify Range, cache-key, response-header, expiry, and invalidation behavior in the selected distribution configuration.
- **Google Cloud Storage + Cloud CDN:** Use a signed URL or authenticated delivery policy for private objects and a backend bucket/origin configuration for edge delivery. Pin object generation/version in the metadata and verify how the chosen URL and cache key preserve that identity.
- **Azure Blob Storage + Azure Front Door:** Use a narrowly scoped, expiring SAS or a trusted authorization layer and keep the storage account private where supported. Verify Range header behavior, origin access restrictions, cache policy, and purge semantics for the configured endpoint.
- Provider signatures and policy objects are not interchangeable. Keep provider-specific URL creation, version resolution, and origin configuration behind an adapter; never assume one provider's URL expiry, ETag, range, or revocation semantics applies to another.

---

## 07 — Components

| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | Distributes control-plane API traffic across healthy gateway/download-service instances | Yes |
| TBD | Client Application | Requests access, follows redirects, downloads/validates ranges, tracks and resumes progress | Yes |
| TBD | DNS | Resolves API and CDN names to service/provider endpoints | Yes |
| TBD | API Gateway | TLS, request IDs, control-plane routing, rate and request-size limits | Yes |
| TBD | Authentication Service | Establishes user or service identity | Yes |
| TBD | Access Control Service | Evaluates ownership, role, ACL, tenant, sharing, and policy | Yes |
| TBD | File Download Service | Resolves metadata/version, checks lifecycle, authorizes, issues delivery capability, and audits | Yes |
| TBD | Metadata Database | Stores file, owner, version, object location, validators, policy references, and status | Yes |
| TBD | Metadata Cache | Caches safe version facts/policy decisions with bounded TTL and invalidation | Optional |
| TBD | Pre-signed URL Generator | Creates narrowly scoped, short-lived read capabilities | Common |
| TBD | CDN / Edge Network | Caches eligible content, supports edge delivery/ranges, and connects to protected origin | Recommended at scale |
| TBD | Origin Shield | Collapses/coalesces regional cache misses and protects object origin | Optional |
| TBD | Object Storage | Durable source of immutable file bytes and versions | Yes |
| TBD | Load Balancer / Origin Router | Routes control and origin traffic to healthy services/origins | Yes, at relevant tier |
| TBD | Monitoring / Audit Pipeline | Tracks access decisions, download outcomes, cache and origin health | Yes |
| TBD | Cache Purge / Lifecycle Worker | Applies revocation, expiry, version retention, and purge workflows | Policy-dependent |

**Component ID policy:** CMP-001 is the only confirmed reusable component relevant to this case study in the current component index. Other IDs remain TBD until component specifications are created and added to the master component index. The metadata database contains facts and policy references; object storage contains the file bytes.

### Component interactions

- Client → DNS → API Gateway/load balancer → Authentication and Access Control → File Download Service.
- Download Service → Metadata DB/cache: resolve file, version, status, size, validators, and storage location.
- Download Service → URL Generator: issue a narrow, short-lived URL/cookie or authenticated redirect.
- Client → CDN: request full object or range; private delivery policy validates the grant.
- CDN → Origin Shield → private object storage: retrieve a cache miss for the exact immutable version.
- Object storage/CDN → Client: stream response headers and bytes; the client may resume with Range/If-Range.
- Download Service/API → Audit pipeline: emit decision and URL issuance events; CDN/provider telemetry supplies data-plane byte/cache events.
- Owner/admin workflow → metadata state, grant revocation, cache purge, and lifecycle workers.
- Monitoring system → operators: alert on elevated 4xx/5xx, cache misses, origin latency, egress, access denials, and stale content.

---

## 08 — Data Flow

### Flow A: Metadata and authorized download intent

1. Client resolves the service name and sends an authenticated metadata or download-intent request over TLS.
2. Gateway applies request-rate and concurrency limits, assigns a request ID, and routes to a healthy instance.
3. Authentication validates the caller; access service checks tenant, file, version, purpose, and current policy.
4. Download service verifies that the file/version is in a deliverable state and not deleted, quarantined, or under a blocking hold.
5. Service reads the minimal metadata and resolves the immutable object version.
6. Service emits an access-decision audit event and returns metadata or a short-lived scoped URL/cookie.
7. Client receives only safe headers and public identifiers; internal storage keys and credentials remain hidden.

### Flow B: CDN cache HIT

1. Client requests the authorized CDN URL, optionally including `Range`, `If-Range`, and conditional headers.
2. CDN validates the signed request/cookie or applies the configured public-content policy.
3. Edge constructs a canonical cache key including immutable version and relevant representation dimensions.
4. Eligible unexpired object is found in edge cache.
5. CDN evaluates conditional/range request against the cached representation and returns `200`, `206`, or `304` as appropriate.
6. Client streams bytes, checks `Content-Range` and ETag, and records progress.
7. CDN emits cache-hit, bytes, status, and timing telemetry; no origin body read is required.

### Flow C: CDN cache MISS and origin fetch

1. Client request passes authorization at the edge or through a signed, narrowly scoped capability.
2. Edge cache misses or revalidation is required.
3. CDN connects to an origin shield or nearest authorized object-storage origin using service identity.
4. Origin resolves the immutable key and returns the selected representation, validators, and requested range.
5. Shield/edge cache stores the response only when cache policy and authorization classification permit it.
6. CDN streams the body to the client while applying bounded buffering and backpressure.
7. Cache/origin telemetry records miss, revalidation, bytes, latency, and status.

### Flow D: Interrupted and resumed download

1. Client saves the partial file, exact byte length, file/version identity, and ETag (or other selected validator).
2. After interruption, client verifies that the local file belongs to the same download and requests `Range: bytes={local_size}-`.
3. Client sends `If-Range` with the stored validator.
4. If the validator matches, service/CDN returns `206` and a `Content-Range` beginning exactly at the local size.
5. Client appends bytes only if version, range start, total length, and response validator are consistent.
6. If validator differs, server returns the full current representation (`200`) or the client restarts explicitly; the old partial bytes are not appended.
7. Client verifies final size and, if available, the trusted checksum before marking the local download complete.

### Flow E: Signed URL expiry, revocation, and deletion

1. A URL expires or the owner revokes access; future authorization/URL issuance requests are denied.
2. For private content, edge/origin checks the grant for each request or uses a narrowly bounded signed-cookie policy.
3. Service denies new grants; operational workflow disables origin access if necessary and purges cached objects where policy requires.
4. Existing in-flight responses may continue unless the delivery provider supports termination; this behavior is documented as a limitation.
5. Metadata marks the version unavailable/deleted and lifecycle cleanup removes origin copies after retention/hold rules.
6. Audit records capture the revocation and purge request; purge completion is verified rather than assumed.

### Error and edge-case paths

- Missing Range returns a full `200`; satisfiable range returns `206`; unsatisfiable range returns `416` with total length when known.
- If `If-Range` does not match, do not append a partial response to old bytes; return a full representation or require a restart.
- A `HEAD` request returns headers for the selected representation without a body, subject to the same authorization and cache policy.
- If object storage is unavailable, return a retryable `503`/`504` as appropriate; do not return a truncated success response.
- A URL expiring mid-download may terminate the current request or affect only the next range request depending on provider behavior; client requests a fresh authorized grant.
- If the client disconnects, cancel upstream work where supported and release buffers/connections promptly.
- If an archive is generated asynchronously, expose job status and deliver only the completed immutable archive artifact.

---

## 09 — Failure Scenarios

| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| CDN edge failure | Provider health telemetry, connection errors, regional latency | Some clients cannot reach an edge | DNS/Anycast provider reroutes; retry with backoff; fail over to another edge/region |
| CDN provider/region outage | Synthetic probes and broad error/latency increase | Cache delivery degraded or unavailable | Secondary CDN/origin fallback if tested; maintain origin capacity policy |
| Object-storage origin outage | Origin 5xx/timeout, health probes | Cache misses and revalidations fail | Serve still-valid stale public content only if policy permits; retry/fail over replicated origin |
| Origin shield failure | Shield health and elevated origin request count | More requests reach storage | Route to another shield/region; request collapsing limits and origin concurrency controls |
| Metadata database outage | DB health, connection pool errors | New grants and metadata reads may fail | Use read replicas/cache only for policy-safe bounded staleness; fail closed for access changes |
| Access-control service outage | Auth/policy timeout or errors | Authorization cannot be confirmed | Fail closed for protected content; avoid stale allow decisions beyond explicit policy TTL |
| Network interruption | Client timeout or partial body | Download is incomplete | Resume from local size with Range + If-Range after reauthorization |
| Signed URL expires | CDN/storage returns authorization failure | New request/range denied | Client requests a fresh grant after authorization; keep partial bytes only if validator matches |
| Invalid/unsatisfiable range | `416`, malformed header, range exceeds size | No requested bytes returned | Include `Content-Range: bytes */length`; client restarts or corrects offset |
| File version changes during resume | If-Range mismatch or version lookup differs | Existing partial bytes cannot be combined | Return full current object or conflict; client restarts and verifies final content |
| Corrupted object | Checksum/integrity audit, storage read error | Incorrect or incomplete bytes possible | Fail request, mark version suspect, restore from replica/backup, invalidate affected cache |
| Slow client | Low send rate, long-lived connection metrics | Buffers and concurrency remain occupied | Bounded buffers, idle timeout, per-tenant concurrency, direct CDN delivery |
| Traffic spike or hot object | RPS/egress/origin saturation | Elevated latency and origin errors | CDN cache, origin shield, request collapsing, prewarm, admission controls |
| Cache key misconfiguration | Unexpected hit ratio, cross-user access tests | Leakage or cache fragmentation | Disable shared caching, correct key, purge affected entries, audit access |
| Stale private cache after revocation | Revocation test and purge telemetry | Content may remain accessible | Reauthorize on request, short signed TTL, versioned key, purge and origin denial |
| DNS failure or stale endpoint | Resolver probes and client errors | API/CDN endpoint unreachable | Redundant DNS/provider configuration; cached TTL-aware failover; status communication |
| Regional outage | Region probes and origin/API health | Regional delivery or metadata fails | Route to healthy region; use replicated immutable object and consistent policy state |
| Download analytics outage | Queue lag, exporter errors | Metrics delayed or missing | Buffer asynchronously; preserve required audit separately; do not block body delivery on aggregates |
| Range amplification abuse | Abnormal range counts, repeated tiny ranges | Excessive origin requests/CPU | Bound range count, request rate and concurrency; coalesce or reject pathological requests |
| Partial response truncation | Client bytes shorter than Content-Length; CDN logs | Client has unusable partial file | Close connection, return error where possible, retry same range with validator |
| Load balancer/API instance failure | Health checks and 5xx | URL issuance/metadata requests fail | Remove unhealthy instance; retry idempotent control request across remaining instances |

Retries must be bounded and jittered. Never treat a completed TCP response as proof that the client persisted the bytes. A successful authorization decision does not guarantee delivery availability, and a cache purge is not a substitute for authorization.

---

## 10 — Scaling

### API and metadata scaling

- Run stateless gateway/download-service instances across zones behind CMP-001; scale from request rate, CPU, connection utilization, and p95 latency.
- Keep download APIs control-plane focused. Avoid serving large response bodies through API instances in the default architecture.
- Index metadata by `(tenant_id, file_id)` and `(file_id, version_id)`; use replicas for safe read-heavy access.
- Cache immutable metadata by version. Policy/ACL decisions need shorter TTLs and explicit invalidation; stale allows are a security decision, not a generic cache optimization.
- Use connection pools and bounded query timeouts; do not hold DB connections while a client downloads bytes.
- Apply per-user and per-tenant limits to URL issuance, concurrent downloads, parallel ranges, and bytes where the business policy requires them.

### CDN and data-plane scaling

- Distribute cacheable objects across edge PoPs; scale delivery capacity independently from metadata APIs.
- Use canonical cache keys with immutable version IDs. Normalize query parameters only when they do not carry authorization or representation semantics.
- Set distinct TTL and cache-control policy for public immutable files, private shared files, and highly sensitive no-store content.
- Use an origin shield or request collapsing to reduce simultaneous cold fetches for popular objects.
- Support byte ranges at edge and origin. Test multi-range behavior, compression, redirects, signed headers, and cache fill semantics with the selected CDN.
- Prefer immutable content-addressed/versioned URLs for long-lived public assets; use short-lived capability URLs for private content.
- Prewarm only when expected demand justifies it; avoid unbounded cache-busting or per-user URL variants that destroy hit ratio.
- Enforce origin bandwidth/concurrency budgets; CDN cache miss bursts must not overload storage.

### Storage, regions, and high availability

- Use object storage as the durable source of file bytes and keep multiple replicas/versions according to recovery and retention policy.
- Choose storage region by residency, user proximity, cost, and restore objective. Geo-replication does not guarantee zero replication lag.
- Keep metadata backup, object replication, CDN configuration, and signing-key recovery as separate runbook items.
- Use versioned object keys so failover does not mix bytes from different revisions. Replicate metadata mapping before routing a request to a replica.
- A secondary CDN/provider needs tested URL signing, cache policy, origin credentials, purge behavior, range semantics, and telemetry integration.
- Fail over by health and policy, not solely by DNS. DNS TTL and client resolvers can delay movement.
- Test origin restore, metadata recovery, key rotation, CDN purge, and deletion propagation under failure.

Large download populations should be served primarily by CDN/object storage and client networks, not by application worker memory. "Millions of concurrent requests" needs separate interpretation as request rate, concurrent TCP/HTTP streams, and active transferred bytes; each drives different limits.

---

## 11 — Trade-offs

### Delivery strategy

| Strategy | Strength | Limitation | Appropriate use |
|---|---|---|---|
| Application-server streaming | Central policy and response control | App bandwidth/connection saturation; expensive large-body path | Small files, transformations, or strict inline policy |
| Direct object-storage URL | Simple, scalable data path | Provider-specific URL, origin exposure, bearer-token leakage risk | Authorized downloads with short-lived scoped URLs |
| CDN with public cache | Low latency and high origin offload | Revocation and cache purge must be managed | Public immutable content |
| CDN with signed URL/cookie | Edge-scale private delivery | Grant lifetime, cache-key policy, and sharing risk | Private content with bounded access windows |
| Authenticated proxy at edge | Per-request authorization and policy flexibility | More edge compute/latency and dependency coupling | Fine-grained private content or immediate policy enforcement |
| Pre-signed URL redirect | Removes bytes from API tier | Anyone holding URL can use it until expiry unless further bound | Private files with short grant lifetime |

### Caching and revocation

| Choice | Benefit | Cost / risk |
|---|---|---|
| Long TTL + immutable version key | High hit ratio and fewer origin reads | Revocation relies on changing version/access grant or purging |
| Short TTL | Faster policy changes | More revalidation/origin traffic and lower hit ratio |
| `no-store` for sensitive content | Prevents compliant caches from storing response | No cache benefit; intermediaries/client behavior still requires security review |
| Authorization-aware cache key | Can share edge infrastructure safely | Identity dimension can fragment cache and leak if incorrectly configured |
| Signed URL | Simple capability-based authorization | Bearer credential can be shared or logged; revocation may be delayed |
| Per-request auth at edge | Current policy evaluated for each request | Adds latency, cost, and identity-service dependency |

### Range and resume behavior

- Range requests reduce repeat transfer after interruption but add request/validator complexity.
- Parallel ranges can improve throughput on some paths but increase connections, request count, and origin work; cap concurrency.
- Multi-range responses can be efficient for some clients but complicate parsing and caching. Single-range resume is simpler.
- Strong ETags and immutable version keys make resume safer; Last-Modified is less precise.
- Serving transformed/compressed representations may invalidate byte offsets. Either pin representation encoding or disable transformations on range paths.
- A client-provided checksum is useful for accidental corruption detection but should not be treated as trusted authorization or content identity unless verified against trusted metadata.

### Required decision record format

For each major decision, record:

1. Context and assumptions.
2. Options considered.
3. Selected option and reason.
4. Security and operational consequences.
5. Revisit trigger, such as measured hit ratio, revocation SLO, or origin saturation.

---

## 12 — Cost

### Cost categories

| Cost area | Main driver | Optimization / caveat |
|---|---|---|
| CDN egress | Delivered bytes, geography, provider plan | Cacheable immutable content reduces origin egress; CDN egress still has cost |
| Origin egress | Cache misses, revalidation, uncached/private bytes | Origin shield, cache policy, and hot-object request collapsing |
| Object storage | Stored bytes, versions, replication, lifecycle | Retention and version policy; avoid deleting required recoverable data |
| Storage requests | GET, range GET, metadata, replication, inventory | Range parallelism and retries can multiply operations |
| API compute | Metadata, authorization, URL issuance, audit | Avoid application-proxied file bytes |
| Database/cache | Metadata reads, policy lookups, audit writes | Indexes, bounded caching, read replicas, data retention |
| Logging/observability | CDN logs, request events, traces, retention | Sample high-volume aggregates while retaining required security audit |
| Multi-CDN / multi-region | Duplicated configuration, egress, replicas, failover capacity | Pay for resilience; test actual failover rather than assuming it works |
| Support and operations | On-call, incident response, abuse and compliance review | Operational maturity is part of total cost |

### Estimation worksheet

Let:

- `D` = downloads/day
- `S` = mean object size in GiB
- `H` = byte cache hit ratio (0–1)
- `P_cdn` = blended CDN egress price per GiB
- `P_origin` = origin egress price per GiB
- `P_store` = monthly object-storage price per GiB
- `R` = average retained GiB

Then approximate monthly egress cost as:

`monthly_egress ≈ 30 × D × S × P_cdn + 30 × D × S × (1 − H) × P_origin`

Approximate monthly storage as:

`monthly_storage ≈ R × P_store`

For the illustrative workload in Section 03, `D = 1,000,000`, `S = 25/1024 GiB`, and `H = 0.85`; compute the exact provider bill only after selecting region, request mix, cache policy, transfer tiers, replication, and current published pricing. Do not assume a CDN hit is free, that all bytes are cacheable, or that request charges are negligible.

This model excludes taxes, tiered pricing, minimum commitments, invalidation requests, origin shield charges, retries, storage API operations, encryption, logs, backup, and inter-region replication. Use each provider's current pricing calculator before a production decision.

---

## 13 — Security

### Threats and controls

| Threat | Control |
|---|---|
| Unauthorized file read | Authenticate user; authorize file, tenant, version, purpose, and state on every grant |
| Sequential ID enumeration | Opaque identifiers, uniform not-found/forbidden policy where appropriate, rate limiting |
| Signed URL leakage | Short TTL, single object/version/read method, HTTPS, restrictive response headers, no URL logging/referrer leakage |
| URL sharing/abuse | Audience/IP binding only where reliable and privacy-appropriate; quotas, anomaly detection, revocation and short expiration |
| Public bucket/origin bypass | Private origin, least-privilege CDN identity, deny direct public reads, restrict network/service principals |
| Cross-user cache leak | Separate cache policy/key for private data; authorization before serving cache hit; test adversarial identities |
| Stale content after access removal | Short grant TTL, policy-aware edge checks, versioned URLs, cache purge, origin denial |
| Path/object-key injection | Resolve object key from trusted metadata; never concatenate untrusted file path into storage location |
| Response header injection | Sanitize filename; use safe Content-Disposition encoding and fixed media-type policy |
| Cache poisoning | Canonicalize cache keys; validate host/query/headers; do not cache error or authorization responses unintentionally |
| Range amplification | Limit range count and request rate; constrain overlapping/tiny ranges; monitor bytes and origin work |
| Credential or token exposure | Keep signing keys in secret management; rotate keys; redact query tokens and cookies from logs |
| Denial of service | Rate/concurrency/byte limits, CDN DDoS controls, origin budgets, circuit breakers |
| Insecure transport | TLS for client/API/origin paths; modern cipher/protocol configuration and certificate rotation |
| Data at rest exposure | Provider encryption at rest and key management; least privilege and key lifecycle |
| Audit privacy leakage | Log identifiers and outcomes, not contents, credentials, or unnecessary personal data |

### Validation rules

- Authorize every file/version before issuing a capability; do not infer access from possession of a file ID.
- Grant only `GET`/`HEAD` for one object version with bounded expiry and required headers.
- Keep bucket/container private. A CDN origin identity must have read access only to the required prefix/container.
- Separate public immutable assets from private or sensitive files using distinct hostnames/configurations where possible.
- Define cache directives explicitly. For sensitive content, consider `Cache-Control: private, no-store`; verify actual CDN behavior and don't assume response headers alone override provider configuration.
- Set safe `Content-Disposition` (`attachment` where appropriate), sanitized filename, and `X-Content-Type-Options: nosniff`.
- Never put long-lived bearer credentials in application logs, analytics URLs, referrer headers, or error reports.
- Treat URL expiry and cache purge as defense-in-depth; authorization is the primary access boundary.
- Verify access denial, revocation, range resume, alternate CDN, and origin-bypass behavior in automated security tests.

---

## 14 — Observability

### Metrics

| Metric | Why it matters |
|---|---|
| Download start rate and active transfers | Separates request volume from concurrency |
| Successful completion rate | Measures end-to-end usefulness, not just HTTP response starts |
| 2xx/3xx/4xx/5xx counts by route and region | Detects auth, range, origin, and provider failures |
| Time to first byte (TTFB), p50/p95/p99 | Captures control-plane and edge/origin startup latency |
| Bytes delivered and throughput | Capacity and cost signal by region, tenant, and cache status |
| CDN hit ratio by request and by bytes | Reveals cache effectiveness and origin offload |
| Origin request rate, egress, latency, and concurrency | Indicates shield/cache misses and origin saturation |
| Range request rate, `206`/`416` counts, range size | Detects resume behavior, invalid clients, and abuse |
| URL issuance/denial/expiry rate | Authorization and signed-capability health |
| Resume success and restart rate | Measures client recovery and validator correctness |
| Client disconnect/truncated response rate | Identifies path/network or timeout issues |
| Storage response latency and error rate | Origin service health |
| Metadata DB/cache hit, latency, and connection utilization | Control-plane bottleneck |
| Per-tenant concurrent downloads and bytes | Fairness, quota, and abuse detection |
| Cache purge request/completion lag | Revocation and lifecycle SLO |

### Derived indicators and alerts

- **Byte hit ratio** = CDN cache-hit bytes / total bytes delivered.
- **Origin offload** = `1 − origin bytes / delivered bytes`, with consistent byte accounting.
- **Download completion rate** = completed downloads / initiated downloads, segmented by file size and client/network cohort.
- **Resume success rate** = valid range resumes completed / valid resume attempts.
- **Origin amplification** = origin GET/range requests / client download starts; rising values may indicate range fragmentation or stampede.
- Alert on control-plane availability, p95 URL issue latency, regional 5xx, origin saturation, sharp byte-hit decline, private-cache policy failures, elevated `416`, and purge lag.
- Use multi-window alerts and distinguish provider-wide incidents from one tenant/client issue.

### Logs and traces

- Propagate request/trace IDs across gateway, authorization, metadata, URL signing, CDN/origin logs where possible.
- Record file ID/version, tenant, decision, response code, cache status, bytes, range bounds, region, latency, and provider request ID.
- Redact signed URL query strings, cookies, authorization headers, secrets, and file contents.
- Use sampled traces for routine delivery and retain complete access/audit events according to policy.
- Download completion from the server is approximate: a server may observe bytes sent but cannot prove they were persisted by the client's device.

### Example alerts

- p95 TTFB > 2× baseline for 10 minutes in a region.
- Origin egress or concurrent reads exceed 80% of tested safe capacity.
- CDN byte hit ratio drops by 20 percentage points from the same-hour baseline.
- Private file receives a cache hit without a valid grant in synthetic security tests.
- `416` rate or repeated small-range requests exceeds configured threshold.
- Signed URL authorization failures spike after key rotation.
- Purge completion latency exceeds the documented revocation objective.

---

## 15 — Simulation

### Learning objective

Demonstrate how delivery path, authorization mode, file size, cache policy, range/resume behavior, client bandwidth, origin capacity, and regional failure affect latency, throughput, origin load, cost, and completion probability.

### Initial simulation controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| concurrent_downloads | Concurrent downloads | Slider | 5000 | 1 | 1000000 |
| downloads_per_second | Download starts / second | Slider | 250 | 1 | 100000 |
| file_size_mib | File size (MiB) | Slider | 25 | 0.01 | 102400 |
| client_bandwidth_mbps | Mean client bandwidth | Slider | 50 | 0.1 | 10000 |
| client_bandwidth_variance | Client bandwidth variation | Slider | 0.5 | 0 | 2 |
| delivery_mode | Delivery mode | Select | cdn_signed | app_proxy / object_signed / cdn_public / cdn_signed |
| cache_hit_ratio | CDN byte hit ratio | Slider | 0.85 | 0 | 1 |
| range_resume_share | Resumed/range downloads | Slider | 0.15 | 0 | 1 |
| interruption_probability | Interruption probability per transfer | Slider | 0.02 | 0 | 0.5 |
| origin_capacity_mbps | Origin egress capacity | Slider | 20000 | 100 | 1000000 |
| edge_capacity_mbps | Edge delivery capacity | Slider | 100000 | 100 | 10000000 |
| metadata_capacity_rps | Metadata/policy capacity | Slider | 2000 | 10 | 100000 |
| url_ttl_seconds | Signed URL TTL | Slider | 900 | 30 | 86400 |
| object_change_probability | Version change during resume | Slider | 0 | 0 | 0.2 |
| range_parallelism | Concurrent range requests per client | Stepper | 1 | 1 | 16 |
| origin_failure_probability | Origin request failure probability | Slider | 0 | 0 | 0.5 |
| private_content_share | Private content share | Slider | 0.5 | 0 | 1 |
| random_seed | Deterministic simulation seed | Integer | 42 | 0 | 2147483647 |

### Derived values

- Payload bytes = `completed_downloads × mean_file_size`, adjusted for retries/restarts.
- Average payload bandwidth = `delivered_bytes × 8 / duration_seconds`.
- Estimated origin payload = `delivered_bytes × (1 − cache_hit_ratio)` for eligible cached traffic; private/uncacheable traffic is modeled separately.
- Byte hit ratio and request hit ratio are separate outputs.
- Approximate transfer time = `bytes / min(client_bandwidth, available_edge_or_origin_bandwidth)`, excluding protocol and congestion effects.
- Control-plane load = starts + metadata reads + URL renewals + range/retry authorization calls.
- Active transfers estimate = arrival rate × mean transfer duration (Little's Law).
- Resume bytes = remaining bytes only when validator/version matches; mismatch causes restart or full response.
- `206` is produced for a satisfiable range; `416` for unsatisfiable range; stale `If-Range` produces a full representation response.
- Signed URL expiry during a transfer may affect only a subsequent request, depending on provider behavior; model this as configurable rather than universal.
- Origin load and download completion are bounded by simulated capacity; do not infer unlimited CDN/storage throughput.
- Cost output uses user-entered price assumptions and is an estimate, not a provider quote.

### Output metrics

- Active and completed downloads, failures, retries, and resumed transfers.
- Download duration, TTFB, throughput, and bytes delivered.
- Range requests, `206`, `416`, full restarts, and validator mismatches.
- CDN request and byte hit ratios; origin request volume, bytes, and utilization.
- Metadata/API utilization and authorization denials.
- Expired URLs, refresh attempts, and completion after refresh.
- Estimated egress/request cost by delivery mode.
- Per-tenant concurrency and throttled requests.
- Availability and delivery outcomes during edge, origin, DB, and region failures.

### Required scenarios

1. Compare application-proxied, direct object, and CDN delivery for large files.
2. Vary cache hit ratio and observe origin egress and TTFB.
3. Interrupt a download and resume with a matching ETag; verify only remaining bytes transfer.
4. Change object version during resume; verify that old partial bytes are not appended.
5. Send satisfiable, unsatisfiable, suffix, and malformed Range requests.
6. Expire a signed URL between range requests and renew it after authorization.
7. Compare private authorization-aware delivery with public immutable cache behavior.
8. Increase parallel range count and observe request amplification, throughput, and origin load.
9. Simulate a hot-object cache miss and test origin shield/request collapsing.
10. Fail one CDN region and route traffic to a healthy region/provider.
11. Fail metadata or access-control service and confirm protected content fails closed.
12. Revoke a file and measure grant expiry, purge propagation, and origin denial behavior.
13. Reduce origin capacity and observe backpressure, stale policy, and error rates.
14. Increase slow clients and test bounded buffers, idle timeouts, and fairness.

### Simulation model warning

This is a teaching model. Real cache policies, URL signing, range support, bandwidth, retry semantics, purge latency, and pricing depend on provider configuration, client implementation, and network conditions. The simulator must label results as estimates and must not model a security-relevant denial as a successful download.

---

## 16 — Design Challenge

### Challenge: Build a secure, resumable global download service

Design a service for users retrieving documents, media, and software packages from web/mobile clients. It must handle large files, resume interrupted transfers, protect private content, reduce origin load with a CDN, and remain observable and recoverable.

**Requirements**
- Use Section 03 assumptions: 1,000,000 downloads/day, 25 MiB average, 50,000 peak concurrent transfers, and an assumed 85% CDN byte hit ratio for eligible content.
- Separate control-plane authorization/metadata from data-plane bytes.
- Support complete download, HTTP byte ranges, validators, and safe resumption.
- Support public immutable and private/revocable delivery with distinct cache policy.
- Keep origin storage private and bind every delivery URL to one object/version/read permission.
- Define metadata fields, file-version identity, and cache-key behavior.
- Explain CDN HIT/MISS, origin shield, and regional failover.
- Include rate limits, quotas, bandwidth/concurrency controls, audit, and security tests.
- Address object/metadata failure, CDN outage, URL expiration, interruption, invalid ranges, cache staleness, and slow clients.
- Provide a transparent cost worksheet and measurable simulation acceptance checks.

**Constraints for the exercise**
- A client must never append bytes from two different file versions.
- A private CDN hit must require a valid access capability or policy check.
- A protected file must not be made public by an origin URL or cache configuration.
- A resumable request with a matching validator must avoid retransferring completed bytes.
- A stale `If-Range` must not produce a partial response that can corrupt a local file.
- Large object bytes must not be buffered in API memory proportional to file size.
- Inject CDN, origin, metadata DB, access-control, and regional failures.
- No invented SLO, provider limit, or price is to be presented as a verified fact.

**Learner deliverables**
1. Functional and non-functional requirements with workload assumptions.
2. Naive and evolved architecture diagrams.
3. Authorization, cache HIT/MISS, range/resume, and failure flows.
4. Metadata/version model and REST endpoint/HTTP response design.
5. Cache policy and signed URL scope/expiry strategy for public and private files.
6. Origin privacy, CDN shield, regional failover, and lifecycle plan.
7. Security, observability, cost, and recovery approach.
8. A simulation plan with measurable acceptance criteria.

**Evaluation prompts**
- Why should an application API avoid proxying every large response body?
- How can a CDN cache hit be served without weakening authorization?
- Which cache-key fields are necessary for version and representation correctness?
- How do `Range`, `206`, `416`, ETag, and `If-Range` work together?
- What should a client do when the validator changes during a resume?
- Why is a pre-signed URL a bearer capability, and what are its limits?
- How does a private origin prevent CDN bypass?
- Why are cache purge and authorization not interchangeable?
- How do you handle origin stampede on a popular cold object?
- How do you distinguish request hit ratio from byte hit ratio?

---

## 17 — AI Interview

### Interviewer question bank

| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What does a file download service provide? | Authorization, metadata, reliable delivery, versioning, observability |
| Q02 | Beginner | Walk through a download request from client to object storage. | Identity, policy, metadata, signed capability, CDN/origin, bytes |
| Q03 | Beginner | Why use a CDN? | Geographic latency, caching, origin offload, limits and cost |
| Q04 | Intermediate | Compare application proxy, direct storage, and CDN delivery. | Control, scalability, exposure, policy, egress |
| Q05 | Intermediate | What do `Range`, `206`, `416`, and `Content-Range` mean? | Correct byte offsets and unsatisfiable behavior |
| Q06 | Intermediate | How does a client safely resume a partial file? | Persist offset/version, `If-Range`, validate response before append |
| Q07 | Intermediate | What if ETag changes during a resumed download? | Do not append; restart from full representation |
| Q08 | Intermediate | How can private content be cached safely? | Signed access/per-request auth, cache isolation, explicit policy |
| Q09 | Intermediate | Why is a signed URL a security-sensitive bearer token? | Scope, expiry, leakage, revocation, logging controls |
| Q10 | Advanced | How would you prevent direct origin bypass? | Private bucket, CDN service identity, least privilege, network policy |
| Q11 | Advanced | How do cache keys and immutable versions prevent stale or mixed content? | Versioned key, representation dimensions, validator consistency |
| Q12 | Advanced | What if cache purge is delayed after deletion? | Reauthorization, short grant, origin denial, purge verification |
| Q13 | Advanced | How do you handle a hot-object cache miss without overloading origin? | Shield, request collapse, prewarm, concurrency budgets |
| Q14 | Advanced | How do you scale millions of concurrent downloads? | Separate request rate, active connections, bytes, CDN and storage capacity |
| Q15 | Advanced | What are the risks of parallel range requests? | Request amplification, abuse, connection and origin load |

### Follow-up evaluation dimensions

- Separates control-plane decisions from data-plane delivery.
- Correctly explains HTTP range and validator semantics.
- Makes cache policy dependent on confidentiality and revocation needs.
- Uses immutable object versions and avoids mixing bytes across revisions.
- Treats signed URLs as bounded bearer credentials.
- Protects origin and measures CDN byte hit ratio/origin offload.
- Includes client interruption, regional failure, purge, retry, and cost behavior.

---

## 18 — Architecture Score

### Proposed evaluation rubric

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements and workload assumptions | 10 | File distribution, concurrency, regions, cacheability, SLO |
| Architecture and data flow | 15 | Client, API, authorization, metadata, CDN, shield, origin |
| HTTP semantics and resumability | 15 | Range/If-Range/ETag, `200`/`206`/`416`, safe append |
| Security and privacy | 15 | Access control, scoped URL, private origin, cache isolation, revocation |
| Scalability and high availability | 15 | CDN/origin scaling, DB, regions, failover, admission controls |
| Storage and metadata correctness | 10 | Immutable version, object location, validators, lifecycle |
| Cost and trade-off justification | 5 | Egress, storage, requests, cache, multi-region |
| Observability and operations | 10 | TTFB, completion, bytes, hit ratio, origin, audit and alerts |
| **Total** | **100** | |

### Scoring guidance

- 0: Missing or technically incorrect.
- 1: Mentioned without explanation.
- 2: Partially explained.
- 3: Correct and reasonably justified.
- 4: Detailed, measurable, and handles edge cases.

Weighted score = sum of (dimension score / 4 × dimension weight).

The score should be accompanied by evidence and reviewer comments; it should not be treated as an objective measure of architecture quality without human review.
