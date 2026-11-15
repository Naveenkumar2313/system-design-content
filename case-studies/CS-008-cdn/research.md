# CS-008 — Design a CDN (Content Delivery Network)

> **Content status:** Educational reference design. This document proposes a generic content-delivery network; it does not claim to describe any provider's private production architecture. Workload, capacity, performance, and availability figures are illustrative assumptions.
>
> **Source alignment:** The Codorythm case-study catalogue identifies this as Case Study 8 in Category A: Web & Internet Systems. The required structure follows the 18-section case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-008 |
| System name | Content Delivery Network |
| Real-world reference | Cloudflare, Amazon CloudFront, Akamai, and Google Cloud CDN are examples of CDN providers, not asserted system designs |
| Category | Web & Internet Systems |
| Category number | A |
| Case number | 8 |
| Difficulty | Beginner to Intermediate |
| Short description | Design a globally distributed delivery service that routes client requests to edge caches, serves reusable content close to users, and retrieves or revalidates misses through regional caches and protected origins. |
| Primary users | Website owners, application teams, media publishers, end users, platform and security operators |
| Key concepts | DNS and anycast routing, edge Points of Presence, HTTP caching, cache keys, TTL, invalidation, origin shield, dynamic acceleration, multi-CDN, TLS, WAF, observability |
| Related component IDs | CMP-001 (Load Balancer) for routing and origin traffic distribution |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-14 |

### Scope and assumptions

- The design serves HTTP(S) web assets, API responses when explicitly cacheable, downloads, and segmented media. It does not design the full application or video encoding pipeline.
- An origin is the authoritative source for an object. An optional regional cache or origin shield aggregates misses before they reach that origin.
- A CDN has distributed data-plane caches and a logically separate configuration/control plane; the exact provider topology is proprietary and varies.
- DNS steering, anycast, and regional routing are alternatives or layers that may be combined; they do not guarantee that each request reaches the geographically closest machine.
- Public shared caching is used only when request and response semantics permit it. Personalized, credentialed, or private content requires a separately designed authorization and cache-key policy.
- All quantities below are illustrative planning assumptions, not disclosures or benchmarks from a CDN vendor.

---

## 01 — Story

A website hosted in one region may serve users on other continents. A request to that origin crosses multiple networks, competes for constrained paths, and consumes origin CPU, connections, and bandwidth. A traffic spike, a distant user, or an origin outage can make pages slow or unavailable even when the content is mostly static and identical for every visitor.

A Content Delivery Network (CDN) distributes cacheable copies of content to edge locations near users. A client resolves a hostname to a CDN entry point, the CDN selects an edge location, and the edge either serves a fresh cached response or retrieves it through a parent cache or from the origin. A CDN can reduce round trips and origin load, but it does not make stale content correct, make a weak origin infinitely scalable, or guarantee authorization for private objects.

### Actors

1. **Viewer/client:** Requests a web page, API response, image, document, software package, or media segment.
2. **Content owner:** Publishes content, configures cache behaviors, and defines origin and access policies.
3. **DNS resolver / routing system:** Directs the hostname toward a CDN service address or provider endpoint.
4. **CDN edge:** Terminates or forwards traffic, evaluates request policy, checks the local cache, and serves or fetches content.
5. **Regional cache / origin shield:** Optionally aggregates edge misses and protects the origin from duplicate fetches.
6. **Origin:** Holds the authoritative content and returns objects, validators, and cache policy.
7. **CDN control plane and operator:** Publishes validated configuration, purges or versions content, and monitors delivery.

### Core user journey

1. A publisher uploads `assets/site-v42.css` to an origin and gives it a long freshness lifetime because the filename is versioned.
2. The publisher configures a CDN distribution, origin, TLS certificate, cache key, and security policy.
3. A viewer requests `https://www.example.com/assets/site-v42.css`.
4. DNS and the CDN's network routing direct the request to an edge location.
5. A fresh edge entry produces a **HIT**, returning the object without an origin fetch.
6. On a **MISS**, the edge retrieves the object from a regional cache/origin shield or directly from the origin, subject to policy.
7. The edge stores an eligible response for its remaining freshness lifetime and returns it to the viewer.
8. When the asset changes, the publisher uploads a new content-hashed path; an explicit purge is reserved for cases that cannot use versioned paths.

### Problem boundaries

This case study focuses on HTTP(S) content delivery, cache behavior, routing, origin protection, and failure handling. It does not prescribe a CDN vendor's private edge topology, design a complete WAF or identity provider, or implement an entire media-processing pipeline. A CDN accelerates eligible traffic; uncached or personalized traffic still depends on application and origin performance.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Route configured hostnames to an appropriate CDN edge entry point | Must |
| FR-02 | Serve fresh cacheable objects from edge storage | Must |
| FR-03 | Fetch eligible misses through an optional parent cache/origin shield to an origin | Must |
| FR-04 | Apply per-host/path cache behaviors, cache keys, TTLs, and origin selection | Must |
| FR-05 | Support cache revalidation with validators such as ETag and Last-Modified where supported | Should |
| FR-06 | Support versioned object paths and controlled invalidation/purge workflows | Must |
| FR-07 | Support TLS for viewer connections and secure CDN-to-origin transport | Must |
| FR-08 | Support private content access controls, such as signed URLs/cookies or edge authorization | Should |
| FR-09 | Support static delivery, dynamic acceleration, and streaming/download workloads where configured | Should |
| FR-10 | Detect unhealthy edge/origin paths and route or fail over according to an explicit policy | Must |
| FR-11 | Provide origin shielding, request collapsing, or equivalent miss aggregation where available | Should |
| FR-12 | Provide metrics, logs, traces, and purge/configuration audit records | Must |
| FR-13 | Support rate controls, WAF/bot policy, and DDoS mitigation at the edge where available | Should |
| FR-14 | Support regional or multi-CDN failover when the availability requirement justifies it | Could |

### Non-functional requirements

| ID | Requirement | Initial target / policy |
|---|---|---|
| NFR-01 | Cache-hit latency | Illustrative in-region edge TTFB p95 under 50 ms, excluding viewer access network |
| NFR-02 | Availability | Illustrative 99.95% monthly for CDN hostname delivery, subject to DNS, provider, and origin dependencies |
| NFR-03 | Cache correctness | Never share personalized or authorization-dependent responses across users without a deliberate, verified policy |
| NFR-04 | Freshness | Each content class has an explicit freshness, revalidation, and stale-serving policy |
| NFR-05 | Origin protection | Bound miss concurrency and origin request rate; use shielding and request collapsing where appropriate |
| NFR-06 | Scalability | Add edge capacity and cache footprint without requiring a central per-request decision |
| NFR-07 | Security | Protect viewer and origin hops, restrict origins, validate signed access, and limit abuse |
| NFR-08 | Configuration safety | Validate and stage distribution, cache, certificate, and security changes with rollback |
| NFR-09 | Observability | Distinguish HIT, MISS, revalidation, bypass, stale serving, and origin errors |

### Out of scope for the initial version

- Authoritative DNS service design and global BGP routing implementation.
- Application business logic and user authentication provider.
- Object creation workflows, video encoding/transcoding, and media packaging.
- Guaranteeing globally instantaneous invalidation or zero propagation delay.
- Provider-specific feature limits, prices, or performance guarantees.

---

## 03 — Scale

All values in this section are **illustrative workload assumptions for teaching and simulation**. They are not vendor disclosures or CDN benchmarks.

### Workload assumptions

| Metric | Illustrative value | Reasoning |
|---|---:|---|
| Viewer requests | 100,000,000 per day | Moderate global website/download exercise |
| Average request rate | ~1,157 requests/sec | 100,000,000 / 86,400 |
| Peak request rate | 12,000 requests/sec | Approx. 10.4× average to model daily peaks |
| Average delivered object size | 32 KiB | Simplified mix of web assets and small downloads |
| Average delivered bytes | ~3.28 TB/day | 100,000,000 × 32 KiB, decimal storage units |
| Average egress rate | ~0.30 Gbit/s | 3.28 TB/day × 8 / 86,400 |
| Peak egress rate | ~3.15 Gbit/s | 12,000 × 32 KiB × 8; large-object bursts require separate sizing |
| Request cache-hit ratio objective | ≥ 90% | Illustrative objective for cacheable traffic, not all requests |
| Byte cache-hit ratio objective | ≥ 85% | Separate from request hit ratio because object sizes differ |
| Origin-bound peak request rate | ≤ 1,200 requests/sec at 90% request HIT | 10% of the assumed peak, before revalidations and bypasses |
| Content update profile | 80% versioned static; 15% revalidated; 5% dynamic/private | Simplified planning distribution |
| Retained edge object set | 5 TB per serving region, illustrative | Capacity depends on popularity, object size, and eviction |

### Capacity notes

- **Request hit ratio** = cache-hit requests / eligible viewer requests. **Byte hit ratio** = bytes served from cache / eligible bytes delivered. They are not interchangeable.
- At 12,000 peak RPS and 32 KiB average response, estimated peak egress is about 3.15 Gbit/s using 1,024-byte KiB and decimal bits/sec. This is only a mean-size model; video and large downloads create heavier tails.
- A 90% request hit ratio yields about 1,200 origin-bound requests/sec at the assumed peak only if the hit ratio applies at that time and to that request population. Revalidation, bypass, uncached APIs, and large-object byte misses add origin load.
- An origin shield may reduce duplicate edge-to-origin requests, but its cache adds another layer of freshness and failure behavior. A shield is not guaranteed to be a single shared cache across every provider or region.
- Edge storage is usually governed by a popularity-based working set and eviction policy; a configured TTL does not guarantee an object remains resident until expiry.
- Capacity planning must account for TLS handshakes, concurrent connections, HTTP/2 or HTTP/3 streams, cache metadata, request collapsing, WAF inspection, purge rate, and origin bandwidth—not only average RPS.

### Questions for a production sizing exercise

- What percentage of requests and bytes are cacheable, and how skewed is object popularity?
- What are the distributions of object sizes, freshness lifetimes, and update frequency?
- Which request headers, cookies, query parameters, and authorization tokens affect representation or access?
- What are peak concurrent viewers, new connections/sec, TLS handshakes/sec, and streaming bitrates?
- What origin request and bandwidth budgets remain during cache cold start, regional failure, or purge?
- Is the required freshness measured in seconds, minutes, or hours, and what stale data is acceptable?

---

## 04 — Naive Architecture

### Simplest viable design

A single origin web server hosts all content. DNS resolves the public hostname to that server, and every viewer request travels to the origin and back.

**Components**
- Viewer/browser/player
- DNS record
- Single origin web server and storage
- Origin monitoring

**Request flow**
1. Viewer requests a resource using the site's hostname.
2. DNS returns the origin's address.
3. Viewer connects directly to the origin.
4. Origin reads or generates the resource and returns it.
5. Origin sends the response across the network to the viewer.

### Naive architecture diagram

See `architecture.png`.

### Why begin here?

The direct-origin design is easy to understand and provides a useful correctness baseline. It exposes the distance, bandwidth, concurrency, and availability problems that a distributed cache and edge-routing layer are intended to improve.

---

## 05 — Problems

| Problem | Why it occurs | User/system impact | Likely improvement |
|---|---|---|---|
| Long-distance latency | Every request traverses the path to one origin region | Slow TTFB and page/media startup | Serve reusable objects from nearby edge caches |
| Origin bandwidth bottleneck | Every viewer downloads the same object from origin | High egress cost and saturated links | Cache popular content at edge and regional tiers |
| Origin request overload | Request rate grows with viewers even for identical assets | CPU, connection, and storage saturation | Cache hits, origin shield, and request collapsing |
| Single origin failure | One server/region owns all content delivery | Site or content becomes unavailable | Replicated origins, regional failover, and permitted stale serving |
| Traffic spikes | Launches, news, or viral content create correlated demand | Origin overload and timeouts | Edge distribution, capacity headroom, shielding, and load shedding |
| Cache-key mistakes | User-specific headers, cookies, or query strings are ignored or mishandled | Wrong representation or private data leakage | Define and test cache-key and authorization policy |
| Stale content | TTL or purge policy does not match publication behavior | Users see old or revoked content | Versioned URLs, validators, explicit invalidation, and freshness design |
| Cache stampede | Many simultaneous misses/refetches target the same popular object | Origin sees a burst despite high average hit ratio | Request collapsing, locks, stale-while-revalidate, and shield |
| Cold or sparse edge cache | Long-tail content is not resident at the selected PoP | Miss latency and origin traffic rise | Regional parent cache, selective pre-warming, and origin capacity |
| DNS/routing convergence | DNS TTLs, resolver behavior, BGP changes, or client reuse delay steering changes | Some clients keep using a degraded path | Health-aware routing, low-risk failover, and measured convergence |
| Origin bypass | Public origin can be reached directly by clients | Attackers bypass edge WAF, rate controls, and cache | Restrict origin to CDN paths/identity and monitor origin access |
| Purge storms | Broad invalidations trigger many simultaneous misses | Origin spike and temporary cache coldness | Versioned filenames, scoped purge, rate controls, and warm-up |

---

## 06 — Architecture Evolution

These are proposed learning stages, not a historical account of any named CDN provider's architecture.

### V1 — Direct origin

- Publish content on one origin and direct the hostname to it.
- Benefit: minimal infrastructure and straightforward freshness.
- Limitation: origin is geographically distant for many users and handles every request.

### V2 — Pull CDN with edge cache

- Configure a CDN hostname, TLS, origin, and cache behaviors.
- The edge fetches eligible content on a cache miss and stores it according to response policy.
- Benefit: repeat requests can be served locally and origin load is reduced.
- New concern: cache key, TTL, authorization, purge, and cold-cache behavior must be correct.

### V3 — Regional cache and origin shield

- Add a parent/Regional cache or provider origin shield between edge PoPs and origin.
- Coalesce or collapse concurrent misses for the same cache key when supported.
- Benefit: fewer duplicate long-haul origin fetches and improved origin protection.
- New concern: another cache tier adds latency on misses, freshness complexity, and a potential regional failure domain.
- The evolved architecture is shown in `architecture-evolved.png`.

### V4 — Content-aware caching, security, and operations

- Classify immutable static, revalidated, streaming, dynamic, and private content separately.
- Apply cache keys and TTLs per behavior; use versioned paths for frequently updated assets.
- Add edge TLS, origin TLS, origin access restriction, WAF/rate controls, signed access, and bot policies.
- Publish configuration through validation, canary, propagation monitoring, and rollback.
- Emit HIT/MISS/revalidation/bypass/stale and origin latency metrics.
- New concern: rules can interact; a cache/security misconfiguration can have global impact.

### V5 — Global redundancy and multi-CDN

- Use DNS, anycast, or provider global routing to direct users toward healthy CDN infrastructure.
- For higher availability or commercial resilience, configure a second CDN and a tested steering/failover policy.
- Maintain compatible origins, certificates, cache keys, signed-access validation, and purge workflows across providers.
- Benefit: provider, region, or network diversity.
- New concern: cache cold starts, DNS/client stickiness, duplicate egress, policy drift, and more difficult incident attribution.

### CDN architecture and cache hierarchy

| Layer | Responsibility | Main trade-off |
|---|---|---|
| DNS / global routing | Maps hostname to CDN endpoint, region, or provider according to policy | Resolver location and TTL may not reflect the individual viewer; convergence is not instant |
| Edge PoP | Terminates viewer connection, applies policy, checks local cache, and serves hits | Fast for resident objects; finite storage and a miss may need a longer path |
| Regional cache / parent tier | Aggregates misses from a set of edge locations | Reduces origin fetch duplication but adds a hop and its own cache lifecycle |
| Origin shield | Provider-specific aggregation point near or in front of origin | Can protect origin; availability and exact scope depend on provider product |
| Origin | Authoritative store or application for the content | Still required for misses, bypasses, revalidation, and dynamic content |
| Control plane | Distributes hostname, cache, security, certificate, and origin configuration | Must propagate safely and remain separate from per-request availability where possible |

### Pull versus push distribution

| Pattern | Content placement | Strength | Limitation |
|---|---|---|---|
| Pull CDN | Edge fetches an object on first request and caches it | Simple publishing workflow; only requested objects need to be fetched | First request is a miss; simultaneous misses can overload origin without collapsing |
| Push CDN | Publisher uploads content into CDN storage or explicitly prepositions objects | Predictable availability for planned launches and large media catalogs | Requires upload/replication workflow, storage management, and update coordination |

Many systems combine the models: push critical or large assets, and pull the long tail. "Push" does not necessarily mean every edge is synchronously populated.

### CDN service types

| Type | Primary workload | Design considerations |
|---|---|---|
| Static-content CDN | Images, fonts, CSS/JS, documents, and versioned downloads | Long TTLs and content-hashed URLs work well when bytes are immutable |
| Dynamic content acceleration | Uncacheable APIs and personalized requests | May optimize network path, connection reuse, and TLS; does not eliminate application/origin processing |
| Video-streaming CDN | Progressive downloads and HLS/DASH segments/manifests | Segment caching, range support, bitrate variants, live freshness, and burst/concurrency planning |
| Public CDN | Provider-operated shared delivery network, commonly reached over the public Internet and used by multiple tenants | Shared footprint and scale; enforce tenant isolation and note that private objects still require authorization |
| Private CDN | Dedicated or self-managed delivery network, or private-network CDN service for controlled enterprise delivery | Access control, private origin reachability, and operational ownership are central |
| Multi-CDN | Delivery across two or more CDN providers | Provider diversity and steering resilience in exchange for policy parity, cold caches, cost, and operations |

These are workload and deployment categories, not mutually exclusive products. A single distribution may serve static assets and dynamic requests using different behaviors.

### Routing choices

| Mechanism | How it helps | Important limitation |
|---|---|---|
| DNS-based routing | Returns a provider/region endpoint based on health, policy, geography, or measured latency | Recursive resolver location can differ from viewer location; DNS caches and clients retain answers |
| Anycast | Announces a shared IP from multiple sites; network routing directs packets toward a selected announcement | BGP selects a network path, not necessarily the physically nearest or least-loaded PoP; route changes can move flows |
| Geographic routing | Maps an observed resolver/client location to a region | Geolocation is approximate and can be affected by VPNs, mobile networks, and resolver placement |
| Latency-based routing | Chooses an endpoint estimated to provide lower network latency | Measurements can be stale and do not account for every client route or application health |
| Intelligent request routing | Combines availability, capacity, path quality, protocol, and policy | Requires reliable telemetry and guardrails; can oscillate or make opaque decisions |

DNS and anycast can be layered: DNS selects a CDN service or address family/provider, while BGP anycast selects a network attachment point. Exact mechanisms differ by vendor and deployment.

### Practical implementation patterns

#### NGINX edge-cache illustration

The following is a **single-node proxy-cache example**, not a complete global CDN. It demonstrates origin shielding, caching only safe methods, cache locking, revalidation, and stale behavior. Validate directive availability and policy against the deployed NGINX version and modules.

```nginx
http {
    proxy_cache_path /var/cache/nginx
        levels=1:2
        keys_zone=edge_cache:100m
        max_size=20g
        inactive=7d
        use_temp_path=off;

    upstream origin_shield {
        server shield.internal.example:443;
        keepalive 64;
    }

    server {
        listen 443 ssl;
        server_name cdn.example.com;
        ssl_certificate /etc/nginx/tls/cdn.example.com.crt;
        ssl_certificate_key /etc/nginx/tls/cdn.example.com.key;

        location /assets/ {
            proxy_pass https://origin_shield;
            proxy_set_header Host origin.example.com;
            proxy_ssl_server_name on;
            proxy_ssl_name origin.example.com;
            proxy_ssl_verify on;
            proxy_ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
            proxy_ssl_verify_depth 2;

            proxy_cache edge_cache;
            proxy_cache_methods GET HEAD;
            proxy_cache_key "$scheme$proxy_host$request_uri";
            proxy_cache_lock on;
            proxy_cache_revalidate on;
            proxy_cache_background_update on;
            proxy_cache_use_stale error timeout updating
                http_500 http_502 http_503 http_504;
            proxy_cache_valid 200 301 302 10m;
            proxy_cache_bypass $http_authorization $cookie_session;
            proxy_no_cache $http_authorization $cookie_session
                $upstream_http_set_cookie;
            add_header X-Cache-Status $upstream_cache_status always;
        }
    }
}
```

The example assumes the origin shield is trusted and the chosen cache key is safe for these assets. Do not copy it unchanged for user-specific responses, authorization-bearing requests, cookies, or untrusted query parameters. Stale serving and status-based caching must match the content's correctness and security requirements.

#### CloudFront-style provider workflow

1. Store the authoritative objects in a private origin where supported.
2. Create a distribution with an origin, viewer TLS certificate, and path-based cache behaviors.
3. Use a managed or custom cache policy that includes only query strings, cookies, and headers that actually vary the representation.
4. Configure origin access controls so clients cannot bypass the distribution to read private objects.
5. Set origin `Cache-Control` metadata or distribution TTL bounds by content class; use content-hashed filenames for immutable assets.
6. Use signed URLs/cookies or an authorization layer for private objects; ensure cache policy does not leak one viewer's response to another.
7. Deploy changes gradually and monitor cache status, origin errors, and invalidation results.

Example operational commands are provider-specific; review current CLI syntax and permissions before use:

```text
aws cloudfront create-invalidation \
  --distribution-id EXAMPLE123 \
  --paths "/assets/legacy.css"
```

Prefer publishing a new content-hashed object path over broad invalidation when the application can update references. An invalidation affects CDN cache state, not copies already stored in browser or intermediary caches.

### Cache policy examples

| Content class | Example response policy | Design notes |
|---|---|---|
| Versioned CSS/JS/image | `Cache-Control: public, max-age=31536000, immutable` | Safe only when the URL changes whenever the bytes change |
| Frequently updated public HTML | `Cache-Control: public, max-age=60, stale-while-revalidate=30` | Verify shared-cache and provider support; short freshness reduces but does not eliminate propagation delay |
| API response varying by language | `Cache-Control: public, s-maxage=60` and a controlled `Vary: Accept-Language` policy | Ensure the selected request header is part of the effective cache key |
| Personalized account response | `Cache-Control: private, no-store` | Do not put user-specific body data in a shared CDN cache |
| Public response with validator | `ETag: "v42"` and a suitable freshness policy | A stale entry can be conditionally revalidated; a 304 avoids retransmitting the body |

### Cache outcomes, validators, and eviction

| Mechanism | Meaning | Example / consequence |
|---|---|---|
| Cache HIT | A fresh eligible stored representation satisfies the request | Edge returns object without an origin body fetch |
| Cache MISS | No usable entry exists at the current tier | Fetch from parent/origin; store only if response and policy allow |
| TTL / expiration | Freshness lifetime after which a stored response is stale | `Cache-Control: public, max-age=60, s-maxage=300` gives browser and shared caches different freshness guidance; provider overrides and minimum/maximum TTL policies can change effective behavior |
| Cache-Control | Response directives control shared/private cacheability and freshness | `public`, `private`, `no-store`, `no-cache`, `max-age`, and `s-maxage` have different semantics; follow RFC 9111 and validate provider-specific overrides |
| ETag / If-None-Match | Entity validator for conditional revalidation | Edge sends `If-None-Match: "v42"`; origin can answer `304 Not Modified` if unchanged |
| Last-Modified / If-Modified-Since | Timestamp validator for conditional revalidation | Useful when ETag is unavailable; timestamp granularity may be less precise |
| Cache revalidation | Checks a stale entry with the origin before reuse | A 304 refreshes metadata and reuses body; a changed object returns a replacement representation |
| Purge / invalidation | Removes or marks matching CDN entries stale before natural expiry | Scope and propagation are provider-specific; a purge can trigger a refill spike and does not recall browser copies |
| `stale-while-revalidate` | Permits a cache to serve eligible stale content while refreshing in background | Reduces waiting on a refresh, but stale content is temporarily visible |
| `stale-if-error` | Permits a cache to serve eligible stale content during specified errors | Can preserve availability, but must be excluded for revoked/private or freshness-critical content |
| Eviction | Removes an object to reclaim finite cache capacity | LRU, LFU, size/popularity-aware, or provider-specific policies may evict an object before its TTL; TTL is not a storage reservation |

For example, `Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=30, stale-if-error=3600` expresses policy inputs, not a guarantee that every CDN implements identical precedence or stale behavior. Verify the selected CDN's cache policy, minimum/maximum TTL, origin-header handling, and authorization behavior.

### Response cache lifecycle

1. The CDN derives a cache key from the configured scheme/host/path and selected query/header dimensions.
2. A fresh eligible stored response is returned without contacting origin.
3. A stale response may be synchronously revalidated with `If-None-Match` or `If-Modified-Since`.
4. Origin returns `304 Not Modified` to update metadata while reusing the cached body, or a new representation such as `200 OK`.
5. An explicit purge removes matching cache entries according to provider propagation behavior; versioned paths avoid dependence on purge timing.
6. Eviction may remove an object before its TTL ends to free storage. Expiration controls freshness, not guaranteed residency.

### Real-world applications

| Use case | CDN role | Important design point |
|---|---|---|
| E-commerce | Deliver product images, styles, scripts, and public catalog content | Do not cache cart, account, or price responses across users unless policy explicitly allows it |
| Video streaming | Deliver HLS/DASH manifests and media segments | Plan for concurrent bitrate traffic, segment cacheability, range behavior, and live freshness |
| Social media | Deliver public profile assets, thumbnails, and popular media | Authorization and rapidly changing/user-specific feeds need careful separation from reusable objects |
| Online gaming | Deliver patches, launchers, maps, and static assets | Large downloads require byte-throughput planning, resumable ranges, and origin protection |
| News websites | Cache articles, images, fonts, and common page fragments | Versioning or short freshness plus revalidation balances breaking news and origin load |
| Software distribution | Serve installers, package files, and container artifacts | Integrity hashes/signatures and immutable versioned paths protect correctness |
| Image and document delivery | Cache public images, PDFs, and generated variants | Normalize variant parameters and avoid cache-key explosion or cross-tenant leakage |
| Large-scale web applications | Accelerate static assets and selected public APIs across regions | Segment cacheable and dynamic/private behaviors; monitor cache and origin separately |

Cloudflare, Amazon CloudFront, Akamai, and Google Cloud CDN are examples of public CDN products. Their footprints, pricing, cache semantics, routing, security features, and limits differ and should be compared using current official documentation and workload tests.

---

## 07 — Components

| Component ID | Component | Responsibility | Required? |
|---|---|---|---|
| CMP-001 | Load Balancer | A reusable traffic-distribution component that may serve origins or CDN management/data services | Workload-dependent |
| TBD | Viewer / Client | Requests a resource and consumes the returned representation or stream | Yes |
| TBD | DNS / Global Traffic Steering | Directs the hostname to a CDN endpoint/provider/region | Yes |
| TBD | CDN Edge PoP | Terminates viewer traffic, applies policy, checks and serves edge cache | Yes |
| TBD | Edge Cache | Stores reusable response objects and metadata subject to freshness/eviction | Yes |
| TBD | Regional Cache / Parent | Aggregates edge misses and reduces origin request fan-out | Recommended at scale |
| TBD | Origin Shield | Protects origin and consolidates fetches where the provider supports it | Optional |
| TBD | Origin / Object Store / Application | Authoritative source for content and uncached responses | Yes |
| TBD | CDN Control Plane | Publishes host, route, cache, security, and certificate configuration | Yes |
| TBD | Purge / Versioning Workflow | Changes content visibility and coordinates cache invalidation | Yes |
| TBD | TLS / Key / Signing Service | Manages certificates, private keys, signed URLs/cookies, and origin trust | Yes |
| TBD | WAF / DDoS / Bot Controls | Filters malicious or abusive traffic before it reaches protected services | Recommended |
| TBD | Monitoring / Log Pipeline | Collects cache status, latency, throughput, origin, security, and audit telemetry | Yes |
| TBD | Multi-CDN Steering | Selects between independent providers or regions | Optional |

**Component ID policy:** CMP-001 is the only confirmed reusable component relevant to this case study in the starter package. Other IDs remain TBD until component specifications are created and added to the master component index. The CDN cache is not the authoritative source of data; origin storage owns the definitive object.

### Component interactions

- Viewer → DNS/global steering → selected CDN edge PoP.
- Edge PoP → policy and cache key evaluation → local cache HIT or eligible MISS.
- Edge MISS → regional cache/origin shield → origin, with responses flowing back through the same hierarchy.
- CDN control plane → staged distribution of cache, origin, TLS, and security configuration.
- Publisher → origin object/version publication → optional purge or pre-warm workflow.
- WAF/DDoS controls → edge request path before cache/origin access according to product policy.
- Edge and parent caches → monitoring/logging for HIT, MISS, revalidation, bypass, stale, and error outcomes.
- Multi-CDN steering → provider endpoint selection; each provider still executes its own cache and origin flow.

---

## 08 — Data Flow

### Flow A: Client request and cache HIT

1. Client requests a URL such as `https://cdn.example.com/assets/logo-v42.png`.
2. DNS resolves the hostname to the CDN service; routing selects an edge PoP.
3. Edge terminates or passes through TLS according to configuration and validates request policy.
4. Edge constructs the effective cache key and checks local cache.
5. A fresh, eligible entry is returned as a HIT with appropriate response headers.
6. Edge records cache status and latency; no origin request is required for the object body.

### Flow B: Cache MISS and origin response

1. Edge receives a request and finds no fresh usable local entry.
2. Edge checks an optional regional cache/origin shield; that tier may also HIT or MISS.
3. On a hierarchy MISS, CDN forwards a permitted request to the configured origin over the protected origin path.
4. Origin returns a representation and cache metadata such as `Cache-Control`, `ETag`, and `Last-Modified`.
5. CDN verifies response cacheability and stores the object under the configured key for the effective freshness lifetime.
6. Response travels back through parent/edge to viewer; logs mark the result as MISS or the provider's corresponding status.

### Flow C: Stale object revalidation

1. Edge finds a stored but stale object.
2. Edge sends a conditional request with `If-None-Match` or `If-Modified-Since` if it has a validator.
3. Origin returns `304 Not Modified`; CDN refreshes the stored metadata and reuses the body.
4. If the object changed, origin returns the new representation and validators.
5. CDN stores the replacement if eligible and returns it to the viewer.
6. If policy allows stale-while-revalidate, an edge may serve stale content while a background revalidation runs; this is not appropriate for every object or directive combination.

### Flow D: Content publication and purge

1. Publisher writes a new object version to origin.
2. For immutable assets, publisher changes references to a content-versioned URL.
3. For mutable URLs, an authorized operator submits a scoped invalidation/purge.
4. CDN control plane validates authorization and distributes purge state.
5. Requests after propagation fetch the current version; already downloaded browser copies and in-flight responses are not recalled.
6. Monitoring checks purge completion, miss rate, origin load, and content correctness.

### Flow E: Private/signed content

1. Application authenticates the user and authorizes access to a resource.
2. Application issues a time-limited signed URL/cookie or an edge-verifiable token.
3. CDN validates signature, resource scope, expiry, and relevant policy before serving content.
4. Cache configuration ensures authorization is checked on every request and that one user's restricted representation cannot satisfy another user's request.
5. CDN serves eligible shared bytes or fetches from a private origin using restricted origin credentials.
6. Expiration, key rotation, revocation, and token leakage are monitored and handled by explicit policy.

### Flow F: Global routing and Multi-CDN

1. Client or recursive resolver resolves the customer hostname.
2. DNS steering or anycast/routing delivers the connection to a provider endpoint or PoP.
3. Provider edge selects a local/parent/origin path using its own health and capacity state.
4. If the provider or region is unhealthy, global steering changes endpoint selection.
5. DNS and client connection reuse may delay adoption; a second provider may have a cold cache.
6. Each provider continues to enforce matching TLS, cache, authorization, purge, and origin protection policy.

### Error and edge-case paths

- Cache bypass: forward only under explicitly configured method, authorization, cookie, or request policy.
- Cache key collision: treat as a security/correctness incident; disable the faulty behavior, purge affected objects, and assess exposure.
- No origin or parent available: serve stale only if policy permits and content is safe; otherwise return a controlled error.
- Conditional origin fetch times out: follow stale-if-error policy if applicable; emit origin timeout and stale-serving metrics.
- Purge delayed: publish versioned URLs and communicate that browser/intermediary copies may persist.
- Signed URL expired or invalid: reject before delivering restricted content, including when a body is cached.
- DNS/anycast route changes: account for cached DNS answers, established connections, and route convergence.
- Origin failure: stop or limit retries, keep safe stale content available when allowed, and alert on origin health.
- Partial object/range request: apply range and cache semantics consistently; do not combine incompatible byte ranges.

---

## 09 — Failure Scenarios

| Failure | Detection | Immediate effect | Mitigation / recovery |
|---|---|---|---|
| Edge server or PoP fails | Provider probes, connection resets, synthetic checks, regional error/latency | Some clients lose a path or see elevated latency | Withdraw unhealthy route, steer to another PoP, and keep regional/provider headroom |
| Regional cache fails | Parent health and origin-fetch telemetry | More edge misses go farther upstream; origin load rises | Fail around to another tier or origin, protect origin with concurrency limits, and recover cache warm-up |
| Origin application/storage fails | Health checks, timeout/5xx rate, provider origin metrics | Misses and revalidations fail; fresh hits may continue | Serve permitted stale responses, use replicated origins, circuit-break retries, and restore origin |
| Edge cache disk/metadata failure | Cache process health, eviction/write errors, rising miss ratio | Content becomes cold or cache bypasses | Replace/rebuild cache, route around node, and throttle origin refetches |
| DNS provider or resolver issue | External synthetic resolution from multiple networks | New clients may fail or use old endpoint | Redundant authoritative DNS, tested failover records, sensible TTLs, and cached-answer awareness |
| BGP/anycast path issue | Route telemetry, regional probes, packet loss | Some networks reach a poor or unreachable PoP | Withdraw route where policy permits, use alternate address/provider, and coordinate network recovery |
| Cache stampede | Sudden origin fetch concurrency for one object | Origin overload despite normal average request rate | Request collapsing, per-key locks, stale-while-revalidate, shield, jittered TTLs, and origin admission control |
| Traffic spike or cache cold start | RPS, bytes/sec, active connections, origin utilization | Edge/parent capacity or origin is saturated | DDoS controls, pre-warm selected content, capacity headroom, bounded retries, and graceful degradation |
| Incorrect broad purge | Purge audit, cache MISS wave, origin load | Many PoPs refill simultaneously | Require scoped permissions/approvals, rate-limit purge, use versioned names, and protect origin during refill |
| Incorrect cache key or rule | HIT content differs by user/variant; security reports | Wrong language/representation or data leakage | Disable caching/rule, purge, audit affected requests, add cache-key tests, and restore safe defaults |
| TLS certificate/key failure | Certificate expiry and handshake metrics | Viewer or origin TLS fails | Automated renewal, staged certificate rollout, private-key protection, and rollback |
| WAF/rate-limit false positive | Block/challenge rates and support reports | Legitimate viewers cannot access content | Version policy, monitor sampled decisions, provide rollback, and isolate emergency changes |
| Multi-CDN steering failure | Provider synthetic probes and endpoint-level error rates | Traffic remains on degraded CDN or fails over poorly | Independent monitoring and DNS control, conservative health thresholds, and tested alternate origin |
| Stale content served beyond policy | Cache-age and freshness telemetry, content validation | User sees outdated or revoked content | Tighten stale policy, purge/version assets, and exclude revocable/private content from stale serving |
| Network partition between CDN and origin | Connect timeout and origin reachability probes | Cache misses fail while hits may work | Serve allowed stale content, fail over to replicated origin, and avoid retry amplification |

Never assume a CDN provider's fail-open behavior, DNS timing, purge propagation, or stale semantics without validating current product documentation and configuration.

---

## 10 — Scaling

### Edge and cache scaling

- **Geographic distribution:** Add PoPs or use provider edge footprints near demand. Placement is an economic and network-planning decision, not simply "one cache per city."
- **Horizontal scale:** Add edge servers, network capacity, cache storage, and request-processing capacity behind provider routing. Keep request decisions local where possible.
- **Distributed caching:** Use edge and optional parent/regional caches. Cache placement should match popularity, object size, freshness, and origin cost.
- **Eviction:** Use a practical replacement policy (often popularity/recency/cost aware). TTL defines freshness, but eviction may remove an unexpired object when space is needed.
- **Cache warming:** Pre-fetch only high-value objects for known launches or live events; indiscriminate pre-warming consumes bandwidth and may not reach every PoP.
- **Request collapsing:** Combine concurrent misses for the same cache key where supported to avoid a thundering herd.

### Origin scaling and protection

- **Origin replication:** Place authoritative or read replicas across regions when consistency and publication flow permit.
- **Origin shield:** Aggregate edge misses before origin; shard or regionalize shields to avoid creating a single hot point.
- **Capacity budget:** Size origin for cache misses, revalidation, bypass/private traffic, purge/refill, and an agreed cache-hit degradation scenario.
- **Backpressure:** Bound concurrent origin fetches, timeouts, retries, and queues. Reject or serve safe stale content rather than building unbounded queues.
- **Publication:** Use immutable, versioned objects to shift cache refresh from global invalidation to ordinary new-key retrieval.

### Availability and global growth

- **Regional redundancy:** Ensure alternate PoPs and parent/origin paths can absorb traffic when one site fails.
- **Multi-region origin:** Test consistency, failover, and object replication; a healthy CDN edge cannot fetch missing data from an unavailable origin.
- **Multi-CDN:** Use when independent provider diversity justifies complexity. Align certificates, cache policy, signed access, origin security, logs, and invalidation.
- **DNS failover:** Monitor from viewer-like networks and account for recursive caches, low TTL behavior, clients pinning addresses, and existing connections.
- **Anycast:** Announce routes from multiple sites and withdraw degraded paths according to network health; BGP convergence and path selection are not instantaneous.
- **Autoscaling:** Add capacity using measured bytes/sec, new connections, cache CPU, storage pressure, and expected warm-up—not RPS alone.

### Capacity measures

Track separately:
- Requests/sec and concurrent requests/streams.
- New TCP/TLS connections and handshake rate.
- Ingress and egress bits/sec, including peak and percentile object sizes.
- Cache storage, index/metadata pressure, eviction, and write rate.
- Cache hit by requests and bytes, separated by content class and region.
- Origin request rate, concurrency, bandwidth, and error budget.
- Purge and configuration propagation times.
- Capacity remaining after losing a PoP, region, shield, or CDN provider.

Scaling a CDN can shift bottlenecks: higher edge capacity can expose origin limitations during cache cold start, and more global coverage can increase TLS, logging, policy, and invalidation complexity.

---

## 11 — Trade-offs

### Caching and freshness

| Strategy | Strength | Limitation |
|---|---|---|
| Long TTL + versioned URL | High hit ratio and low origin load; deterministic new version selection | Publishing must update references; old versions consume cache/storage until eviction |
| Short TTL | Bounds normal staleness without frequent explicit purge | More revalidation/origin traffic and potentially higher latency |
| Conditional revalidation | Validator can avoid retransmitting an unchanged body | Still requires an origin round trip; origin must implement validators correctly |
| Explicit purge/invalidation | Removes selected CDN objects before their TTL | Propagation is not an instantaneous guarantee across all client/browser caches; broad purge can create refill load |
| Stale-while-revalidate | Reduces wait and stampede on expiry for eligible objects | Serves stale content during refresh; not suitable for revocable or strongly fresh data |
| Stale-if-error | Preserves availability during selected origin failures | Potentially outdated content; define bounded windows and content safety |

### CDN deployment options

| Choice | Option A | Option B | Considerations |
|---|---|---|---|
| Distribution model | Pull | Push | First-request miss simplicity vs. upload/pre-position complexity |
| Cache hierarchy | Edge only | Edge + regional parent/shield | Fewer layers vs. origin protection and aggregated fill |
| Routing | DNS steering | Anycast | Policy/geography flexibility vs. network path selection and convergence |
| Provider model | Public/shared provider | Private/self-managed CDN | Operational reach/features vs. control, cost, and staffing burden |
| Providers | Single CDN | Multi-CDN | Simpler policy and warm cache vs. provider diversity and coordination |
| Freshness | TTL/revalidation | Explicit purge | Routine simplicity vs. immediate-ish control (with propagation caveats) |
| Dynamic content | Direct origin | Dynamic acceleration through CDN | Fewer network hops vs. routing/TLS/security and provider dependency |

### Cache key and security decisions

The cache key must represent every input that changes the response. A key that is too broad can leak data; one that is too specific fragments the cache and lowers hit ratio. Include only required query parameters, cookies, and headers, normalize carefully, and test with adversarial variants. Authorization should be checked before serving protected cached content; merely adding a token to a cache key is not a substitute for validating it.

### Required decision record format

For every major decision, interns should document:
- Context and problem.
- Options considered.
- Chosen option.
- Why it was selected.
- What is gained.
- What is sacrificed.
- Conditions that would justify revisiting it.

---

## 12 — Cost

No provider, region, traffic commitment, transfer pricing, or current quote is specified. Therefore, this section provides a cost model rather than a fabricated currency estimate.

### Cost categories

| Cost item | Main cost driver | Measurement needed |
|---|---|---|
| CDN requests | Request count, method, and feature tier | Requests by region, status, cache outcome, and product |
| Data transfer | Bytes delivered to viewers and between tiers/regions | Egress bytes by geography, protocol, and object class |
| Origin traffic | Miss, revalidation, bypass, and fill bytes | Origin requests/sec, bytes, and cross-region transfer |
| Origin shield / regional cache | Requests, bytes, compute/storage, or service tier | Parent hit ratio and refill volume |
| Purge/invalidation | Number of paths/tags and provider pricing policy | Purge frequency, breadth, and API operations |
| WAF / DDoS / bot controls | Request inspection, rule tier, and attack volume | Inspected requests, blocked traffic, and feature charges |
| TLS / certificates | Handshakes, managed certificates, key services | New TLS connections, renewals, HSM/KMS usage |
| Logging and analytics | Log volume, retention, queries, cardinality | Events/sec, sampled fields, retention, and export cost |
| Multi-CDN steering | DNS queries, health probes, and duplicated provider minimums | Query volume, probe regions, provider commitments |
| Origin storage and replication | Object size, replication, and retention | Stored bytes, versions, replicas, and lifecycle policy |

### Estimation worksheet

Populate using selected providers, regions, and current pricing:

- Viewer delivery cost = request charges + egress bytes × applicable regional rate.
- Origin cost = cache-miss/revalidation/bypass bytes × origin egress and request pricing.
- Cache-tier cost = shield/parent service charges or self-managed compute, storage, and network.
- Security cost = WAF/DDoS/bot inspection tiers and request volume.
- Operations cost = logs, metrics, purge automation, certificates, and on-call staffing.
- Multi-CDN cost = redundant base commitments + measured traffic split + steering and testing.

Document pricing date, geography, currency, traffic assumptions, byte/request hit ratios, origin retry/refill load, negotiated discounts, and peak capacity reserves. A high request-hit ratio alone does not establish a low bill; large-object byte misses and inter-region transfers may dominate.

---

## 13 — Security

### Threats and controls

| Threat | Mitigation |
|---|---|
| DDoS / volumetric flood | Provider-scale scrubbing, network ACLs, origin isolation, rate controls, and tested emergency policy |
| Cache poisoning | Normalize and validate inputs; include representation-affecting fields in the cache key; reject ambiguous host/header/path forms |
| Cross-user cache leakage | Do not cache personalized responses publicly; enforce authorization before cache delivery and separate cache keys/policies |
| Origin bypass | Restrict origin network access to CDN egress/identity where possible; use origin access control, private connectivity, mTLS, or secret headers |
| Credential/token leakage | Use short-lived signed URLs/cookies, protect logs/referrers, scope resources, and rotate signing keys |
| URL signature replay | Bind signature to resource, expiry, and optional constraints; use short validity and revocation strategy |
| TLS downgrade or expired certificate | Enforce HTTPS and modern TLS policy; automate certificate deployment and expiry alerts |
| CDN-to-origin interception | Use authenticated TLS to origin; validate certificate identity and avoid insecure plaintext paths |
| WAF false negatives/positives | Stage managed/custom rules, monitor decisions, tune by endpoint, and retain rapid rollback |
| Bot scraping / abuse | Use rate limits, bot signals, challenge/allow policies, and origin protection without treating IP as reliable user identity |
| Purge/config compromise | Least privilege, MFA, approval for broad purges, audit log, API token scope, and configuration rollback |
| Sensitive logs and URL parameters | Redact tokens/cookies, limit retention/access, and avoid secrets in query strings where feasible |
| Malicious or oversized object | Validate publication, cap request/response sizes, scan uploads at origin workflow, and isolate content types |
| Tenant/configuration leakage | Separate distributions, origin credentials, keys, logs, and administrative roles by tenant |

### Validation rules

- Use HTTPS for viewer connections and authenticated TLS to origins where required.
- Make the origin unavailable to arbitrary public clients when CDN controls are a required security boundary.
- Cache only responses whose authorization and representation semantics are explicitly understood.
- Test that Authorization, Cookie, Host, query parameters, `Vary`, and range requests cannot cause cross-user or cross-variant reuse.
- Do not assume a signed URL is private if its signature is logged, shared, or valid for too long.
- Verify invalidation and purge permissions; a purge is an operationally sensitive action.
- Establish WAF/rate-limit policies that protect origin while avoiding uncontrolled blocking of legitimate traffic.
- Redact personal data and secret tokens from cache logs and analytics exports.

---

## 14 — Observability

### Metrics

| Metric | Type | Why it matters |
|---|---|---|
| `cdn_viewer_requests_total` | Counter | Total demand, method, response code, and content class |
| `cdn_cache_requests_total{status}` | Counter | Separates HIT, MISS, revalidated, bypass, stale, and error outcomes |
| `cdn_cache_hit_ratio_requests` | Gauge / derived | Measures request reuse; must be scoped to eligible traffic |
| `cdn_cache_hit_ratio_bytes` | Gauge / derived | Measures byte offload; large objects may dominate bandwidth |
| `cdn_viewer_ttfb_ms` | Histogram | Client-facing first-byte latency, segmented by cache result and region |
| `cdn_edge_response_duration_ms` | Histogram | Edge processing and response behavior |
| `cdn_origin_fetch_duration_ms` | Histogram | Origin-path latency and dependency health |
| `cdn_origin_requests_total` | Counter | Origin offload, revalidation, and bypass load |
| `cdn_origin_bytes_total` | Counter | Origin bandwidth and cache-fill cost |
| `cdn_viewer_bytes_total` | Counter | Delivered throughput and egress |
| `cdn_origin_errors_total` | Counter | Miss/revalidation failures, timeouts, and 5xx responses |
| `cdn_active_connections` | Gauge | Edge resource pressure |
| `cdn_edge_storage_bytes` / `cdn_evictions_total` | Gauge / counter | Cache capacity and churn |
| `cdn_purge_propagation_seconds` | Histogram | Measures control-plane purge completion |
| `cdn_tls_handshake_errors_total` | Counter | Certificate, protocol, or cipher issues |
| `cdn_security_actions_total{action}` | Counter | WAF, rate-limit, bot, and access policy outcomes |
| `cdn_availability_synthetic` | Gauge | End-to-end viewer perspective across networks and regions |

### Derived indicators and alerts

- Request hit ratio = HIT requests / eligible viewer requests; explicitly exclude bypassed or uncacheable traffic from the denominator where appropriate.
- Byte hit ratio = bytes served from cache / eligible viewer bytes.
- Origin offload = 1 − origin-bound eligible requests / eligible viewer requests; also compute by bytes.
- MISS ratio increase with origin latency increase may indicate cold cache, purge, key fragmentation, or origin degradation.
- High hit ratio with rising TTFB may indicate edge/network saturation, TLS processing, or viewer path problems.
- Alert on origin error rate, cache status shifts, regional availability, storage/eviction pressure, purge delay, certificate expiry, and unusual security action spikes.

### Logging and tracing

- Log request ID, distribution/host, normalized path class, region/PoP, cache status, origin status/latency, bytes, TLS version, and policy outcome.
- Redact cookies, Authorization, signed tokens, and sensitive query parameters.
- Sample high-volume HITs while retaining sufficient MISS/error/stale detail for diagnosis.
- Propagate a trace/correlation ID to origins without exposing secrets or changing cache semantics.
- Keep origin and CDN clocks/identifiers correlated to distinguish viewer, edge, shield, and origin latency.

---

## 15 — Simulation

### Learning objective

Let learners observe how content popularity, TTL, cache-key dimensions, edge count, origin capacity, origin shield, purge, and failure policies affect cache hits, viewer latency, bandwidth, and origin load.

### Initial simulation controls

| Control | Default | Suggested range |
|---|---:|---:|
| Viewer requests/sec | 1,000 | 10–100,000 |
| Edge PoP count | 6 | 1–100 |
| Objects in catalog | 10,000 | 10–1,000,000 |
| Popularity skew (Zipf alpha) | 0.9 | 0.0–1.5 |
| Average object size | 32 KiB | 1 KiB–100 MiB |
| Edge cache capacity | 10 GiB | 100 MiB–10 TiB |
| Object TTL | 300 seconds | 0–86,400 |
| Origin RTT | 80 ms | 5–500 |
| Origin capacity | 2,000 requests/sec | 10–100,000 |
| Baseline request HIT ratio | 90% | Derived |
| Origin shield | Enabled | On / Off |
| Request collapsing | Enabled | On / Off |
| Cache policy | Public static | Static / revalidate / bypass |
| Stale serving | Disabled | Disabled / SWR / stale-if-error |
| Purge scope | One path | Path / tag / all |
| Origin failure | None | Timeout / 5xx / unavailable |
| Routing mode | DNS | DNS / anycast / multi-CDN |
| CDN provider failure | None | None / region / provider |

These values are simulator defaults, not provider specifications or production benchmarks.

### Model equations and assumptions

- Requests are assigned to a simulated PoP using a documented routing model and deterministic random seed.
- Request popularity follows a configurable Zipf-like distribution; exact object popularity must be generated and exposed to learners.
- Per-PoP request cache hit ratio = fresh eligible cache hits / eligible viewer requests for that PoP.
- Byte hit ratio = cached response bytes / eligible response bytes delivered.
- On a fresh HIT, modeled origin request count is zero.
- On a MISS, origin requests equal unique cache-key misses after optional parent hit and request collapsing; the simulator must not assume every viewer miss yields a distinct origin request when collapse is enabled.
- TTL controls freshness eligibility. Cache capacity and replacement policy can evict a fresh object before expiration.
- Revalidation uses a configured validator-hit probability and models a 304 as origin request/latency with a smaller response body, not a free cache hit.
- Origin load = fill requests + revalidations + bypassed requests + retries; report request and byte rates separately.
- Viewer latency = client-to-edge path + edge processing + (on MISS) parent/origin path + origin service + response transfer; HIT and MISS latency are separate estimates.
- Origin saturation occurs when offered origin work exceeds modeled capacity; use a bounded queue and show timeout/rejection rather than an unbounded queue.
- A purge removes matching CDN entries after a modeled propagation delay and can create a refill burst.
- Stale-while-revalidate may return a stale eligible object while one background refresh proceeds; stale-if-error is only used for configured failures and time windows.
- Multi-CDN failover changes routing after modeled detection and resolver/client convergence delay; the alternate provider starts with its own cache state.
- All latency, capacity, and availability results are simulated estimates, not provider measurements.

### Expected outputs

- Request and byte cache-hit ratios by PoP and overall.
- HIT, MISS, revalidation, bypass, stale, and error request counts.
- Viewer TTFB estimate for HIT, MISS, and failed-origin paths.
- Origin request rate, bandwidth, active fetches, and utilization.
- Shield/parent hit ratio and origin offload.
- Edge storage occupancy, eviction rate, and object residency.
- Purge completion time, refill burst, and freshness outcome.
- Successful, rejected, queued, stale-served, and timed-out responses.
- Per-region/provider request distribution and failover convergence.
- Security denials and private-content cache-isolation checks.

### Experiments

1. Compare direct-origin delivery with an edge cache for nearby and distant viewers.
2. Use uniform popularity versus a highly skewed object popularity distribution.
3. Increase edge storage and observe eviction, request hit ratio, byte hit ratio, and cost proxy.
4. Compare a short TTL, long TTL, and versioned object URL.
5. Revalidate an unchanged object using ETag and compare bytes transferred with a full fetch.
6. Purge one object versus all objects and observe refill/origin load.
7. Trigger a popular-object cache stampede with and without request collapsing, shield, and stale-while-revalidate.
8. Fail an origin and compare fresh hits, stale-if-error, and uncached behavior.
9. Add a regional cache/origin shield and measure miss latency versus origin request reduction.
10. Introduce a cache-key omission for a user-specific response and verify the simulator detects a privacy violation.
11. Compare DNS steering and anycast under resolver-location mismatch, regional failure, and convergence delay.
12. Fail one CDN in a multi-CDN setup and observe the alternate provider's cold-cache and origin refill behavior.

### Simulation model warning

The simulator must state its cache replacement policy, object popularity distribution, request routing assumptions, DNS/BGP convergence model, revalidation semantics, and failure timing. It must not treat a configured TTL as guaranteed residency, imply that a purge clears browser caches, equate request hit ratio with byte offload, or claim production latency from a simplified model.

---

## 16 — Design Challenge

### Challenge: Design a reliable global content delivery network

Design a CDN for a web platform that serves static assets, public APIs, private downloads, and video segments to viewers in multiple regions. The design must reduce viewer latency and origin load while preserving content correctness, access controls, and recoverability.

**Requirements**
- Define content classes and cacheability policy for immutable, mutable, dynamic, and private responses.
- Draw DNS/global routing, edge PoPs, regional cache/origin shield, origins, control plane, and monitoring.
- Explain cache keys, TTL, Cache-Control, validators, purge, and content versioning.
- Compare pull and push distribution and choose a strategy for each content type.
- Explain DNS-based routing and anycast, including their different selection and convergence characteristics.
- Protect origins with network/access controls, TLS, shielding, request limits, and retry budgets.
- Define safe signed URL/cookie validation and ensure private content cannot leak through shared cache.
- Design regional, origin, and optional multi-CDN failure handling.
- Specify cache, latency, byte, origin-offload, and availability monitoring.

**Constraints for the exercise**
- Start with one origin region and six edge PoPs.
- Use Section 03's illustrative 12,000 peak requests/sec and 32 KiB average response.
- Target at least 90% request hit ratio for eligible static content; report byte hit separately.
- Keep origin traffic under the stated budget during normal operation and one regional-cache failure.
- Inject a 20× request spike, a popular-object stampede, an origin timeout, a broad purge, and one CDN-provider outage.
- State the maximum acceptable staleness for each content class.

**Learner deliverables**
1. Functional and non-functional requirements with workload and freshness assumptions.
2. Naive and evolved architecture diagrams.
3. Cache HIT, MISS, revalidation, purge, and private-content flows.
4. Cache-key, TTL, validator, eviction, and invalidation policy.
5. Pull/push and DNS/anycast routing comparison.
6. Origin shielding, cache-fill, and request-collapsing design.
7. Origin, regional, and multi-CDN failover plan.
8. Security controls for TLS, origin access, WAF, signed links, and cache isolation.
9. Cost model and trade-off records.
10. A simulation plan with measurable acceptance criteria.

**Evaluation prompts**
- Which content may be safely shared by a public cache?
- How do you know all response-varying inputs are represented in the effective cache key?
- What does TTL guarantee, and what can eviction do before expiry?
- How does ETag revalidation differ from a cache HIT?
- How can a purge increase origin load, and how is refill controlled?
- What happens to private content if authorization is checked only on MISS?
- Does anycast select the geographically nearest edge or the best network route?
- How long can a DNS-based failover take from a viewer's perspective?
- What happens when all edge caches are cold and the origin is unhealthy?
- How do you measure origin offload without hiding uncached or bypass traffic?

---

## 17 — AI Interview

### Interviewer question bank

| ID | Difficulty | Question | What a strong answer should cover |
|---|---|---|---|
| Q01 | Beginner | What problem does a CDN solve? | Distance, repeated content, origin load, and failure boundaries |
| Q02 | Beginner | Walk through a request that produces a cache HIT. | DNS/routing, PoP, key lookup, freshness, response |
| Q03 | Beginner | What happens on a cache MISS? | Parent/shield, origin fetch, response policy, store and return |
| Q04 | Intermediate | Compare request hit ratio and byte hit ratio. | Different object sizes and traffic cost |
| Q05 | Intermediate | Explain TTL, Cache-Control, ETag, and a 304 response. | Freshness vs validation vs body transfer |
| Q06 | Intermediate | When should content be versioned rather than purged? | Immutable URL strategy, stale browser caches, rollback |
| Q07 | Intermediate | How can a bad cache key leak personalized content? | Representation dimensions, authorization, cache isolation tests |
| Q08 | Intermediate | Compare DNS routing and anycast. | Resolver-based policy vs BGP path selection and convergence |
| Q09 | Intermediate | What does an origin shield do, and what does it not guarantee? | Miss aggregation, origin protection, extra tier/failure behavior |
| Q10 | Advanced | How would you prevent a popular-object cache stampede? | Request collapsing, stale revalidation, jitter, bounded origin work |
| Q11 | Advanced | How should the CDN behave when origin fails? | Fresh hits, stale-if-error constraints, uncached/private failures |
| Q12 | Advanced | How do signed URLs remain secure when responses are cached? | Validate on every request, scope, expiry, key management, no leakage |
| Q13 | Advanced | How would you design a Multi-CDN failover? | Independent health, DNS/client convergence, cold cache, policy parity |
| Q14 | Advanced | How do you plan origin capacity for a purge or cold start? | Miss ratio, object popularity, refill, request collapse, rate budgets |
| Q15 | Advanced | Which metrics reveal that latency is caused by origin rather than edge? | HIT/MISS segmentation, TTFB, origin latency, bytes, region, status |

### Follow-up evaluation dimensions

- States workload, object popularity, freshness, and authorization assumptions.
- Separates cache freshness from cache residency and purge propagation.
- Explains cache-key correctness and privacy risks.
- Distinguishes DNS, anycast, and per-request CDN routing behavior.
- Sizes both viewer delivery and origin-fill capacity.
- Includes failure, stale-serving, and cold-cache behavior.
- Uses vendor-specific semantics only when supported by current official documentation.

---

## 18 — Architecture Score

### Proposed evaluation rubric

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements and workload assumptions | 10 | Content classes, viewer traffic, object sizes, freshness, SLO |
| Architecture and request flow | 15 | DNS/routing, edge, parent/shield, origin, control plane |
| Cache correctness and freshness | 15 | Cache key, TTL, validation, eviction, purge/versioning |
| Scalability and origin protection | 15 | Hit/offload model, request collapsing, capacity, cold-start plan |
| Reliability and global failover | 15 | PoP/region/origin/provider failure and convergence |
| Security and privacy | 10 | TLS, origin access, authorization, signed content, cache isolation |
| Cost and trade-off justification | 5 | Request/byte billing, origin transfer, security, operational costs |
| Observability and operations | 10 | HIT/MISS, TTFB, origin, purge, security, and freshness signals |
| **Total** | **100** | |

### Scoring guidance

- 0: Missing or technically incorrect.
- 1: Mentioned without explanation.
- 2: Partially explained.
- 3: Correct and reasonably justified.
- 4: Detailed, measurable, and handles edge cases.

Weighted score = sum of (dimension score / 4 × dimension weight).

The score should be accompanied by evidence and reviewer comments; it should not be treated as an objective measure of architecture quality without human review.
