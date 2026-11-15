# CS-008 — Simulation Specification

## Goal

Demonstrate how content popularity, TTL, cache-key dimensions, edge count, origin capacity, origin shield, purge, and failure policies affect cache hits, viewer latency, bandwidth, and origin load.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| viewer_rps | Viewer requests/sec | Slider | 1000 | 10 | 100000 |
| edge_pops | Edge PoPs | Stepper | 6 | 1 | 100 |
| object_count | Objects in catalog | Slider | 10000 | 10 | 1000000 |
| popularity_alpha | Zipf popularity skew | Slider | 0.9 | 0.0 | 1.5 |
| average_object_bytes | Average object bytes | Slider | 32768 | 1024 | 104857600 |
| edge_cache_bytes | Edge cache capacity | Slider | 10737418240 | 104857600 | 10995116277760 |
| ttl_seconds | Object TTL | Slider | 300 | 0 | 86400 |
| origin_rtt_ms | Origin round-trip time | Slider | 80 | 5 | 500 |
| origin_capacity_rps | Origin capacity | Slider | 2000 | 10 | 100000 |
| origin_shield | Enable parent/shield cache | Toggle | true | — | — |
| request_collapse | Collapse same-key concurrent misses | Toggle | true | — | — |
| cache_policy | Content policy | Select | public_static | public_static / revalidate / bypass |
| stale_mode | Stale serving policy | Select | disabled | disabled / swr / stale_if_error |
| purge_scope | Invalidation scope | Select | one_path | one_path / tag / all |
| origin_failure | Inject origin failure | Select | none | none / timeout / 5xx / unavailable |
| routing_mode | Global routing mode | Select | dns | dns / anycast / multi_cdn |
| failed_provider_percent | Provider capacity unavailable (%) | Slider | 0 | 0 | 90 |
| dns_convergence_seconds | Simulated resolver/client convergence | Slider | 60 | 0 | 3600 |
| random_seed | Deterministic simulation seed | Integer | 42 | 0 | 2147483647 |

## Derived values

These are simplified teaching-model equations. The implementation must document and test them.

- Request hit ratio = fresh eligible cache HIT requests / eligible viewer requests.
- Byte hit ratio = response bytes served from cache / eligible response bytes delivered.
- Origin offload by request = `1 - origin-bound eligible requests / eligible viewer requests`; report byte offload separately.
- Average request RPS = requests/day / 86,400 where daily workload is used.
- Estimated egress bit rate = requests/sec × average response bytes × 8.
- Object popularity is generated from a documented Zipf-like distribution with `popularity_alpha`; deterministic seed ensures repeatability.
- Each PoP receives requests according to the selected routing model; per-PoP caches are independent unless a shared parent tier is enabled.
- TTL controls freshness eligibility. Cache capacity and the declared eviction policy may remove an object before expiration.
- On fresh HIT, modeled origin body-fetch count is zero.
- On MISS, parent HIT avoids origin fetch; request collapsing combines concurrent same-key fills if enabled.
- Revalidation uses a configured validator-hit probability. A 304 still counts as an origin request/round trip but transfers only modeled metadata, not a full body.
- Origin load = fill requests + revalidations + bypassed requests + retries; report request/sec and bytes/sec.
- Viewer latency = client-to-edge path + edge processing + response transfer; MISS adds parent/origin path and origin service. Report HIT, MISS, and revalidation separately.
- An origin queue is bounded. When offered work exceeds capacity, show queueing, rejection, or timeout; never use an unbounded queue.
- Purge removes matched objects after a modeled propagation delay and can cause a cache refill burst.
- Stale-while-revalidate may serve eligible stale content while one background refresh proceeds; stale-if-error applies only to allowed failures and configured windows.
- Multi-CDN failover occurs after health detection and modeled DNS/client convergence. The alternate provider starts with its independent cache state.
- All latency, capacity, and availability results are simulated estimates, not provider measurements.

## Output metrics

- Request and byte cache-hit ratios by PoP and overall.
- HIT, MISS, revalidation, bypass, stale, and error request counts.
- Viewer TTFB estimate for HIT, MISS, revalidation, and failed-origin paths.
- Origin request rate, bandwidth, active fetches, utilization, and rejection rate.
- Shield/parent hit ratio and origin offload.
- Edge storage occupancy, eviction rate, and object residency.
- Purge completion time, refill burst, and freshness outcome.
- Successful, rejected, queued, stale-served, and timed-out responses.
- Per-region/provider request distribution and failover convergence.
- Security denials and private-content cache-isolation checks.

## Required scenarios

1. Compare direct-origin delivery with edge caching for nearby and distant viewers.
2. Compare uniform object popularity with a highly skewed distribution.
3. Increase edge storage and observe eviction, request hit ratio, byte hit ratio, and cost proxy.
4. Compare a short TTL, long TTL, and versioned object URL.
5. Revalidate an unchanged object using ETag and compare bytes transferred with a full fetch.
6. Purge one object versus all objects and observe refill/origin load.
7. Trigger a popular-object stampede with and without request collapsing, shield, and stale-while-revalidate.
8. Fail an origin and compare fresh hits, stale-if-error, and uncached behavior.
9. Add a regional cache/origin shield and measure miss latency versus origin request reduction.
10. Introduce a cache-key omission for a user-specific response and verify a privacy violation is detected.
11. Compare DNS steering and anycast under resolver-location mismatch, regional failure, and convergence delay.
12. Fail one CDN in a multi-CDN setup and observe alternate-provider cold-cache and origin refill behavior.

## Acceptance checks

- Request and byte hit ratios use separate denominators and exclude or disclose bypass/un-cacheable requests.
- A fresh HIT does not generate an origin body fetch.
- A parent HIT does not count as an edge HIT unless the stated metric explicitly combines tiers.
- TTL expiry triggers revalidation/fetch only when the object remains resident; eviction may cause an earlier MISS.
- A 304 reduces modeled bytes but still includes origin request and round-trip latency.
- Request collapsing bounds same-key concurrent origin fills when enabled.
- Origin queueing is bounded and overload produces visible rejection/timeouts.
- Purge is scoped, has a visible propagation delay, and creates a measurable refill wave.
- Stale serving occurs only for configured eligible content and failure windows.
- A user-specific response cannot be served to a different simulated user due to a cache-key collision.
- Multi-CDN failover includes health detection, convergence delay, and independent alternate cache state.
- Simulation labels all latency/capacity outcomes as estimates and not provider benchmarks.
