# CS-013 — Simulation Specification

## Goal
Demonstrate scheduled website checks, worker capacity, queue backlog, target failures and alert delivery.

## Controls
| Key | Label | Type | Default | Min | Max |
|---|---|---|---|---|---|
| monitors | Active monitors | Stepper | 10000 | 1 | 100000 |
| interval_sec | Check interval (seconds) | Slider | 60 | 30 | 600 |
| worker_instances | Probe workers | Stepper | 3 | 1 | 20 |
| worker_capacity | Checks/sec per worker | Slider | 100 | 10 | 1000 |
| failed_instances | Failed workers | Stepper | 0 | 0 | worker_instances |
| queue_limit | Pending job limit | Slider | 10000 | 100 | 100000 |
| broker_available | Broker available | Toggle | true | — | — |
| store_available | Result store available | Toggle | true | — | — |
| failure_percent | Target failure percentage | Slider | 0 | 0 | 100 |
| notification_available | Notification provider available | Toggle | true | — | — |
| probe_timeout_ms | Probe timeout (ms) | Slider | 5000 | 1000 | 10000 |

## Derived values
- One-second ticks; distribute monitor due times evenly using deterministic monitor IDs. Average offered checks/sec = monitors / interval_sec.
- Healthy worker capacity = max(0, worker_instances − failed_instances) × worker_capacity. Capacity is an effective measured input, not derived from timeout.
- Admit due jobs up to available queue space when broker available. Record unadmitted jobs as missed checks; never as target downtime.
- Process up to healthy capacity from queued jobs when broker and result store available. Next backlog = previous backlog + admitted − processed. Assume atomic completion per tick; actual in-flight probes and retries are not modeled.
- Classify completed checks using deterministic per-monitor failure inputs; failure_percent controls a stable subset of targets. Timeout is a failed target check only when a probe actually runs and reaches its deadline; infrastructure failures are unknown.
- Maintain consecutive failure/success counters per monitor. Open incident after two consecutive failed checks; resolve after two consecutive successful checks. Unknown resets the streak without changing incident state.
- Generate one notification per incident transition; retain pending notifications when provider unavailable and deliver on recovery. Model provider as unlimited when available; use transition ID for deduplication.
- Utilization = offered checks / healthy capacity, undefined at zero capacity. Track queue age separately from probe latency; timeout setting bounds modeled probe duration but does not change effective capacity automatically.

## Output metrics
- Due, admitted, completed and missed checks/sec; queue depth and oldest job age.
- Per-worker check distribution and utilization; target success/failure/unknown counts.
- Open/resolved incidents, pending and delivered notifications; detection delay.
- Worker and dependency health.

## Required scenarios
1. Baseline.
2. 10× monitors.
3. One worker failed.
4. All workers unavailable.
5. Broker unavailable.
6. Result store unavailable.
7. Target failures and recovery.
8. Notification provider outage and recovery.

## Acceptance checks
- Baseline averages 166.67 checks/sec with capacity 300 and no growing backlog.
- Failed workers receive no new jobs after detection.
- Overload increases backlog or missed checks; missed checks are unknown.
- Two consecutive failed checks open one incident; two successes resolve it.
- Notification outage retains pending transitions; recovery delivers without duplicate logical notifications.
- Admitted jobs equal completed jobs plus backlog change each tick.
- Results deterministic for the same inputs and model version.
