# CMP-001 — Load Balancer Simulation Specification

## Goal
Show how a load balancer distributes incoming requests across healthy backend targets and how routing policy, capacity, and failure affect the system.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| incoming_rps | Incoming requests/sec | Slider | 1000 | 10 | 100000 |
| backend_count | Backend servers | Stepper | 3 | 1 | 20 |
| backend_capacity_rps | Capacity per server | Slider | 500 | 100 | 5000 |
| algorithm | Routing algorithm | Select | round_robin | — | — |
| failed_backends | Failed backends | Stepper | 0 | 0 | backend_count |
| service_time_ms | Average service time | Slider | 20 | 1 | 1000 |
| request_pattern | Request distribution | Select | uniform | — | — |

## Simplified model

- Healthy backend count = backend count − failed backends.
- Total configured healthy capacity = healthy backend count × capacity per server.
- If incoming RPS is below capacity, distribute requests according to the selected algorithm.
- If incoming RPS exceeds capacity, show overload. The chosen model must specify whether excess requests queue, time out, or are rejected.
- Round Robin should distribute approximately evenly across healthy equal-weight targets over a sufficiently large sample.
- Weighted Round Robin should distribute according to configured weights.
- Least Connections requires simulated active connection counts; do not approximate it as Round Robin.

## Outputs

- Requests routed to each backend.
- Backend utilization.
- Successful, rejected, and timed-out requests.
- Health status.
- Estimated latency.
- Distribution imbalance indicator.

## Required experiments

1. Three equal backends, Round Robin.
2. One backend failed.
3. One backend with half capacity.
4. Weighted Round Robin with unequal weights.
5. Traffic greater than total healthy capacity.
6. Compare Round Robin and Least Connections under different request durations.

## Acceptance criteria

- Failed backends receive no new requests once marked unhealthy.
- Total routed traffic equals accepted traffic in the model.
- Per-backend counts sum to total routed requests.
- Overload is visible and explained.
- Same inputs and model version produce deterministic results.
- UI labels all values as simulated estimates.
