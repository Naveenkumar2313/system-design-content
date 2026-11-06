# CS-001 — Simulation Specification

## Goal
Demonstrate the relationship between redirect traffic, load distribution, cache hit ratio, database pressure, latency, and failures.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| redirect_rps | Incoming redirect RPS | Slider | 1000 | 10 | 100000 |
| create_rps | Link creation RPS | Slider | 10 | 0 | 10000 |
| app_instances | Application instances | Stepper | 3 | 1 | 20 |
| instance_capacity_rps | Capacity per instance | Slider | 500 | 100 | 5000 |
| cache_enabled | Enable cache | Toggle | true | — | — |
| cache_hit_ratio | Cache hit ratio | Slider | 90 | 0 | 100 |
| db_read_capacity | Database read capacity/sec | Slider | 2000 | 100 | 20000 |
| db_latency_ms | Database read latency | Slider | 20 | 1 | 1000 |
| lb_algorithm | Routing algorithm | Select | round_robin | — | — |
| failed_instances | Failed app instances | Stepper | 0 | 0 | app_instances |
| broker_available | Broker available | Toggle | true | — | — |

## Derived values

These are simplified teaching-model equations. The implementation must document and test them.

- Effective application capacity = healthy instances × per-instance capacity.
- Application utilization = incoming redirect RPS / effective application capacity.
- Cache hits/sec = redirect RPS × cache hit ratio.
- Cache misses/sec = redirect RPS − cache hits/sec.
- Database lookup RPS = cache misses/sec + any required uncached management reads.
- Database utilization = database lookup RPS / configured database read capacity.
- Estimated overload occurs when offered load exceeds configured capacity.

Latency model should be explicitly documented. A simple simulator may use fixed base latency plus dependency latency and a queueing penalty near saturation. Do not label this as measured production latency.

## Output metrics

- Incoming RPS
- Successful redirects/sec
- Rejected / timed-out requests/sec
- Per-instance request count
- Cache hit and miss counts
- Database read RPS and utilization
- Estimated average and p95 latency
- Broker event backlog
- Health state of each instance

## Required scenarios

1. Baseline.
2. Cache disabled.
3. 10× traffic.
4. One application instance failed.
5. All application instances overloaded.
6. Database latency spike.
7. Broker unavailable.
8. Compare Round Robin and Least Connections.

## Acceptance checks

- A failed instance receives no new requests after health detection.
- Turning cache off increases database lookups.
- Raising traffic beyond total capacity increases queueing, rejection, or latency according to the model.
- Broker failure affects analytics freshness/durability according to the configured policy, not silently.
- Results are deterministic for the same inputs and model version.
