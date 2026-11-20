# CS-012 — Simulation Specification

## Goal
Demonstrate event ingestion capacity, durable buffering, worker throughput, backlog, freshness and failures.

## Controls
| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| incoming_eps | Incoming events/sec | Slider | 1000 | 10 | 100000 |
| app_instances | Collector instances | Stepper | 3 | 1 | 20 |
| instance_capacity_eps | Capacity per collector | Slider | 500 | 100 | 5000 |
| worker_instances | Worker instances | Stepper | 3 | 1 | 20 |
| worker_capacity_eps | Capacity per worker | Slider | 400 | 100 | 5000 |
| broker_capacity | Broker acceptance capacity/sec | Slider | 5000 | 100 | 100000 |
| backlog_limit | Broker backlog limit | Slider | 100000 | 1000 | 1000000 |
| failed_instances | Failed collectors | Stepper | 0 | 0 | app_instances |
| broker_available | Broker available | Toggle | true | — | — |
| store_available | Analytical store available | Toggle | true | — | — |
| lb_algorithm | Routing algorithm | Select | round_robin | — | — |

## Derived values
- Use one-second ticks; initial backlog B = 0. Assume unique valid events and no automatic retries in the model.
- Collector capacity = max(0, app_instances − failed_instances) × instance_capacity_eps.
- Worker capacity = worker_instances × worker_capacity_eps.
- Accepted events = min(incoming_eps, collector capacity, broker_capacity, max(0, backlog_limit − B)) when broker available; otherwise zero.
- Processed events = min(B + accepted events, worker capacity) when broker and store available; otherwise zero.
- Next backlog = B + accepted − processed; rejected = incoming_eps − accepted.
- This conservative model reserves queue space before processing each tick. Accepted events are not lost during outages.
- Backlog drain estimate = next backlog / worker capacity when dependencies available and new arrivals stop; unavailable otherwise. This is not measured report freshness; track oldest unprocessed event age using FIFO cohorts and report watermark separately.
- Utilization = offered demand / capacity; undefined at zero capacity.
- Routing: Round Robin cycles healthy collectors; Least Connections chooses lowest active count with stable tie-breaking using deterministic request durations. Aggregate capacity model does not predict routing-specific throughput differences.

## Output metrics
- Incoming, accepted, rejected and processed events/sec.
- Per-collector request distribution, collector and worker utilization.
- Broker backlog, drain estimate, oldest unprocessed event age and report watermark.
- Health state and dependency availability.

## Required scenarios
1. Baseline.
2. 10× traffic.
3. One collector failed.
4. Worker saturation.
5. Broker unavailable.
6. Analytical store unavailable.
7. Recovery and backlog drain.
8. Compare Round Robin and Least Connections.

## Acceptance checks
- Baseline accepts and processes 1000 events/sec with zero backlog.
- Failed collectors receive no new requests after detection.
- Worker saturation increases backlog until admission is limited.
- Broker outage accepts no events; store outage preserves accepted backlog.
- Recovery drains backlog when processing exceeds arrivals.
- Accepted events equal processed events plus backlog change per tick.
- Results deterministic for same inputs and model version.
