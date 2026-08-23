---
id: DOC-08
title: Politeness, robots.txt, and Rate Limiting
version: 1.0.0
---

# Politeness and robots.txt

## 1. Principles

- P-1: Politeness rules are enforced in the dispatch path (fail-closed), not by monitoring after the fact.
- P-2: The crawler's interests never override a target site's expressed wishes.

## 2. robots.txt acquisition and caching

Per Host `(scheme, host, port)`:

1. Cache lookup: valid cached entry within TTL [CFG-008] ⇒ use it.
2. Else fetch `scheme://host:port/robots.txt` with UA Token, using the same fetch/timeout machinery as pages but exempt from page caps; this fetch itself obeys the host politeness window.
3. Interpret the HTTP status of the robots request per RFC 9309:
   - `2xx` → parse per RFC 9309 (groups matching UA Token, falling back to `*` group if no specific match).
   - `4xx` (incl. 404) → treat as "allow everything" for this Host.
   - `5xx` / network error / unparseable body → **UNKNOWN**: mark Host `robots_deferred_until = now + backoff` (starts 60 s, ×2 per consecutive failure, cap 24 h). No page fetches to that Host while deferred [DEC-007].
4. Cache stores: verdict function inputs + crawl_delay (seconds, from the applicable group) + fetched_at.

- R-100: If multiple groups match (specific token present), the `*` group MUST be ignored entirely (RFC 9309 §5.2).
- R-101: The longest-match rule applies to path prefixes; `Allow` and `Disallow` compared by longest path, tie ⇒ `Allow`.
- R-102: `Crawl-delay` values > 60 s are honored exactly (no clamping); missing ⇒ use [CFG-007].

## 3. Enforcement points

The robots verdict gates:
(a) every initial fetch of a URL identity;
(b) every redirect hop target [FR-021];
(c) recrawl eligibility re-check at dispatch time (verdicts can change between runs).

## 4. Rate limiting math

For each Host h, HOST REGISTRY maintains monotonic `next_allowed_fetch_at(h)`.

```
EffectiveDelay(h) = max( CFG-007,
                         crawl_delay(h),
                         backoff_ms(h) )      // if dynamic backoff enabled
backoff_ms(h)     = min(CFG-022 * CFG-023^(consecutive_failures(h)), 600000)
                    // reset to 0 on any success
```

Dispatch of a task for h atomically: verify `now ≥ next_allowed_fetch_at(h)`
and inflight < CFG-009, then set
`next_allowed_fetch_at(h) = max(next_allowed_fetch_at(h), now) + EffectiveDelay(h)`.

- R-110: Start-to-start spacing between two requests to one Host ≥ EffectiveDelay — guaranteed by construction above.
- R-111: HTTP `Retry-After` on 429/503 responses overrides computed backoff when larger: `next_allowed_fetch_at = max(computed, now + Retry-After)`.

## 5. Honesty

- R-120: UA Token format: `[ProductName]/[version] (+[contact-url])`; constant across all requests [CFG-018]. Compression/fingerprint variation is forbidden.
- R-121: Conditional requests (`If-None-Match`/`If-Modified-Since`) SHOULD be sent on refetches when validators are known; 304 responses count as success with unchanged payload.
