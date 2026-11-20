# CS-016 — Research References and Evidence Register

## Source documents supplied for this task
- URL Shortener reference files and previous case-study files establish the template, not vendor internals.
- The supplied CS-016 reference text establishes the scope and evidence register format.

## Evidence policy for interns
Prefer official sources and distinguish standards, draft guidance, vendor documentation and proposed design decisions. Record the supported claim, publisher, publication or update date, and access date. Do not present illustrative simulation parameters or proposed architectures as measured vendor facts.

## Technical references to consult and verify during research

The following findings were checked on 19 November 2026. Source titles and publishers are recorded in the evidence log rather than repeated as a separate list of links.

- HTTP provides standardized request and response semantics suitable for a CAPTCHA service interface.
- Interactive CAPTCHA challenges can create accessibility barriers. Alternative verification experiences should be considered in the proposed design.
- Public CAPTCHA product documentation supports server-side verification as an integration pattern. A client-side completion indication alone is insufficient to authorize a protected action.
- The proposed service uses load balancing, separate challenge and verification components, authoritative state, policy management and monitoring. These are educational design choices, not verified vendor infrastructure.

## Claims requiring external verification before publication as vendor facts

Actual vendor architecture, proprietary detection methods, production latency, traffic volume, availability and infrastructure topology are not established by the reviewed documents. No numerical vendor performance claims are asserted. Simulation workloads and architecture evolution stages are illustrative assumptions.

## Evidence log

| Claim / section | Source URL | Source type | Date | Confidence | Reviewer |
|---|---|---|---|---|---|
| HTTP request and response semantics — HTTP Semantics, IETF / RFC Editor | https://www.rfc-editor.org/rfc/rfc9110 | Internet Standard | Published June 2022; accessed 2026-11-19 | High | AI source check; human review pending |
| Interactive CAPTCHA can create accessibility barriers — Inaccessibility of CAPTCHA, W3C | https://www.w3.org/TR/turingtest/ | Group Draft Note; non-normative | Published 2021-12-16; accessed 2026-11-19 | High for documented accessibility concerns | AI source check; human review pending |
| Server-side verification is a documented integration pattern — Verifying the user's response, Google for Developers | https://developers.google.com/recaptcha/docs/verify | Official vendor documentation | Updated 2024-10-14; accessed 2026-11-19 | High for documented interface | AI source check; human review pending |
| Protected actions require backend verification rather than relying on frontend completion — Developer Guide, hCaptcha | https://docs.hcaptcha.com/ | Official vendor documentation | Publication date not stated; accessed 2026-11-19 | High for documented integration | AI source check; human review pending |
| Load balancing, separated services, authoritative state and multi-region deployment | Not applicable — proposed case-study design | Educational design assumption | 2026-11-19 | Not a vendor fact | Human design review pending |
| Capacity estimates and simulation workloads | Not applicable — illustrative model | Educational assumption | 2026-11-19 | Not measured production data | Human design review pending |
