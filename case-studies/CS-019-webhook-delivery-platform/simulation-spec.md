# CS-019 — Simulation Specification

## Goal

Demonstrate event routing, HTTP delivery, retries, jitter, DLQ behavior, replay, circuit breakers, tenant fairness, and at-least-once duplicates.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| events_per_second | Events/s | Slider | 350 | 1 | 100000 |
| subscriptions_per_event | Subscriptions/event | Slider | 3 | 1 | 100 |
| endpoint_failure_probability | Endpoint failure | Slider | 0.05 | 0 | 1 |
| timeout_ms | HTTP timeout | Slider | 10000 | 100 | 120000 |
| worker_count | Delivery workers | Slider | 100 | 1 | 10000 |
| max_retries | Max retries | Stepper | 6 | 0 | 20 |
| base_retry_seconds | Base retry delay | Slider | 30 | 1 | 3600 |
| jitter_fraction | Jitter fraction | Slider | 0.20 | 0 | 1 |
| per_endpoint_rps | Endpoint RPS | Slider | 50 | 1 | 10000 |
| circuit_failure_threshold | Circuit threshold | Slider | 20 | 1 | 1000 |
| random_seed | Deterministic seed | Integer | 19 | 0 | 2147483647 |

## Derived values

- Delivery tasks = events × subscriptions.
- Initial delivered bytes = tasks × payload size.
- Retry tasks = failed tasks × configured retry probability.
- Retry schedule = bounded exponential backoff plus jitter.
- Queue age = offered task rate / worker/endpoint service capacity.
- Circuit open time = cooldown × failure-state observations.
- Replay adds a separate task and preserves the original event ID.

## Required scenarios

1. One event with several matching subscriptions.
2. Transient `500` followed by success.
3. Permanent `410` and endpoint notification.
4. Timeout/DNS/network failure.
5. Worker crash after endpoint request before acknowledgment.
6. Replay and consumer duplicate handling.
7. Circuit closed/open/half-open behavior.
8. Hot tenant and endpoint fairness.
9. Queue outage and outbox recovery.
10. Replay storm and endpoint rate limit.
11. Unsafe redirect or SSRF endpoint.
12. Secret rotation and signature verification.

## Acceptance checks

- One event creates a stable delivery task per subscription.
- Duplicate tasks preserve event ID.
- Consumer example rejects duplicate side effects.
- Retry schedule is bounded and jittered.
- Permanent failure stops or enters DLQ.
- Unsafe endpoint target is never contacted.
- Replay preserves original event/payload identity.
