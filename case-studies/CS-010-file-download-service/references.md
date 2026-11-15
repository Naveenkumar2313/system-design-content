# CS-010 — Research References and Evidence Register

## Source documents supplied for this task

1. `Codorythm_250_Case_Studies.docx.pdf`
   - Category A: Web & Internet Systems.
   - Case 10: Design a File Download Service.
   - Establishes case-study title and catalogue scope; it does not prescribe a vendor implementation.

2. `System_Design_Knowledge_Library.docx.pdf`
   - Provides general system-design curriculum context.
   - It does not document any specific vendor's private production architecture.

## Evidence policy for interns

For any claim about a real vendor's actual implementation:
- Prefer official product documentation, standards, security guidance, and published papers.
- Record title, publisher, URL, product/version, access date, and the exact claim supported.
- Label third-party analysis as secondary evidence.
- Do not turn a common design pattern into a claim about a named organization's private architecture.
- Confirm provider-specific behavior before relying on it for range delivery, signing, cache policy, deletion, or failover.

## Technical references to consult and verify during research

- HTTP Semantics, RFC 9110 (Range Requests, conditional requests, validators, status codes): https://www.rfc-editor.org/rfc/rfc9110
- HTTP Caching, RFC 9111 (cache directives, validation, shared caches): https://www.rfc-editor.org/rfc/rfc9111
- HTTP Semantics, Range Requests section: https://www.rfc-editor.org/rfc/rfc9110.html#name-range-requests
- Amazon S3, downloading objects: https://docs.aws.amazon.com/AmazonS3/latest/userguide/download-objects.html
- Amazon S3, pre-signed URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Amazon CloudFront, cache behavior and cache policies: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-the-cache-key.html
- Amazon CloudFront, restrict access to an Amazon S3 origin: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html
- Google Cloud Storage, download objects: https://cloud.google.com/storage/docs/downloading-objects
- Google Cloud CDN, caching overview: https://cloud.google.com/cdn/docs/caching
- Azure Blob Storage, download blobs with .NET: https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-download
- Azure Front Door, caching: https://learn.microsoft.com/en-us/azure/frontdoor/front-door-caching
- OWASP Authorization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- TLS 1.3, RFC 8446: https://www.rfc-editor.org/rfc/rfc8446
- Object storage lifecycle, replication, encryption, and pricing documentation for the selected provider and region.

These sources support protocol, security, and product behavior; they do not prove a specific company's architecture, throughput, durability, cache hit ratio, or cost.

## Claims requiring external verification before publication as vendor facts

- Range support and `If-Range` behavior for the selected storage/CDN endpoint and response transformations.
- Whether a signed URL can be revoked early, and the maximum/minimum expiry or signing constraints.
- Cache key defaults, signed-cookie/URL behavior, query/header forwarding, and error caching.
- Cache purge propagation and whether existing in-flight responses can be terminated.
- Origin access identity/service-principal semantics and protection from direct-origin bypass.
- ETag and checksum semantics for multipart, encrypted, compressed, or transformed objects.
- Object consistency, versioning, replication lag, lifecycle, and deletion behavior.
- CDN range caching, request collapsing, shield, failover, and regional behavior.
- Current egress, request, storage, log, and inter-region pricing.
- Product-specific DDoS, WAF, abuse, and rate-limiting coverage.

## Evidence log

| Claim / section | Source | Source type | Date | Confidence | Reviewer |
|---|---|---|---|---|---|
| HTTP defines Range, Content-Range, partial response, and conditional range semantics | RFC 9110 | IETF Standards Track RFC | 2026-11-14 | High | To be assigned |
| HTTP cache behavior and directives are specified separately from application authorization | RFC 9111 | IETF Standards Track RFC | 2026-11-14 | High | To be assigned |
| S3 supports object download and pre-signed access mechanisms subject to permissions and URL constraints | Amazon S3 user guide | Official product documentation | 2026-11-14 | High; verify configuration | To be assigned |
| CloudFront cache behavior is configured through cache policies and cache keys | CloudFront Developer Guide | Official product documentation | 2026-11-14 | High; verify current configuration | To be assigned |
| CloudFront can restrict access to an S3 origin using its documented private-origin mechanism | CloudFront Developer Guide | Official product documentation | 2026-11-14 | High; verify current configuration | To be assigned |
| Google Cloud Storage and Cloud CDN provide documented object download and caching capabilities | Google Cloud documentation | Official product documentation | 2026-11-14 | High; confirm exact endpoint behavior | To be assigned |
| Azure Blob Storage and Azure Front Door document blob download and cache controls | Microsoft Learn | Official product documentation | 2026-11-14 | High; confirm exact endpoint behavior | To be assigned |
| Authorization should be deny-by-default and checked for each protected resource/action | OWASP Authorization Cheat Sheet | OWASP security guidance | 2026-11-14 | High | To be assigned |
