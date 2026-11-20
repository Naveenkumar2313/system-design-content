# CS-014 — Simulation Specification

## Goal
Demonstrate gateway capacity, quotas, cache and upstream limits.

## Controls
| Key | Label | Type | Default | Min | Max |
| --- | --- | --- | --- | --- | --- |
| incoming_rps | Incoming RPS | Slider | 1000 | 10 | 100000 |
| gateway_instances | Gateway instances | Stepper | 3 | 1 | 20 |
| instance_capacity | Capacity per instance | Slider | 500 | 100 | 5000 |
| failed_instances | Failed instances | Stepper | 0 | 0 | gateway_instances |
| quota_rps | Aggregate quota RPS | Slider | 1200 | 0 | 100000 |
| quota_available | Quota store available | Toggle | true | — | — |
| cache_enabled | Cache enabled | Toggle | false | — | — |
| cache_hit_ratio | Eligible cache hit ratio (%) | Slider | 30 | 0 | 100 |
| upstream_capacity | Upstream capacity RPS | Slider | 2000 | 0 | 100000 |
| upstream_latency_ms | Upstream latency (ms) | Slider | 80 | 1 | 1000 |
| gateway_overhead_ms | Gateway overhead (ms) | Slider | 10 | 1 | 100 |

## Derived values
- One-second deterministic fluid model; all requests valid and authorized, all are eligible public GETs for the optional cache experiment. No retries or queues modeled.
- Healthy capacity = max(0, gateway_instances − failed_instances) × instance_capacity.
- Admitted = min(incoming_rps, healthy capacity); gateway overload = incoming_rps − admitted (503).
- If quota unavailable, all admitted requests return 503; otherwise allowed = min(admitted, quota_rps); quota rejections = admitted − allowed (429).
- Cache hits = allowed × cache_hit_ratio / 100 if enabled, else zero. Upstream offered = allowed − hits.
- Upstream successes = min(upstream offered, upstream_capacity); upstream overload = upstream offered − successes (503).
- Successes = cache hits + upstream successes. All categories sum to offered RPS. Aggregate quota is a per-tick teaching approximation, not a production token bucket.
- Successful cache hits take gateway_overhead_ms; successful misses take gateway_overhead_ms + upstream_latency_ms. Average is weighted by successful counts; p95 is the smallest modeled latency covering 95% of successes. Undefined when no successes. No queueing or real latency distribution modeled.

## Output metrics
- Offered, admitted, successful, 429 and 503 requests/sec.
- Healthy capacity, per-instance distribution, cache hits, upstream offered and successful RPS.
- Modeled average and p95 successful latency; dependency health.

## Required scenarios
1. Baseline.
2. 10× traffic.
3. One gateway failed.
4. All gateways failed.
5. Quota exhausted.
6. Quota store unavailable.
7. Cache enabled.
8. Upstream saturation.

## Acceptance checks
- Baseline: 1,000 successes/sec, 1,500 gateway capacity, 90 ms modeled latency.
- Failed replicas receive no new requests; zero healthy capacity rejects all offered traffic.
- Quota outage produces no forwarded requests.
- Cache enabled at baseline reduces upstream offered RPS to 700.
- Successes + all rejection categories equal incoming RPS.
- Same inputs and model version produce deterministic results.
