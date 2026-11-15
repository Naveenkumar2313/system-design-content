# CS-009 — Research References and Evidence Register

## Source documents supplied for this task

1. `Codorythm_250_Case_Studies.docx.pdf`
   - Category A: Web & Internet Systems.
   - Case 9: Design a File Upload Service.
   - Establishes the case-study title and catalogue scope; it does not prescribe a vendor implementation.

2. `System_Design_Knowledge_Library.docx.pdf`
   - Provides general system-design curriculum context.
   - It does not document any specific vendor's private production architecture.

## Evidence policy for interns

For any claim about a real vendor's actual implementation:
- Prefer official product documentation, standards, security guidance, and published papers.
- Record title, publisher, URL, product/version, access date, and the exact claim supported.
- Label third-party analysis as secondary evidence.
- Do not turn a common design pattern into a claim about a named organization's private architecture.

## Technical references to consult and verify during research

- Amazon S3, multipart upload overview: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- Amazon S3, upload objects with pre-signed URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html
- Amazon S3, checking object integrity: https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity.html
- Google Cloud Storage, resumable uploads: https://cloud.google.com/storage/docs/resumable-uploads
- Azure Blob Storage, upload blobs and block blobs: https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-upload
- OWASP File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- HTTP Integrity Fields, RFC 9530: https://www.rfc-editor.org/rfc/rfc9530
- HTTP Semantics, RFC 9110: https://www.rfc-editor.org/rfc/rfc9110
- TLS 1.3, RFC 8446: https://www.rfc-editor.org/rfc/rfc8446
- Object storage lifecycle, replication, and versioning documentation for the selected provider and storage class.

These sources support protocol, security, and product behavior; they do not prove a specific company's architecture, throughput, durability, or cost.

## Claims requiring external verification before publication as vendor facts

- Provider-specific multipart minimum/maximum part sizes, part counts, object size, and upload-session lifetime.
- Whether a provider operation is idempotent and how duplicate part numbers/offsets are handled.
- ETag and checksum semantics for single-part, multipart, encrypted, and composite objects.
- Pre-signed URL expiry limits, exact signed headers, overwrite behavior, and revocation semantics.
- Resumable session URI sensitivity, expiration, and offset/status protocol.
- Object visibility/consistency, event ordering, notification delivery guarantees, and replication lag.
- Current storage durability, lifecycle, versioning, encryption, quotas, and pricing.
- Product-specific malware scanning, CDN access, and private-origin behavior.

## Evidence log

| Claim / section | Source URL | Source type | Date | Confidence | Reviewer |
|---|---|---|---|---|---|
| S3 multipart allows independent part upload/retry and requires a completion or abort workflow | Amazon S3 Multipart Upload Overview | Official product documentation | 2026-11-14 | High | To be assigned |
| S3 pre-signed upload URLs grant time-limited access within the signer's permissions and target key | Amazon S3 Pre-signed URL Upload | Official product documentation | 2026-11-14 | High | To be assigned |
| S3 supports selected checksum algorithms and server-side validation | Amazon S3 Object Integrity | Official product documentation | 2026-11-14 | High | To be assigned |
| Cloud Storage resumable upload uses a session URI and can resume after transfer interruption | Google Cloud Storage Resumable Uploads | Official product documentation | 2026-11-14 | High | To be assigned |
| Azure Blob Storage supports upload paths including block blob staging/commit through client tooling | Azure Blob Upload | Official product documentation | 2026-11-14 | High | To be assigned |
| File upload defenses include allowlists, content validation, generated names, size limits, authorization, and scanning | OWASP File Upload Cheat Sheet | OWASP security guidance | 2026-11-14 | High | To be assigned |
| HTTP defines content and representation digest fields for integrity verification | RFC 9530 | IETF Standards Track RFC | 2026-11-14 | High | To be assigned |
