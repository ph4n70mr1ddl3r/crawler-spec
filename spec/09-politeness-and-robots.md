---
id: DOC-08
title: Politeness, robots.txt, and Rate Limiting
version: 1.12.0
---

# Politeness and robots.txt

## 1. Principles

- P-1: Politeness rules are enforced in the dispatch path (fail-closed), not by monitoring after the fact.
- P-2: The crawler's interests never override a target site's expressed wishes.

## 2. robots.txt acquisition and caching

Per Host `(scheme, host, port)`:

1. Cache lookup: valid cached entry within TTL [CFG-008] — elapsed
   monotonic time since `robots_fetched_at_mono` [DOC-11 §1], so TTL expiry
   is replay-deterministic like every other scheduler-consumed expiry
   [DEC-012], [NFR-006] — ⇒ use it.
2. Else fetch `scheme://host:port/robots.txt` with UA Token, using the same
   fetch/timeout machinery as pages. Acquisition is single-flight per Host
   [R-105]: while an exchange is in flight, concurrent gate queries await
   its verdict instead of initiating another. The robots fetch is exempt only from
   page-success caps [CFG-006] and from Content Store storage; every transport
   safety cap [DOC-16 §3] still applies (security precedence [R-000]). It obeys
   the host politeness window, advances `next_allowed_fetch_at` identically to
   a page dispatch [FR-012], and holds one per-Host and one global unit
   while in flight. Redirect responses for the robots request itself are
   followed per [DOC-09 §3] hop rules — SSRF [R-400], the [CFG-037] URL
   blocklist [R-131], politeness, and caps apply — but scope [R-030] and
   robots gates are not applied recursively
   (there is no robots-for-robots); the verdict is taken from the final
   response. A robots fetch that terminates without a final response —
   redirect cap exhausted [R-130], redirect loop [R-132], a `Location`
   that is missing, unparsable, or not an acceptable absolute http(s) URL
   [R-130], hop SSRF block [R-400], a hop target matching the [CFG-037]
   URL blocklist (the blocklist MUST NOT be bypassable by redirects —
   robots fetches included), or an [ERR-018] hop-wait abort — is a
   network-class failure: the Host moves to
   UNKNOWN/deferral per item 3 (fail closed [DEC-007]). robots.txt exchanges
   are not page fetches: they create no
   fetch_events rows (visibility is via robots_queries_total [DOC-15 §1])
   and never modify consecutive_failures or pages_crawled [R-112]. Their
   politeness advance and unit acquisition/release mirror the host-side
   effects of [T-1]/[T-2] — the window advances before the robots request is
   sent, and the units are released when the exchange completes and its
   verdict/cache update commits — but no URL Record participates: robots
   exchanges commit neither [T-1] nor [T-2]. The one global and one
   per-Host unit are acquired atomically before the robots request is sent
   (mirroring [T-1]'s single-commit acquisition): while waiting for
   capacity the exchange holds no units, so a saturated global pool cannot
   deadlock against robots acquisition — in-flight tasks release their
   units within their timeouts.
3. Interpret the HTTP status of the robots request per RFC 9309:
   - `2xx` → parse per RFC 9309. Parsing is error-tolerant (NFR-014):
     unrecognized or malformed lines are ignored, so a 2xx body always yields
     a rule set, possibly empty; a 2xx body never yields UNKNOWN — an empty
     rule set ⇒ ALLOW everything, exactly like `4xx`. Group selection per
     [R-100]. Only the first 500 KiB of the body is processed (RFC 9309
     size cap), excess bytes are ignored.
   - `4xx` (incl. 404) → treat as "allow everything" for this Host.
   - `5xx` / network error / body that fails to decode (e.g. `Content-Encoding`
     failure) → **UNKNOWN**: mark Host `robots_deferred_until_mono = now + backoff` (starts [CFG-044], ×2 (fixed) per consecutive failure — the streak is
     persisted as `hosts.robots_defer_failures` [DOC-11 §1] so escalation
     survives restarts, and is cleared when an authoritative verdict is
     obtained — cap [CFG-040]); `robots_deferred_since_mono` is set on the first deferral of the streak; both deferral timestamps (`robots_deferred_until_mono`, `robots_deferred_since_mono`) are cleared when an authoritative verdict is obtained. No page fetches to that Host while deferred [DEC-007].
4. Cache stores: verdict function inputs + crawl_delay (seconds, from the applicable group) + fetched_at — the monotonic TTL basis `robots_fetched_at_mono` plus the wall-clock audit timestamp `robots_fetched_at`; persisted on the Host row (`robots_rules`) [DOC-11 §1].

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
- R-102: `Crawl-delay` is honored exactly as received — no clamping, including values > 60 s; a missing `Crawl-delay` ⇒ use [CFG-007]. If the applicable group contains multiple `Crawl-delay` lines, the first one applies (error-tolerant parsing ignores the rest).
- R-103: If a Host remains continuously in the UNKNOWN/deferred state for ≥
  [CFG-040] — measured from `robots_deferred_since_mono` [DOC-11 §1] — every
  URL Record on that Host in a gated state (ST-100 or ST-150) MUST be moved
  to ST-190/`ROBOTS_UNKNOWN_TIMEOUT` (bounded resource use under permanent
  robots failure [G-4]); until that threshold is reached, gated URLs stay in
  their current state and are reconsidered after each deferral expiry. In-flight
  records (ST-110/ST-120) are never affected; ST-130 records complete normally.
  The threshold expiry (`robots_deferred_since_mono + [CFG-040]`) is a
  scheduler wake source [R-211], and the scheduling loop performs the sweep.
  The sweep condition is re-evaluated on every scheduler iteration (its wake
  sources — discovery, deferral-expiry retries, due-time promotions — recur
  while the Host stays deferred): records that enter a gated state after the
  threshold first passed (new discoveries, ST-140→ST-100 recrawl promotions)
  are excluded by the next evaluation, not only those already gated at the
  threshold expiry.
- R-104: Revalidation (stale-while-revalidate): when the cached entry has
  expired ([CFG-008] TTL) but exists, the refetch is triggered by the first
  gate query needing a verdict and performed per items 1–3 above
  (single-flight [R-105]); while it is in flight, the previous entry's rules remain authoritative
  for gate queries. A terminal 2xx/4xx response replaces the cached verdict
  atomically; a failure moves the Host to deferral per item 3 (the stale rules
  are retained for the next successful parse but do not gate during
  deferral).
- R-105: Robots acquisition is single-flight per Host and lazily initiated:
  at most one robots.txt exchange per Host is in flight at any time — any
  gate query arriving while an acquisition is in flight awaits that
  exchange's verdict (committed atomically per [R-104]) instead of starting
  another. The exchange is initiated by the first event needing a verdict:
  a gate query against a Host with `robots_state` = INITIAL [FR-030], the
  first gate query after cache TTL expiry ([R-104] revalidation), or the
  first gate query after a deferral expires (the §2.3 retry — previously
  its initiator was implicit in §2.1's "else fetch"). While no
  authoritative verdict exists, the gate returns UNKNOWN [DEC-007]; an
  acquisition's completion is a scheduler wake source [R-211].
- R-106: `robots_state` lifecycle (the column is the single authoritative
  gate input [DOC-11 §1]): INITIAL from Host-row creation until the first
  authoritative verdict; a parsed 2xx response ⇒ OK (rules cached, the
  applicable group's `crawl_delay_s` stored); a 4xx response ⇒ ALLOW_ALL
  (empty rule set — allow everything); entering deferral (§2.3, including a
  failed revalidation [R-104]) ⇒ DEFERRED; obtaining an authoritative
  verdict ⇒ OK or ALLOW_ALL with all three deferral columns cleared
  (§2.3). The gate verdict is UNKNOWN iff `robots_state` ∈ {INITIAL,
  DEFERRED} — including after a deferral expires while the retry [R-105]
  is still in flight — and is otherwise computed from the cached rules:
  ALLOW/DISALLOW under OK, ALLOW under ALLOW_ALL. `robots_fetched_at_mono`
  is (re)set exactly when an authoritative verdict is stored (§2.4).

## 3. Enforcement points

The robots verdict gates:
(a) every initial fetch of a URL identity — evaluated during dispatch
    selection, while the record is ST-100 [FR-011(d)];
(b) every redirect hop target [FR-021];
(c) recrawl eligibility re-check at dispatch time (verdicts can change between runs).
A verdict that changes after [T-1] but before the request is sent is handled
by [R-054]: DISALLOW ⇒ ST-110→ST-190 with unit release; UNKNOWN (Host
deferred) ⇒ compensated back to ST-100 and reconsidered after deferral
expiry.

## 4. Rate limiting math

For each Host h, HOST REGISTRY maintains monotonic `next_allowed_fetch_at(h)`.

```
EffectiveDelay(h) = max( CFG-007,                  // ms [CFG-007]
                         1000 × crawl_delay_s(h),  // Crawl-delay is stored in
                                                  // seconds [DOC-11 §1]; ×1000
                                                  // converts to ms — comparing
                                                  // raw seconds against ms terms
                                                  // would under-wait (5 ≠ 5000)
                         backoff_ms(h) )           // backoff term applies only
                                                  // if [CFG-011]=true; ALL THREE
                                                  // TERMS ARE MILLISECONDS
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
