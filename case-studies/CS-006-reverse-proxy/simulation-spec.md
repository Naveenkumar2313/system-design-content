# CS-ReverseProxy — Simulation Specification

## Goal
Demonstrate proxy load distribution, cache effectiveness, origin pressure, latency and failures.

## Controls
| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| incoming_rps | Incoming RPS | Slider | 1000 | 10 | 100000 |
| app_instances | Proxy instances | Stepper | 3 | 1 | 20 |
| instance_capacity_rps | Capacity per instance | Slider | 500 | 100 | 5000 |
| cache_enabled | Enable cache | Toggle | true | — | — |
| eligible_fraction | Cache-eligible fraction | Slider | 50 | 0 | 100 |
| cache_hit_ratio | Eligible hit ratio | Slider | 80 | 0 | 100 |
| origin_capacity | Origin capacity/sec | Slider | 2000 | 100 | 20000 |
| origin_latency_ms | Origin latency | Slider | 20 | 1 | 1000 |
| lb_algorithm | Routing algorithm | Select | round_robin | — | — |
| failed_instances | Failed proxy instances | Stepper | 0 | 0 | app_instances |
| origin_available | Origin available | Toggle | true | — | — |

## Derived values
- Effective capacity = healthy instances × capacity per instance.
- Admitted RPS = min(incoming RPS, effective capacity).
- Hits = admitted RPS × eligible fraction / 100 × hit ratio / 100 when cache enabled; otherwise zero.
- Origin demand = admitted RPS − hits.
- Successful origin RPS = min(origin demand, origin capacity) when available; otherwise zero.
- Successful RPS = hits + successful origin RPS; unsuccessful RPS = incoming RPS − successful RPS.
- Utilization = offered demand / capacity; undefined for zero capacity.
- Illustrative latency: 2 ms base for hits, 2 ms + origin latency for successful forwarded requests; calculate weighted mean and p95 from these modeled samples. Failed requests reported separately.
- Model routing with deterministic request durations and assignment; Round Robin cycles healthy instances, Least Connections selects lowest active count with stable tie-breaking. Aggregate capacity model does not predict routing-specific throughput differences.

## Output metrics
- Incoming, successful and unsuccessful RPS.
- Per-instance request counts, cache hits and misses, origin demand and utilization.
- Estimated mean and p95 latency, health state.

## Required scenarios
1. Baseline.
2. Cache disabled.
3. 10× traffic.
4. One proxy failed.
5. All proxies overloaded.
6. Origin latency spike.
7. Origin unavailable.
8. Compare Round Robin and Least Connections.

## Acceptance checks
- Failed instances receive no new requests after detection.
- Cache disabled increases origin demand at equal admitted load.
- Overload increases rejection according to the model.
- Origin outage allows fresh eligible hits but fails forwarded requests.
- Results deterministic for same inputs and model version.
