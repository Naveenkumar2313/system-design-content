# CS-006 — Research References and Evidence Register

## Source documents supplied for this task

1. `Codorythm_250_Case_Studies.docx.pdf`
   - Category A: Web & Internet Systems.
   - Case 6: Design a Reverse Proxy.
   - Establishes the case-study title and catalogue scope; it does not prescribe a product implementation.

2. `System_Design_Knowledge_Library.docx.pdf`
   - Provides general system-design curriculum context.
   - It does not document any specific reverse-proxy vendor's private deployment.

## Evidence policy for interns

For any claim about a real vendor's actual implementation:
- Prefer official product documentation, engineering publications, standards, or published papers.
- Record title, publisher, URL, product/version, access date, and the exact claim supported.
- Label third-party analysis as secondary evidence.
- Do not convert a common design pattern into a claim about a named vendor's private architecture.

## Technical references to consult and verify during research

- NGINX, `ngx_http_proxy_module`: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- NGINX, `ngx_http_upstream_module`: https://nginx.org/en/docs/http/ngx_http_upstream_module.html
- NGINX, `ngx_http_limit_req_module`: https://nginx.org/en/docs/http/ngx_http_limit_req_module.html
- NGINX, `ngx_http_ssl_module`: https://nginx.org/en/docs/http/ngx_http_ssl_module.html
- HAProxy Configuration Manual 3.2: https://docs.haproxy.org/3.2/configuration.html
- HAProxy Proxying Essentials: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/
- Envoy HTTP Connection Management: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_connection_management
- Envoy Upstream Health Checking: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/health_checking
- HTTP Semantics (RFC 9110): https://www.rfc-editor.org/rfc/rfc9110
- HTTP Caching (RFC 9111): https://www.rfc-editor.org/rfc/rfc9111
- HTTP/1.1 (RFC 9112): https://www.rfc-editor.org/rfc/rfc9112
- TLS 1.3 (RFC 8446): https://www.rfc-editor.org/rfc/rfc8446

These sources support protocol and product behavior; they do not verify a specific company's internal proxy fleet.

## Claims requiring external verification before publication as vendor facts

- A named organization's reverse-proxy products, topology, instance counts, and capacity.
- Product-specific algorithm support, retry semantics, health checks, cache behavior, and configuration limitations.
- TLS defaults, cipher policy, supported protocol versions, and certificate-management behavior for a particular release.
- Performance, latency, throughput, and cost for a specific deployment.

## Evidence log

| Claim / section | Source URL | Source type | Date | Confidence | Reviewer |
|---|---|---|---|---|---|
| To be populated | | | | | |
