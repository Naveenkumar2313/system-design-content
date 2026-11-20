# CS-012 — Data Flow Specification

## Flow 1: Collect event
```mermaid
sequenceDiagram
 participant C as Website SDK
 participant LB as Load Balancer
 participant API as Collector
 participant Q as Event Broker
 C->>C: Check applicable preferences
 C->>LB: POST /api/v1/events
 LB->>API: Forward request
 API->>API: Validate schema and quotas
 API->>Q: Publish event
 Q-->>API: Durable acknowledgment
 API-->>C: 202 Accepted via LB
```
## Flow 2: Process events
```mermaid
flowchart LR
 A[Event Broker] --> B[Processing Worker]
 B --> C[Deduplication and normalization]
 C --> D[Raw Archive]
 C --> E[Analytical Store and Rollups]
 E --> F[Commit offset after durable writes]
 D --> F
```
## Flow 3: Generate report
```mermaid
sequenceDiagram
 participant D as Dashboard
 participant R as Reporting API
 participant K as Report Cache
 participant S as Analytical Store
 D->>R: GET /api/v1/reports
 R->>R: Authenticate and authorize site
 R->>K: Lookup tenant-scoped versioned key
 alt Fresh hit
 K-->>R: Report and watermark
 else Miss
 R->>S: Query rollups
 S-->>R: Counts and watermark
 R->>K: Populate cache
 end
 R-->>D: Report and freshness
```
## Error and edge-case paths
- Invalid event: 400; oversized payload: 413; quota exceeded: 429.
- Broker unavailable: 503; retry with same event ID.
- Duplicate delivery: idempotent writes and rollups.
- Late event: correct affected window and version cached reports.
- Unauthorized report: 401 or 403; store unavailable: controlled 503 or explicitly stale authorized result.
- Worker commits offsets only after required durable writes succeed.
