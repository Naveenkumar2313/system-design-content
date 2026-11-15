# CS-006 — Simulation Specification

## Goal
Demonstrate how a reverse proxy routes traffic, terminates TLS, applies cache and rate-limit policy, and responds to proxy or origin failures.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| incoming_rps | Incoming requests/sec | Slider | 1000 | 10 | 100000 |
| proxy_instances | Reverse-proxy instances | Stepper | 4 | 1 | 20 |
| proxy_capacity_rps | Capacity per proxy | Slider | 5000 | 100 | 20000 |
| origin_count | Origin servers | Stepper | 4 | 1 | 50 |
| origin_capacity_rps | Capacity per origin | Slider | 4000 | 100 | 20000 |
| algorithm | Routing algorithm | Select | round_robin | — | — |
| failed_proxies | Failed proxy instances | Stepper | 0 | 0 | proxy_instances |
| failed_origins | Failed origins | Stepper | 0 | 0 | origin_count |
| service_time_ms | Average origin service time | Slider | 20 | 1 | 1000 |
| tls_termination | Terminate client TLS | Toggle | true | — | — |
| cache_enabled | Enable response cache | Toggle | false | — | — |
| cache_hit_ratio | Eligible response cache hit ratio | Slider | 0 | 0 | 100 |
| rate_limit_rps | Per-client request limit | Slider | 100 | 1 | 100000 |
| burst_capacity | Rate-limit burst capacity | Slider | 200 | 0 | 10000 |
| health_detection_seconds | Health detection delay | Slider | 5 | 0 | 60 |
| upstream_timeout_ms | Upstream timeout | Slider | 1000 | 10 | 60000 |

## Derived values

These are simplified teaching-model equations. The implementation must document and test them.

- Healthy proxies = proxy instances − failed proxies.
- Effective proxy capacity = healthy proxies × per-proxy capacity.
- Healthy origins = origin count − failed origins.
- Effective origin capacity = healthy origins × per-origin capacity.
- Eligible cache hits/sec = incoming eligible RPS × cache hit ratio.
- Origin-bound RPS = incoming RPS − eligible cache hits/sec.
- For equal healthy targets, Round Robin assigned RPS per target ≈ origin-bound RPS / healthy target count, before capacity limits.
- For Weighted Round Robin, backend share = backend weight / sum of healthy weights; multiply that share by origin-bound RPS.
- Least Connections must use simulated active connections. A teaching approximation may use Little's Law, `active connections ≈ assigned RPS × mean service time in seconds`, and must update as service time or load changes.
- IP Hash maps each simulated client key to a backend. Repeated requests retain affinity until the key or backend membership changes; skewed client populations must be visible.
- TLS termination adds a configurable proxy processing cost; do not treat a fixed overhead as a vendor benchmark.
- A cache hit bypasses origins only for requests classified as eligible. Authenticated/private responses must not be included by default.
- Failed targets stop receiving new requests only after the configured health-detection delay.
- If offered load exceeds proxy or origin capacity, the simulator must expose a bounded queue and explain whether excess work is rejected or times out.
- Rate limits should use a documented token-bucket model with rate and burst; delayed/rejected behavior must be explicit.
- Estimated latency should identify proxy, cache, queue, and origin contributions and be labeled simulated.

## Output metrics

- Incoming, forwarded, cache-served, rate-limited, rejected, successful, and timed-out requests/sec.
- Per-proxy and per-origin request count and utilization.
- Active connection estimate and queue depth.
- Cache hits, misses, bypasses, and origin-bound request rate.
- Health state and detection delay for proxies and origins.
- Estimated average and p95 latency.
- Routing imbalance and remaining N+1 capacity.

## Required scenarios

1. Baseline with four proxies and four equal origins.
2. One proxy instance failed; observe entry redistribution and remaining capacity.
3. One origin failed; observe health detection and traffic removal.
4. Weighted Round Robin with 1:2:1 weights.
5. Compare Least Connections under different origin service times.
6. IP Hash with uniform and skewed client populations.
7. Cache enabled with eligible public requests; verify origin request reduction.
8. Authenticated/private requests; verify cache bypass.
9. Rate and burst limits exceeded.
10. Offered load greater than proxy capacity and then greater than origin capacity.
11. TLS termination overhead and health-detection delay.
12. Bad route rollout followed by rollback to a known-good version.

## Acceptance checks

- Failed origins and proxies receive no new requests after the model's health-detection point.
- Proxy and origin utilization are calculated independently; proxy capacity does not mask an origin bottleneck.
- Per-origin counts sum to requests forwarded to origins.
- Cache hits do not exceed eligible requests; private requests bypass shared caching.
- Weighted Round Robin converges to configured shares for equal-capacity targets under similar request costs.
- Least Connections uses active-connection state, not a Round Robin substitute.
- IP Hash affinity and skew are visible and change when backend membership changes.
- Rate limits respect configured rate and burst and expose delayed/rejected request counts.
- Over-capacity traffic produces visible bounded queueing, rejection, or timeout behavior.
- Same inputs and model version produce deterministic results.
- UI labels all latency and capacity outcomes as simulated estimates.
