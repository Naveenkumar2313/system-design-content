# CS-007 — Research References and Evidence Register

## Source documents supplied for this task

1. `Codorythm_250_Case_Studies.docx.pdf`
   - Category A: Web & Internet Systems.
   - Case 7: Design a Load Balancer.
   - Establishes the case-study title and catalogue scope; it does not prescribe a vendor implementation.

2. `System_Design_Knowledge_Library.docx.pdf`
   - Provides general system-design curriculum context, including networking and traffic-management topics.
   - It does not document any specific vendor's private production architecture.

## Evidence policy for interns

For any claim about a real vendor's actual implementation:
- Prefer official product documentation, standards, engineering publications, and published papers.
- Record title, publisher, URL, product/version, access date, and the exact claim supported.
- Label third-party analysis as secondary evidence.
- Do not turn a common design pattern into a claim about a named vendor's private architecture.

## Technical references to consult and verify during research

- NGINX, `ngx_http_upstream_module`: https://nginx.org/en/docs/http/ngx_http_upstream_module.html
- NGINX, `ngx_http_proxy_module`: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- HAProxy Configuration Manual 3.2: https://docs.haproxy.org/3.2/configuration.html
- Envoy Load Balancing: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancing
- AWS Application Load Balancer target groups: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html
- AWS Application Load Balancer target-group attributes: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-group-attributes
- AWS Network Load Balancer target-group health checks: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/target-group-health-checks.html
- AWS Elastic Load Balancing documentation: https://docs.aws.amazon.com/elasticloadbalancing/
- Amazon Route 53 routing policies: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html
- TLS 1.3, RFC 8446: https://www.rfc-editor.org/rfc/rfc8446
- HTTP Semantics, RFC 9110: https://www.rfc-editor.org/rfc/rfc9110
- HTTP/1.1, RFC 9112: https://www.rfc-editor.org/rfc/rfc9112

These sources support protocol and product behavior; they do not prove a specific organization's internal architecture or capacity.

## Claims requiring external verification before publication as vendor facts

- A named organization's load-balancer technology, topology, node count, and failover design.
- Product-specific algorithm availability, selection scope, health-check behavior, and fail-open policy.
- Cross-zone, connection-draining, source-address, and TLS semantics for a particular cloud service.
- Throughput, latency, availability, and cost for a specific deployment.
- Current quotas, regional feature availability, and configuration limits.

## Evidence log

| Claim / section | Source URL | Source type | Date | Confidence | Reviewer |
|---|---|---|---|---|---|
| NGINX HTTP upstream groups default to weighted Round Robin and document alternative policies | NGINX `ngx_http_upstream_module` | Official product documentation | 2026-11-14 | High | To be assigned |
| AWS ALB target groups route to registered targets and support per-group health checks | AWS ALB target-group documentation | Official product documentation | 2026-11-14 | High | To be assigned |
| Envoy documents weighted Round Robin, weighted least request, random, ring hash, and Maglev policy families | Envoy Load Balancing documentation | Official product documentation | 2026-11-14 | High | To be assigned |
| TLS 1.3 protects client/server application communication against eavesdropping and tampering | RFC 8446 | IETF standard | 2026-11-14 | High | To be assigned |
