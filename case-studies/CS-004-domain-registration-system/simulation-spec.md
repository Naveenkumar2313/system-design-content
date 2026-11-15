# CS-DomainRegistration — Simulation Specification

## Goal
Demonstrate search load, cache hits, registry quotas, latency and registration backlog.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---|---|---|
| search_rps | Incoming search RPS | Slider | 200 | 10 | 100000 |
| create_rps | Registration orders/sec | Slider | 5 | 0 | 1000 |
| app_instances | API instances | Stepper | 3 | 1 | 20 |
| instance_capacity_rps | Capacity per instance | Slider | 500 | 100 | 5000 |
| cache_enabled | Enable cache | Toggle | true | — | — |
| cache_hit_ratio | Cache hit ratio | Slider | 80 | 0 | 100 |
| registry_capacity | Registry operations/sec | Slider | 100 | 1 | 20000 |
| registry_latency_ms | Registry latency | Slider | 100 | 1 | 5000 |
| lb_algorithm | Routing algorithm | Select | round_robin | — | — |
| failed_instances | Failed API instances | Stepper | 0 | 0 | app_instances |
| worker_capacity | Worker jobs/sec | Slider | 10 | 1 | 1000 |
| broker_available | Broker available | Toggle | true | — | — |

## Derived values
- Healthy capacity = max(0, instances − failed instances) × capacity per instance.
- Admitted searches = min(search RPS, healthy capacity); reserve API capacity for order submissions first.
- Cache hits = admitted searches × hit ratio (zero when disabled); misses = admitted searches − hits.
- Registry demand = misses + 2 × attempted registration jobs (one check and one create in this simplified model).
- Reserve registry capacity for jobs; cap search misses at remaining registry capacity. Jobs completed/sec ≤ min(worker capacity, queued jobs/sec, registry capacity / 2); zero if broker unavailable.
- Backlog next tick = max(0, backlog + admitted orders − completed jobs) for one-second ticks.
- Utilization = demand / capacity; zero capacity is unavailable, not a division by zero.
- Estimate latency from base + registry latency on misses + documented queue penalty; report p95 from simulated samples. Payment failures and renewals are outside this simplified throughput model.

## Output metrics
- Incoming RPS, successful searches/sec, rejected requests/sec, per-instance counts, cache hits/misses, registry utilization, average/p95 latency, order backlog and instance health.

## Required scenarios
1. Baseline.
2. Cache disabled.
3. 10× traffic.
4. One API instance failed.
5. All instances overloaded.
6. Registry latency spike.
7. Broker unavailable.
8. Compare Round Robin and Least Connections.

## Acceptance checks
- Failed instances receive no new requests after detection.
- Cache disabled increases registry demand for equal admitted search load.
- Overload increases rejection or backlog.
- Broker outage preserves durable pending orders; no false confirmation.
- Same inputs and model version produce deterministic results.
