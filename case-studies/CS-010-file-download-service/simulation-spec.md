# CS-010 — Simulation Specification

## Goal

Demonstrate how delivery path, authorization, cache policy, object size, range/resume behavior, client bandwidth, origin capacity, and regional failures affect download latency, throughput, origin load, completion, and estimated cost.

## Controls

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

## Derived values

These are simplified teaching-model equations. The implementation must document and test them.

- Payload bytes = completed downloads × mean file size, adjusted for retries/restarts.
- Average payload bandwidth = delivered bytes × 8 / duration seconds.
- Estimated origin payload = delivered bytes × `(1 − cache_hit_ratio)` for eligible content; private/uncacheable traffic is modeled separately.
- Byte hit ratio and request hit ratio are distinct outputs.
- Approximate transfer time = bytes / min(client bandwidth, available edge or origin bandwidth), excluding protocol and congestion effects.
- Control-plane load = starts + metadata reads + URL renewals + range/retry authorization calls.
- Active transfers estimate = arrival rate × mean transfer duration (Little's Law).
- Resume transfers only remaining bytes when the object validator/version matches; mismatch causes restart or full response.
- `206` is produced for a satisfiable range; `416` for an unsatisfiable range; stale `If-Range` produces a full representation response.
- Signed URL expiry during a transfer may affect only a subsequent request, depending on provider behavior; model this as configurable rather than universal.
- Origin load and download completion are bounded by simulated capacity; do not infer unlimited CDN/storage throughput.
- Cost output uses user-entered price assumptions and is an estimate, not a provider quote.

## Output metrics

- Active and completed downloads, failures, retries, and resumed transfers.
- Download duration, TTFB, throughput, and bytes delivered.
- Range requests, `206`, `416`, full restarts, and validator mismatches.
- CDN request and byte hit ratios; origin request volume, bytes, and utilization.
- Metadata/API utilization and authorization denials.
- Expired URLs, refresh attempts, and completion after refresh.
- Estimated egress/request cost by delivery mode.
- Per-tenant concurrency and throttled requests.
- Availability and delivery outcomes during edge, origin, DB, and region failures.

## Required scenarios

1. Compare application-proxied, direct object, and CDN delivery for large files.
2. Vary cache hit ratio and observe origin egress and TTFB.
3. Interrupt a download and resume with matching ETag; verify only remaining bytes transfer.
4. Change object version during resume; verify old partial bytes are not appended.
5. Send satisfiable, unsatisfiable, suffix, and malformed Range requests.
6. Expire a signed URL between range requests and renew after authorization.
7. Compare private authorization-aware delivery with public immutable cache behavior.
8. Increase parallel range count and observe request amplification, throughput, and origin load.
9. Simulate a hot-object cache miss and test origin shield/request collapsing.
10. Fail one CDN region and route traffic to a healthy region/provider.
11. Fail metadata or access-control service and confirm protected content fails closed.
12. Revoke a file and measure grant expiry, purge propagation, and origin denial behavior.
13. Reduce origin capacity and observe backpressure, stale policy, and error rates.
14. Increase slow clients and test bounded buffers, idle timeouts, and fairness.

## Acceptance checks

- A matching `If-Range` resume returns a range beginning at the saved local offset.
- A stale validator never results in appending bytes from a different object version.
- Unsatisfiable range returns `416` and the total representation length where known.
- Private cache hits require a valid capability or policy check.
- No origin URL bypasses the intended CDN/service authorization boundary.
- Cache hit and miss have distinguishable origin-byte and latency outputs.
- Control-plane or policy failure does not turn protected content into a successful public response.
- Range concurrency and retries remain within configured bounds.
- Cost and performance outcomes are labeled as estimates, not measured provider behavior.
