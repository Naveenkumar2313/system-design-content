# CS-015 — Simulation Specification

## Goal
Demonstrate token refill, burst capacity, decision capacity and dependency failures.

## Controls
| Key | Label | Type | Default | Min | Max |
|---|---|---|---|---|---|
| incoming_rps | Incoming RPS | Slider | 1000 | 10 | 100000 |
| decision_instances | Decision instances | Stepper | 3 | 1 | 20 |
| instance_capacity | Capacity per instance | Slider | 500 | 100 | 5000 |
| failed_instances | Failed instances | Stepper | 0 | 0 | decision_instances |
| refill_rps | Refill tokens/sec | Slider | 800 | 1 | 100000 |
| burst_capacity | Burst capacity | Slider | 1200 | 1 | 100000 |
| state_capacity | State capacity/sec | Slider | 2000 | 0 | 100000 |
| state_available | State available | Toggle | true | — | — |
| state_latency_ms | State latency (ms) | Slider | 3 | 1 | 1000 |
| base_overhead_ms | Base overhead (ms) | Slider | 2 | 1 | 100 |

## Derived values
- One-second deterministic model, one authenticated tenant-route key, integer controls, one token per request, no retries or queues. Initialize tokens to burst capacity; first tick has no additional refill.
- At subsequent tick boundaries refill tokens = min(burst_capacity, previous tokens + refill_rps) when state is available. During outage freeze modeled state; on recovery resume normal refill without retroactive credit. Idle scenario may inject zero offered requests as a scripted test.
- Healthy capacity = max(0, decision_instances − failed_instances) × instance_capacity.
- Admitted = min(incoming_rps, healthy capacity); worker overload = incoming_rps − admitted (503).
- Evaluated = min(admitted, state_capacity) if state available, else 0; state errors = admitted − evaluated (503).
- Allowed = min(evaluated, tokens); denied = evaluated − allowed (429); subtract allowed from tokens.
- Allowed + denied + worker overload + state errors = incoming RPS. Allowed means forwarded, not guaranteed upstream success.
- Successful decision latency (allow or deny) = base_overhead_ms + state_latency_ms; average and p95 equal this fixed value, undefined if no decisions complete. No queueing or production latency distribution modeled.
- Policy changes clamp existing tokens to new capacity; restart simulation only when explicitly requested.

## Output metrics
- Offered, admitted, evaluated, allowed, 429 and 503 requests/sec.
- Remaining tokens, healthy capacity, state utilization, per-instance distribution and dependency health.
- Modeled average and p95 completed decision latency.

## Required scenarios
1. Baseline for at least five ticks.
2. 10× traffic.
3. One decision instance failed.
4. All decision instances failed.
5. Burst exhausted.
6. State unavailable and recovery.
7. State capacity saturation.
8. Idle refill followed by a burst.

## Acceptance checks
- Baseline ticks 1–3 allow 1000, 1000, 800 respectively; remaining tokens 200, 0, 0; tick 3 denies 200. Completed decision latency 5 ms.
- Failed workers receive no new requests; zero healthy capacity rejects all traffic.
- State outage produces no forwarded requests; 503 is distinct from 429.
- Tokens never exceed capacity or fall below zero.
- All outcome categories sum to offered requests; same inputs and model version produce deterministic results.
