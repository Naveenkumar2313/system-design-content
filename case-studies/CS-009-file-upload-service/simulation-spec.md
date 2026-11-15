# CS-009 — Simulation Specification

## Goal

Demonstrate how upload strategy, file size, part size, network interruption, retry policy, storage, metadata, queue, and worker capacity affect completion time, reliability, cost, and time to availability.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| concurrent_clients | Concurrent clients | Slider | 1000 | 1 | 100000 |
| file_size_bytes | File size | Slider | 104857600 | 1024 | 107374182400 |
| client_bandwidth_mbps | Average client bandwidth | Slider | 20 | 0.1 | 1000 |
| part_failure_probability | Network interruption per part | Slider | 0.02 | 0 | 0.5 |
| upload_strategy | Upload strategy | Select | multipart_direct | single / proxy / multipart / resumable |
| part_size_bytes | Part size | Slider | 16777216 | 1048576 | 268435456 |
| parallel_parts | Parallel parts per client | Stepper | 4 | 1 | 32 |
| session_ttl_seconds | URL/session TTL | Slider | 900 | 60 | 86400 |
| retry_attempts | Retry attempts | Stepper | 5 | 0 | 20 |
| storage_capacity_mbps | Object-storage capacity | Slider | 10000 | 100 | 1000000 |
| api_capacity_rps | Metadata API capacity | Slider | 1000 | 10 | 100000 |
| db_write_capacity_rps | Metadata DB write capacity | Slider | 2000 | 10 | 100000 |
| queue_capacity_jobs_per_second | Queue consumer capacity | Slider | 500 | 1 | 50000 |
| worker_seconds_per_file | Worker processing time | Slider | 30 | 0.1 | 3600 |
| scanner_failure_probability | Scanner failure rate | Slider | 0 | 0 | 0.2 |
| tenant_quota_bytes | Tenant storage quota | Slider | 107374182400 | 1048576 | 109951162777600 |
| verify_checksum | Verify file checksum | Toggle | true | — | — |
| enable_cdn_download | Enable CDN downloads | Toggle | false | — | — |
| random_seed | Deterministic simulation seed | Integer | 42 | 0 | 2147483647 |

## Derived values

These are simplified teaching-model equations. The implementation must document and test them.

- Single-stream transfer time = `file_size_bytes × 8 / effective_bandwidth_bits_per_second`.
- Multipart part count = `ceil(file_size_bytes / part_size_bytes)`, subject to selected provider limits.
- Effective parallel transfer rate ≤ `min(client_bandwidth, storage_capacity, parallel_parts × per-part rate)`.
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

## Output metrics

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

## Required scenarios

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

## Acceptance checks

- Completed parts remain committed after a simulated connection interruption.
- Resume transfers only missing parts/offsets and never exceeds configured retry/parallelism limits.
- Repeated initiation with the same key and payload returns the same session; conflicting payload returns a conflict.
- Duplicate completion does not create multiple logical files or processing jobs.
- Provider ETags are not treated as full-file checksums unless the selected provider/algorithm explicitly guarantees it.
- Metadata state and actual object/session state reconcile after partial failures.
- A file is not downloadable before integrity verification and required scanning succeed.
- Queue and worker failures preserve pending state, retry safely, and expose backlog/age.
- Expired multipart uploads are aborted and quota/storage accounting is reconciled.
- Quota reservations remain bounded under concurrent initiation.
- Simulation labels all throughput, latency, and cost outcomes as estimates.
