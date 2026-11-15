# CS-DNSService — Data Flow Specification

## Flow 1: Resolve DNS query
```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant R as Recursive Resolver
    participant K as TTL Cache
    participant Root as Root DNS
    participant T as TLD DNS
    participant A as Authoritative DNS
    C->>LB: DNS query (UDP/TCP)
    LB->>R: Forward query
    R->>K: Lookup QNAME/QTYPE/QCLASS
    alt Valid cache hit
        K-->>R: RRset or negative answer
    else Cache miss
        R->>Root: Query if referral not cached
        Root-->>R: TLD referral
        R->>T: Query
        T-->>R: Authoritative referral
        R->>A: Query
        A-->>R: Answer or negative response
        R->>R: Validate DNSSEC and TTL
        R->>K: Cache eligible result
    end
    R-->>LB: DNS response
    LB-->>C: DNS response
```

## Flow 2: Publish authoritative zone update
```mermaid
sequenceDiagram
    participant O as Zone Owner
    participant API as Zone API
    participant DB as Zone Store
    participant P as Publisher
    participant A as Authoritative Replicas
    O->>API: Submit record change
    API->>API: Authenticate and validate
    API->>DB: Commit version and serial
    DB-->>API: Commit success
    API-->>O: Accepted, publication pending
    P->>DB: Read committed version
    P->>A: Distribute validated snapshot
    A-->>P: Acknowledge version
```

## Flow 3: Analytics
```mermaid
flowchart LR
    A[Resolver] -->|Aggregate telemetry| B[Event Broker]
    B --> C[Metrics Worker]
    C --> D[Metrics Store]
    D --> E[Dashboard]
```

## Error and edge-case paths
- Unknown name: NXDOMAIN; existing name without requested type: NOERROR/NODATA.
- Expired cache entry: resolve again; do not reset TTL on cache hits.
- Upstream timeout: bounded retries, then SERVFAIL.
- Bogus DNSSEC result: SERVFAIL under normal validation policy.
- Truncated UDP response: retry using TCP as appropriate.
- Publication failure: retain last valid zone version and retry; report pending status.
- Telemetry outage: follow defined buffering/drop policy without blocking resolution.
