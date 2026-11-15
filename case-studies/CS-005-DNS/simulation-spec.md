# CS-DNSService — Simulation Specification

## Goal
Demonstrate query traffic, load distribution, cache hits, upstream pressure, latency and failures.

## Controls
| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| query_qps | Incoming query QPS | Slider | 1000 | 10 | 100000 |
| app_instances | Resolver instances | Stepper | 3 | 1 | 20 |
| instance_capacity_qps | Capacity per instance | Slider | 500 | 100 | 5000 |
| cache_enabled | Enable cache | Toggle | true | — | — |
| cache_hit_ratio | Cache hit ratio | Slider | 90 | 0 | 100 |
| upstream_capacity | Upstream capacity/sec | Slider | 2000 | 100 | 20000 |
| upstream_latency_ms | Upstream request latency | Slider | 20 | 1 | 1000 |
| lb_algorithm | Routing algorithm | Select | round_robin | — | — |
| failed_instances | Failed resolver instances | Stepper | 0 | 0 | app_instances |
| upstream_available | Upstream available | Toggle | true | — | — |

## Derived values
- Effective capacity = max(0, instances − failed instances) × capacity per instance.
- Admitted QPS = min(incoming QPS, effective capacity).
- Hits = admitted QPS × hit ratio / 100; zero when cache disabled.
- Misses = admitted QPS − hits.
- Upstream demand = 3 × misses (fixed illustrative requests per miss).
- Completed misses = min(misses, upstream capacity / 3) when available; otherwise zero.
- Successful QPS = hits + completed misses; unsuccessful QPS = incoming QPS − successful QPS.
- Utilization = demand / capacity; report unavailable for zero capacity.
- Estimate latency using fixed base latency plus three upstream request latencies on completed misses and a documented queue penalty; calculate p95 from modeled samples. Report failed requests separately. Zone publishing is outside this throughput model.

## Output metrics
- Incoming QPS, successful responses/sec, rejected or failed queries/sec.
- Per-instance request counts, cache hits/misses, upstream demand and utilization.
- Average and p95 latency, resolver health.

## Required scenarios
1. Baseline.
2. Cache disabled.
3. 10× traffic.
4. One resolver failed.
5. All resolvers overloaded.
6. Upstream latency spike.
7. Upstream unavailable.
8. Compare Round Robin and Least Connections (TCP; UDP uses flow-aware routing).

## Acceptance checks
- Failed instances receive no new queries after health detection.
- Cache disabled increases upstream demand for equal admitted load.
- Overload increases rejection or latency according to the model.
- Upstream outage permits valid cache hits but fails unresolved misses.
- Results deterministic for same inputs and model version.
