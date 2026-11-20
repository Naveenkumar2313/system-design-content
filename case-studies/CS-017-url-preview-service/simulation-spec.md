# CS-017 — Simulation Specification

## Goal

Demonstrate cache behavior, metadata extraction, SSRF prevention, redirect handling, queue latency, worker capacity, negative caching, and preview cost.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| preview_requests_per_second | Preview requests/s | Slider | 100 | 1 | 100000 |
| unique_url_ratio | New URL ratio | Slider | 0.10 | 0 | 1 |
| cache_hit_ratio | Cache hit ratio | Slider | 0.90 | 0 | 1 |
| html_size_kib | HTML size | Slider | 500 | 1 | 2048 |
| fetch_timeout_ms | Fetch timeout | Slider | 1500 | 50 | 30000 |
| max_redirects | Redirect limit | Slider | 5 | 0 | 20 |
| worker_count | Worker count | Slider | 50 | 1 | 1000 |
| concurrency_per_worker | Concurrency/worker | Slider | 100 | 1 | 500 |
| ssrf_probability | Malicious URL share | Slider | 0.01 | 0 | 1 |
| origin_failure_probability | Origin failure share | Slider | 0.05 | 0 | 1 |
| negative_ttl_seconds | Negative TTL | Slider | 300 | 30 | 86400 |
| positive_ttl_seconds | Positive TTL | Slider | 86400 | 60 | 2592000 |
| browser_render_ratio | Browser-render share | Slider | 0.05 | 0 | 1 |

## Derived values

- Cache misses = requests × `(1 - cache_hit_ratio)`.
- Fetch jobs = misses × `unique_url_ratio`.
- Queue age = offered fetch jobs / bounded worker capacity.
- Daily fetch bytes = fetch jobs × HTML size × retry factor.
- SSRF attempts are rejected before socket creation.
- Browser jobs use a separate pool.
- Cost and latency are simulated estimates.

## Required scenarios

1. Cache HIT versus MISS.
2. Localhost/private/link-local/metadata URL attacks.
3. DNS rebinding and redirect escape.
4. OG/Twitter/HTML/Schema/fallback extraction.
5. Malformed HTML and missing metadata.
6. Cache stampede.
7. Worker saturation.
8. Redis/database/queue failure.
9. Duplicate worker completion.
10. Oversized HTML/compression/image.
11. Per-domain hot spot.
12. Preview refresh/invalidation.

## Acceptance checks

- Private destinations are rejected before fetch.
- Every redirect is revalidated.
- One normalized URL has one logical in-flight fetch.
- Cache HIT does not contact origin.
- Downloaded HTML is never executed.
- Text is escaped and bounded.
- Negative cache prevents immediate retry storms.
- Outputs are clearly labeled as estimates.
