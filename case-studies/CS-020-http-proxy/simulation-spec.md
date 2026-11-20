# CS-020 — Simulation Specification

## Goal

Demonstrate proxy throughput, connection reuse, DNS behavior, cache HIT/MISS, TLS CONNECT, authentication policy, request smuggling protection, failure, and scaling.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| requests_per_second | Requests/s | Slider | 100000 | 1 | 1000000 |
| active_connections | Active connections | Slider | 500000 | 1 | 10000000 |
| average_response_kib | Average response | Slider | 100 | 1 | 102400 |
| pool_reuse_ratio | Upstream pool reuse | Slider | 0.80 | 0 | 1 |
| cache_hit_ratio | Cache hit ratio | Slider | 0.40 | 0 | 1 |
| dns_cache_hit_ratio | DNS cache hit ratio | Slider | 0.95 | 0 | 1 |
| connect_request_share | CONNECT share | Slider | 0.30 | 0 | 1 |
| tls_interception_share | Interception share | Slider | 0.10 | 0 | 1 |
| upstream_failure_probability | Upstream failure | Slider | 0.01 | 0 | 1 |
| max_connections_per_node | Node connection limit | Slider | 100000 | 100 | 1000000 |
| random_seed | Deterministic seed | Integer | 20 | 0 | 2147483647 |

## Derived values

- Payload throughput `RPS × response_KiB × 8 / 10^9 Gbit/s`.
- Expected upstream connections = requests × `(1 - pool_reuse_ratio)` adjusted for method/authority compatibility.
- Cache miss origin load = requests × `(1 - cache_hit_ratio)`.
- DNS lookup load = requests × `(1 - dns_cache_hit_ratio)`.
- Required nodes = active connections / max connections per node and traffic/CPU limits.
- TLS handshake load = CONNECT/interception traffic × certificate/session behavior.
- Latency includes parser, auth, DNS, pool, upstream, stream, and log work.

## Required scenarios

1. Forward absolute-form HTTP GET.
2. CONNECT pass-through and 200 established.
3. Intercepted TLS workflow.
4. Cache HIT/MISS/conditional 304.
5. Pool reuse versus cold upstream.
6. DNS failure and rebinding.
7. Conflicting CL/TE request-smuggling attempt.
8. Private destination/metadata SSRF attempt.
9. Slow upstream and circuit breaker.
10. Connection limit/backpressure.
11. Traffic spike and horizontal scale.
12. Proxy node failure and client reconnect.

## Acceptance checks

- Plain HTTP destination comes from absolute-form/Host consistently.
- CONNECT never decrypts pass-through bytes.
- Interception is explicitly configured and trust-controlled.
- Host/private IP policy is validated.
- Smuggling-conflict requests are rejected.
- Cache excludes private/auth/error policy as configured.
- Pool reuse does not mix authority/TLS credentials.
- Outputs are labeled as estimates.
