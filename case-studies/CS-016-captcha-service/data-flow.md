# CS-016 — Data Flow Specification

## Flow 1: Challenge issuance
```mermaid
sequenceDiagram
    participant S as Site backend
    participant LB as Load Balancer
    participant A as Challenge API
    participant DB as State Store
    S->>LB: Request challenge
    LB->>A: Forward request
    A->>DB: Record challenge lifecycle
    A-->>S: Challenge reference
```
## Flow 2: Verification
```mermaid
sequenceDiagram
    participant S as Site backend
    participant LB as Load Balancer
    participant V as Verification Service
    participant DB as State Store
    S->>LB: Submit proof
    LB->>V: Forward request
    V->>DB: Validate lifecycle and consume if valid
    DB-->>V: State outcome
    V-->>S: Verification decision
```
## Flow 3: Telemetry
```mermaid
flowchart LR
    A[Challenge API] --> T[Telemetry pipeline]
    V[Verification Service] --> T
    T --> M[Aggregate metrics]
```
## Error and edge-case paths
- Expired or consumed proof: reject and offer fresh verification.
- State unavailable: controlled unavailable response; no positive decision.
- Telemetry unavailable: bounded buffering and alerting.
- Accessibility issue: offer alternative verification or recovery.
