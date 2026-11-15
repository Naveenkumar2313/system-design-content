# CS-008 — Research References and Evidence Register

## Source documents supplied for this task

1. `Codorythm_250_Case_Studies.docx.pdf`
   - Category A: Web & Internet Systems.
   - Case 8: Design a CDN.
   - Establishes the case-study title and catalogue scope; it does not prescribe a vendor implementation.

2. `System_Design_Knowledge_Library.docx.pdf`
   - Provides general system-design curriculum context.
   - It does not document any specific vendor's private production architecture.

## Evidence policy for interns

For any claim about a real vendor's actual implementation:
- Prefer official product documentation, standards, engineering publications, and published papers.
- Record title, publisher, URL, product/version, access date, and the exact claim supported.
- Label third-party analysis as secondary evidence.
- Do not turn a common design pattern into a claim about a named vendor's private architecture.

## Technical references to consult and verify during research

- Amazon CloudFront, Introduction and delivery concepts: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html
- Amazon CloudFront, cache expiration and stale content: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html
- Amazon CloudFront, origins and Origin Shield: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesOrigin.html
- Amazon CloudFront, invalidation and versioned filenames: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html
- Amazon CloudFront, signed URLs: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-signed-urls.html
- Cloudflare, cache and origin Cache-Control: https://developers.cloudflare.com/cache/concepts/cache-control/
- Cloudflare, purge cache: https://developers.cloudflare.com/cache/how-to/purge-cache/
- Google Cloud CDN, caching: https://docs.cloud.google.com/cdn/docs/caching
- Google Cloud CDN, signed URLs and private content: https://docs.cloud.google.com/cdn/docs/authenticate-content
- Akamai Content Delivery Network product overview: https://www.akamai.com/products/content-delivery-network
- HTTP Caching, RFC 9111: https://www.rfc-editor.org/rfc/rfc9111
- HTTP Semantics, RFC 9110: https://www.rfc-editor.org/rfc/rfc9110
- HTTP Cache-Control Extensions for Stale Content, RFC 5861: https://www.rfc-editor.org/rfc/rfc5861
- TLS 1.3, RFC 8446: https://www.rfc-editor.org/rfc/rfc8446

These sources support protocol and product behavior; they do not prove a specific organization's private architecture, cache topology, performance, or capacity.

## Claims requiring external verification before publication as vendor facts

- A named provider's exact PoP count, cache hierarchy, routing algorithm, and failure topology.
- Product-specific cache key defaults, TTL precedence, negative caching, stale behavior, and purge semantics.
- Request-collapsing or Origin Shield behavior and its scope for a particular product/tier.
- DNS/anycast selection, health checks, route convergence, and regional failover time.
- Signed URL validation order and cache behavior for the selected provider.
- Current quotas, regional feature availability, prices, and performance benchmarks.

## Evidence log

| Claim / section | Source URL | Source type | Date | Confidence | Reviewer |
|---|---|---|---|---|---|
| CloudFront uses edge locations and retrieves cache misses from configured origins | Amazon CloudFront Introduction | Official product documentation | 2026-11-14 | High | To be assigned |
| CloudFront cache freshness can be controlled by TTL/cache headers and stale entries may be revalidated | Amazon CloudFront Expiration | Official product documentation | 2026-11-14 | High | To be assigned |
| CloudFront can use a configured Origin Shield between edge and origin | CloudFront Origins / Origin Shield | Official product documentation | 2026-11-14 | High | To be assigned |
| CloudFront supports invalidation and recommends versioned file names for frequently updated files | CloudFront Invalidation | Official product documentation | 2026-11-14 | High | To be assigned |
| CloudFront signed URLs use a policy/signature and configured trusted public keys | CloudFront Signed URLs | Official product documentation | 2026-11-14 | High | To be assigned |
| HTTP shared-cache freshness, validators, and stale response behavior are specified | RFC 9111 | IETF Standards Track RFC | 2026-11-14 | High | To be assigned |
| `stale-while-revalidate` and `stale-if-error` cache-control extensions are defined | RFC 5861 | IETF Informational RFC | 2026-11-14 | High | To be assigned |
| Cloud CDN documents cache modes and origin cache-control behavior | Google Cloud CDN Caching | Official product documentation | 2026-11-14 | High | To be assigned |
| Cloud CDN documents signed URLs, signed cookies, and private origin access options | Google Cloud CDN Authenticate Content | Official product documentation | 2026-11-14 | High | To be assigned |
