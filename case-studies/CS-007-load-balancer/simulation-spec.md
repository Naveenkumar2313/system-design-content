# CS-007 — Simulation Specification

## Goal
Demonstrate how load-balancing algorithm, backend capacity, request duration, health detection, and load-balancer redundancy affect traffic distribution, throughput, latency, and failure recovery.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| incoming_rps | Incoming requests/sec | Slider | 1000 | 10 | 100000 |
| backend_count | Backend servers | Stepper | 3 | 1 | 20 |
| backend_capacity_rps | Capacity per server | Slider | 5000 | 100 | 20000 |
| backend_weights | Backend weights | Stepper array | 1,1,1 | 1 | 10 |
| algorithm | Routing algorithm | Select | round_robin | — | — |
| lb_instances | Load-balancer nodes | Stepper | 2 | 1 | 20 |
| lb_capacity_rps | Capacity per LB node | Slider | 8000 | 100 | 50000 |
| failed_backends | Failed backends | Stepper | 0 | 0 | backend_count |
| failed_lb_nodes | Failed LB nodes | Stepper | 0 | 0 | lb_instances |
| service_time_ms | Average service time | Slider | 20 | 1 | 1000 |
| request_pattern | Request cost distribution | Select | uniform | — | — |
| health_interval_seconds | Health-check interval | Slider | 5 | 1 | 60 |
| unhealthy_threshold | Consecutive failures | Slider | 3 | 1 | 10 |
| session_persistence | Enable affinity | Toggle | false | — | — |
| tls_termination | Terminate TLS | Toggle | false | — | — |
| tls_overhead_ms | Modeled TLS processing | Slider | 1 | 0 | 20 |

## Derived values

These are simplified teaching-model equations. The implementation must document and test them.

- Healthy load balancers = LB instances − failed LB nodes.
- Effective LB capacity = healthy LB nodes × per-node capacity.
- Healthy backends = backend count − failed backends.
- Effective backend capacity = sum of capacity for healthy backends.
- At incoming rate `λ` RPS and mean connection/service lifetime `W` seconds, estimated average in-flight work is `L = λ × W` (Little's Law).
- Round Robin assigns equal counts to equal healthy targets over a sufficiently large sample, before capacity limits.
- Weighted Round Robin target share = `target_weight / sum(healthy_target_weights)`.
- Least Connections chooses the target with the fewest simulated active connections.
- A common Weighted Least Connections model chooses the lowest `active_connections / weight`; it must not be presented as a universal product formula.
- IP Hash assigns by a deterministic client key and current target set; skewed client populations and target changes must affect results.
- Random selection uses a deterministic pseudo-random seed for repeatability; expected shares approach equal distribution only over sufficiently large samples.
- Least Response Time uses a documented latency moving average and outstanding-work term. The exact model and tie-breaker must be explicit.
- Failed targets stop receiving new work only after the configured health-detection delay.
- When offered traffic exceeds LB or backend capacity, model a bounded queue and show excess as rejected or timed out; never use an unbounded queue.
- TLS overhead consumes modeled LB capacity when termination is enabled; it is an estimate, not a product benchmark.
- Persistence pins the modeled client to a target where possible and can create imbalance; behavior after target removal must be defined.
- Report application database utilization only as a downstream dependency of the application servers, not a per-request load-balancer query.

## Output metrics

- Requests/connections assigned per target and per LB node.
- Healthy target count, health-detection time, and target recovery.
- Per-target active connections, utilization, and estimated response time.
- Successful, rejected, queued, and timed-out work.
- LB-node utilization and remaining N+1 capacity.
- Estimated average and p95 latency, labeled simulated.
- Distribution imbalance and affinity impact.
- Application database load only for modeled backend requests that access it.

## Required scenarios

1. Three equal targets under Round Robin.
2. Weighted Round Robin with weights 1:2:1.
3. Least Connections with different connection durations.
4. Weighted Least Connections with unequal target capacity.
5. IP Hash with uniform and skewed client populations.
6. Random selection over a short and long sample.
7. Least Response Time with one slow target and noisy observations.
8. One backend failed with different health-check thresholds.
9. One LB node failed in active-active and active-passive modes.
10. Traffic exceeds backend capacity, then LB capacity.
11. Sticky sessions enabled with one overloaded target.
12. Compare L4 pass-through and L7 TLS termination overhead.

## Acceptance checks

- Failed backends receive no new work after modeled health detection.
- Failed LB nodes are excluded from new frontend distribution after node detection.
- Per-target assignments sum to all accepted backend-bound traffic.
- Weighted Round Robin converges to configured shares for equal-capacity targets and similar work.
- Least Connections uses active connection state rather than Round Robin.
- Weighted Least Connections uses the declared scoring formula and updates with weight changes.
- IP Hash affinity and client skew are observable; pool changes may remap clients.
- Random results are deterministic for the same seed and model version.
- Least Response Time identifies its averaging window and does not claim measured production latency.
- Queueing, rejection, and timeout behavior is bounded and visible when capacity is exceeded.
- Database load is attributed to backend application work, not load-balancer lookups.
- UI labels all latency and capacity outcomes as simulated estimates.
