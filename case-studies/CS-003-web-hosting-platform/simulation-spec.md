# CS-WebHosting — Simulation Specification

## Goal
Demonstrate the relationship between website traffic, load distribution, cache hit ratio, database pressure, latency, and failures.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| serve_rps | Incoming website RPS | Slider | 1000 | 10 | 100000 |
| deploy_rps | Deployment jobs/sec | Slider | 1 | 0 | 100 |
| app_instances | Origin instances | Stepper | 3 | 1 | 20 |
| instance_capacity_rps | Capacity per instance | Slider | 500 | 100 | 5000 |
| cache_enabled | Enable cache | Toggle | true | — | — |
| cache_hit_ratio | Cache hit ratio | Slider | 90 | 0 | 100 |
| db_read_capacity | Origin storage read capacity/sec | Slider | 2000 | 100 | 20000 |
| db_latency_ms | Origin storage read latency | Slider | 20 | 1 | 1000 |
| lb_algorithm | Routing algorithm | Select | round_robin | — | — |
| failed_instances | Failed origin instances | Stepper | 0 | 0 | app_instances |
| worker_capacity | Deployment worker capacity/jobs per sec | Slider | 2 | 1 | 100 |
| broker_available | Broker available | Toggle | true | — | — |

## Derived values

These are simplified teaching-model equations. The implementation must document and test them.

- Effective origin capacity = healthy instances × per-instance capacity.
- Origin utilization = cache misses/sec / effective origin capacity; deployment workers are separate.
- Cache hits/sec = incoming website RPS × effective cache hit ratio (zero when cache is disabled).
- Cache misses/sec = incoming website RPS − cache hits/sec.
- Origin storage lookup RPS = cache misses/sec (one lookup per miss in this model).
- Storage utilization = origin storage lookup RPS / configured database read capacity.
- Estimated overload occurs when offered load exceeds configured capacity.

- Deployment backlog growth/sec = max(0, deployment RPS − worker capacity) when broker is available; otherwise jobs remain pending in durable metadata.
- With zero healthy origins, misses fail; cache hits can still succeed.

Latency model should be explicitly documented. A simple simulator may use fixed base latency plus dependency latency and a queueing penalty near saturation. Do not label this as measured production latency.

## Output metrics

- Incoming RPS
- Successful website requests/sec
- Rejected / timed-out requests/sec
- Per-instance request count
- Cache hit and miss counts
- Database website RPS and utilization
- Estimated average and p95 latency
- Broker event backlog
- Health state of each instance

## Required scenarios

1. Baseline.
2. Cache disabled.
3. 10× traffic.
4. One origin instance failed.
5. All origin instances overloaded.
6. Storage latency spike.
7. Broker unavailable.
8. Compare Round Robin and Least Connections.

## Acceptance checks

- A failed instance receives no new requests after health detection.
- Turning cache off increases origin requests.
- Raising traffic beyond total capacity increases queueing, rejection, or latency according to the model.
- Broker failure affects deployment progress/durability according to the configured policy, not silently.
- Results are deterministic for the same inputs and model version.
