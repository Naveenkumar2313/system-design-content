# CS-016 — Simulation Specification

## Goal
Demonstrate capacity and availability using synthetic verification requests, not real challenge solving.
## Controls
| Key | Label | Type | Default | Min | Max |
|---|---|---|---|---|---|
| issue_rps | Issuance RPS | Slider | 100 | 0 | 10000 |
| verify_rps | Verification RPS | Slider | 100 | 0 | 10000 |
| instances | Service instances | Stepper | 3 | 1 | 20 |
| instance_capacity | Capacity per instance | Slider | 100 | 1 | 5000 |
| failed_instances | Failed instances | Stepper | 0 | 0 | instances |
| state_capacity | State operations/sec | Slider | 500 | 0 | 20000 |
| state_available | State available | Toggle | true | — | — |
| state_latency_ms | State latency | Slider | 20 | 1 | 1000 |
## Derived values
- One-second deterministic model; one abstract state operation per admitted request; no queues or retries.
- Offered = issue_rps + verify_rps.
- Healthy capacity = max(0, instances − failed_instances) × instance_capacity.
- Admitted = min(offered, healthy capacity).
- Completed = min(admitted, state_capacity) if state available; otherwise zero.
- Unavailable = offered − completed. Completed means a decision or issuance response, not necessarily a successful proof.
- Modeled completed latency = 5 ms + state_latency_ms; mean and p95 equal this fixed value; undefined when no requests complete.
## Output metrics
Offered, admitted, completed, unavailable, healthy capacity and modeled latency.
## Required scenarios
1. Baseline.
2. 10× traffic.
3. One failed instance.
4. All instances failed.
5. State unavailable and recovery.
6. State capacity saturation.
7. State latency spike.
## Acceptance checks
- Completed + unavailable = offered.
- Failed instances receive no new work.
- State outage produces zero completed requests.
- Identical inputs produce identical outputs.
- No claim about real-world challenge effectiveness.
