# CS-013 — Data Flow Specification

## Flow 1: Register monitor
```mermaid
sequenceDiagram
 participant C as Owner
 participant LB as Load Balancer
 participant API as Monitor API
 participant DB as Configuration Database
 C->>LB: POST /api/v1/monitors
 LB->>API: Forward request
 API->>API: Authenticate, verify ownership and validate target
 API->>DB: Store monitor and schedule
 DB-->>API: Commit
 API-->>C: 201 Created via LB
```
## Flow 2: Execute scheduled check
```mermaid
sequenceDiagram
 participant S as Scheduler
 participant Q as Job Broker
 participant W as Probe Worker
 participant T as Authorized Website
 participant R as Result Store
 S->>Q: Publish due job with unique check ID
 Q->>W: Lease job
 W->>W: Validate destination and deadline
 W->>T: Bounded HTTP health check
 T-->>W: Response or timeout
 W->>R: Idempotent result write
 R-->>W: Durable commit
 W->>Q: Acknowledge job
```
## Flow 3: Incident and reporting
```mermaid
flowchart LR
 A[Result Store] --> B[Incident Evaluator]
 B --> C[Incident State and Outbox]
 C --> D[Notification Worker]
 D --> E[Authorized Recipient]
 A --> F[Reporting API]
 C --> F
 F --> G[Dashboard]
```
## Error and edge-case paths
- Invalid or unauthorized target: reject registration; enforce destination policy at execution.
- Duplicate job: unique check ID prevents duplicate result and incident evaluation.
- Probe timeout: record failed observation; infrastructure failure or missed schedule: unknown.
- Store unavailable: do not acknowledge job; retry within deadline or record unknown after recovery.
- Late results: retain for audit; evaluate ordered slots using a bounded lateness policy.
- Notification failure: retry with bounded backoff and transition ID; alert operators on exhausted retries.
