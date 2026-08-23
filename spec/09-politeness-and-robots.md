---
id: DOC-08
title: Politeness, robots.txt, and Rate Limiting
version: 1.8.0
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
   while in flight. Redirect responses for the robots request itself are
   followed per [DOC-09 §3] hop rules — SSRF [R-400], politeness, and caps
   apply — but scope [R-030] and robots gates are not applied recursively
   (there is no robots-for-robots); the verdict is taken from the final
   response. A robots fetch that terminates without a final response —
   redirect cap exhausted [R-130], hop SSRF block [R-400], or [ERR-018]
   hop-wait abort — is a network-class failure: the Host moves to
   UNKNOWN/deferral per item 3 (fail closed [DEC-007]). robots.txt exchanges
   are not page fetches: they create no
   fetch_events rows (visibility is via robots_queries_total [DOC-15 §1])
   and never modify consecutive_failures or pages_crawled [R-112].
3. Interpret the HTTP status of the robots request per RFC 9309:
   - `2xx` → parse per RFC 9309. Parsing is error-tolerant (NFR-014):
     unrecognized or malformed lines are ignored, so a 2xx body always yields
     a rule set, possibly empty; a 2xx body never yields UNKNOWN — an empty
     rule set ⇒ ALLOW everything, exactly like `4xx`. Group selection per
     [R-100]. Only the first 500 KiB of the body is processed (RFC 9309
     size cap), excess bytes are ignored.
   - `4xx` (incl. 404) → treat as "allow everything" for this Host.
   - `5xx` / network error / body that fails to decode (e.g. `Content-Encoding`
     failure) → **UNKNOWN**: mark Host `robots_deferred_until = now + backoff` (starts 60 s, ×2 per consecutive failure, cap [CFG-040]); `robots_deferred_since_mono` is set on the first deferral of the streak and cleared when an authoritative verdict is obtained. No page fetches to that Host while deferred [DEC-007].
4. Cache stores: verdict function inputs + crawl_delay (seconds, from the applicable group) + fetched_at; persisted on the Host row (`robots_rules`) [DOC-11 §1].

- R-100: Group selection (RFC 9309): a group matches the crawler when its
  `User-agent` value is a case-insensitive substring of the UA Token's
  product name (the part before `/`). If more than one non-`*` group matches,
  only the most specific one (longest `User-agent` value) applies; in
  particular, the `*` group MUST be ignored entirely whenever any matching
  specific group exists. If no group matches and no `*` group exists, the
  rule set is empty ⇒ ALLOW everything.
- R-101: Rule matching is byte-wise and case-sensitive against the URL's
  path concatenated, when a query is present, with `?` followed by the query
  string (so `Disallow: /*?` can match query-bearing URLs). Exactly two
  special characters are recognized in rule values: `*` (matches any
  sequence of characters, including the empty sequence) and a terminal `$`
  (anchors the end of the match target); no other character is special.
  Precedence between a matching `Allow` and a matching `Disallow` is by
  longest rule value in characters (wildcards counting as one); tie ⇒
  `Allow`. An empty `Disallow` value ⇒ ALLOW everything.
- R-102: `Crawl-delay` is honored exactly as received — no clamping, including values > 60 s; a missing `Crawl-delay` ⇒ use [CFG-007].
- R-103: If a Host remains continuously in the UNKNOWN/deferred state for ≥
  [CFG-040] — measured from `robots_deferred_since_mono` [DOC-11 §1] — every
  URL Record on that Host in a gated state (ST-100 or ST-150) MUST be moved
  to ST-190/`ROBOTS_UNKNOWN_TIMEOUT` (bounded resource use under permanent
  robots failure [G-4]); until that threshold is reached, gated URLs stay in
  their current state and are reconsidered after each deferral expiry. In-flight
  records (ST-110/ST-120) are never affected; ST-130 records complete normally.
  The threshold expiry (`robots_deferred_since_mono + [CFG-040]`) is a
  scheduler wake source [R-211], and the scheduling loop performs the sweep.
- R-104: Revalidation (stale-while-revalidate): when the cached entry has
  expired ([CFG-008] TTL) but exists, the refetch is scheduled per items 1–3
  above; while it is in flight, the previous entry's rules remain authoritative
  for gate queries. A terminal 2xx/4xx response replaces the cached verdict
  atomically; a failure moves the Host to deferral per item 3 (the stale rules
  are retained for the next successful parse but do not gate during
  deferral).

## 3. Enforcement points

The robots verdict gates:
(a) every initial fetch of a URL identity — evaluated during dispatch
    selection, while the record is ST-100 [FR-011(d)];
(b) every redirect hop target [FR-021];
(c) recrawl eligibility re-check at dispatch time (verdicts can change between runs).
A verdict that changes after [T-1] but before the request is sent is handled
by [R-054]: DISALLOW ⇒ ST-110→ST-190 with slot release; UNKNOWN (Host
deferred) ⇒ compensated back to ST-100 and reconsidered after deferral
expiry.

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

- R-110: Start-to-start spacing between two requests to one Host ≥ EffectiveDelay — guaranteed by construction above. The request for a dispatched URL MUST be sent immediately after [T-1] commits; the only permitted pre-send step is the [R-054] re-check (which, when it fires, sends nothing), so dispatch-to-send latency cannot erode the guarantee.
- R-111: HTTP `Retry-After` on 429/503 responses overrides computed backoff when larger: `next_allowed_fetch_at = max(computed, now + Retry-After)`. `Retry-After` is parsed as delta-seconds or HTTP-date (RFC 9110); unparseable values are ignored (computed backoff applies).
- R-112: `consecutive_failures(h)` is incremented exactly by retryable page-fetch failures completed against Host h — outcome RETRYABLE with error_class ∈ {ERR-001 (transient resolution failure; permanent NXDOMAIN excluded), ERR-002, ERR-003, ERR-005, ERR-006, ERR-012, ERR-013}. ERR-010 (deferral) and every PERMANENT outcome leave it unchanged; any SUCCESS/UNCHANGED page fetch resets it to 0 [R-231]. robots.txt exchanges never modify it (their failure semantics are the deferral streak, §2.3). For redirect chains crossing Hosts, all host-counter effects (`consecutive_failures`, `pages_crawled`) attribute to the Host of the final hop — the exchange that produced the outcome; the source Host's counters are unaffected ([T-2]).

## 5. Honesty

- R-120: UA Token format: `[ProductName]/[version] (+[contact-url])`; constant across all requests [CFG-018]. Compression/fingerprint variation is forbidden.
- R-121: Conditional requests (`If-None-Match`/`If-Modified-Since`) SHOULD be sent on refetches when validators are known; 304 responses count as success with unchanged payload.
