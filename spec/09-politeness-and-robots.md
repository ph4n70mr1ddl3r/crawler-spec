---
id: DOC-08
title: Politeness, robots.txt, and Rate Limiting
version: 1.4.0
---

# Politeness and robots.txt

## 1. Principles

- P-1: Politeness rules are enforced in the dispatch path (fail-closed), not by monitoring after the fact.
- P-2: The crawler's interests never override a target site's expressed wishes.

## 2. robots.txt acquisition and caching

Per Host `(scheme, host, port)`:

1. Cache lookup: valid cached entry within TTL [CFG-008] ⇒ use it.
2. Else fetch `scheme://host:port/robots.txt` with UA Token, using the same
   fetch/timeout machinery as pages. The robots fetch is exempt only from
   page-success caps [CFG-006] and from Content Store storage; every transport
   safety cap [DOC-16 §3] still applies (security precedence [R-000]). It obeys
   the host politeness window, advances `next_allowed_fetch_at` identically to
   a page dispatch [FR-012], and holds one per-host/global concurrency slot
   while in flight.
3. Interpret the HTTP status of the robots request per RFC 9309:
   - `2xx` → parse per RFC 9309 (groups matching UA Token, falling back to `*` group if no specific match); only the first 500 KiB of the body is processed (RFC 9309 size cap), excess bytes are ignored.
   - `4xx` (incl. 404) → treat as "allow everything" for this Host.
   - `5xx` / network error / unparseable body → **UNKNOWN**: mark Host `robots_deferred_until = now + backoff` (starts 60 s, ×2 per consecutive failure, cap [CFG-040]); `robots_deferred_since_mono` is set on the first deferral of the streak and cleared when an authoritative verdict is obtained. No page fetches to that Host while deferred [DEC-007].
4. Cache stores: verdict function inputs + crawl_delay (seconds, from the applicable group) + fetched_at; persisted on the Host row (`robots_rules`) [DOC-11 §1].

- R-100: If multiple groups match (specific token present), the `*` group MUST be ignored entirely (RFC 9309).
- R-101: The longest-match rule applies to path prefixes; `Allow` and `Disallow` compared by longest path, tie ⇒ `Allow`.
- R-102: `Crawl-delay` values > 60 s are honored exactly (no clamping); missing ⇒ use [CFG-007].
- R-103: If a Host remains continuously in the UNKNOWN/deferred state for ≥
  [CFG-040] — measured from `robots_deferred_since_mono` [DOC-11 §1] — every
  URL Record on that Host in a gated state (ST-100 or ST-150) MUST be moved
  to ST-190/`ROBOTS_UNKNOWN_TIMEOUT` (bounded resource use under permanent
  robots failure [G-4]); until that threshold is reached, gated URLs stay in
  their current state and are reconsidered after each deferral expiry. In-flight
  records (ST-110/ST-120) are never affected; ST-130 records complete normally.

## 3. Enforcement points

The robots verdict gates:
(a) every initial fetch of a URL identity — evaluated during dispatch
    selection, while the record is ST-100 [FR-011(d)];
(b) every redirect hop target [FR-021];
(c) recrawl eligibility re-check at dispatch time (verdicts can change between runs).
A verdict that changes after [T-1] but before the request is sent is handled
by the ST-110→ST-190 transition [DOC-07 §2] with slot release [R-051].

## 4. Rate limiting math

For each Host h, HOST REGISTRY maintains monotonic `next_allowed_fetch_at(h)`.

```
EffectiveDelay(h) = max( CFG-007,
                         crawl_delay(h),
                         backoff_ms(h) )      // backoff term applies only
                                              // if [CFG-011]=true
backoff_ms(h)     = consecutive_failures(h) = 0
                    ? 0
                    : min(CFG-022 * CFG-023^(consecutive_failures(h) - 1),
                          CFG-035)
                    // reset to 0 on any success; exponent aligned with the
                    // per-URL retry formula [DOC-13 §3]
```

Dispatch of a task for h atomically: verify `now ≥ next_allowed_fetch_at(h)`
and inflight < CFG-009, then set
`next_allowed_fetch_at(h) = max(next_allowed_fetch_at(h), now) + EffectiveDelay(h)`.

- R-110: Start-to-start spacing between two requests to one Host ≥ EffectiveDelay — guaranteed by construction above.
- R-111: HTTP `Retry-After` on 429/503 responses overrides computed backoff when larger: `next_allowed_fetch_at = max(computed, now + Retry-After)`. `Retry-After` is parsed as delta-seconds or HTTP-date (RFC 9110); unparseable values are ignored (computed backoff applies).

## 5. Honesty

- R-120: UA Token format: `[ProductName]/[version] (+[contact-url])`; constant across all requests [CFG-018]. Compression/fingerprint variation is forbidden.
- R-121: Conditional requests (`If-None-Match`/`If-Modified-Since`) SHOULD be sent on refetches when validators are known; 304 responses count as success with unchanged payload.
