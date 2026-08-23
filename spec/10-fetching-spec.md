---
id: DOC-09
title: Fetching Specification (HTTP Behavior)
version: 1.8.0
---

# Fetching

## 1. Request construction

| Aspect | Rule |
|---|---|
| Method | GET only. |
| HTTP versions | HTTP/1.1 mandatory; HTTP/2 optional via ALPN; no h2c upgrade; HTTP/2 server push, if negotiated, MUST be disabled (no unsolicited resources are fetched). |
| TLS | TLS 1.2 minimum; certificate verification is mandatory and fail-closed [ERR-003]. |
| Headers | `User-Agent` = [CFG-018] exact; `Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`; `Accept-Encoding: gzip, deflate, br`; `From` iff [CFG-019]; conditional validators on refetches [R-121]. |
| Forbidden | Cookies, Authorization, custom fingerprint headers, referer spoofing; no request headers beyond those specified here (transport-mandated headers — `Host`/`:authority`, `Content-Length`, connection management — are exempt). |
| Body | None. |

- R-160: The crawler MUST be stateless with respect to site state: it MUST NOT store cookies, MUST NOT transmit any cookie received from any site, and MUST ignore `Set-Cookie` headers entirely.

## 2. Timeouts (all independent, all abortive)

| Timer | Config | Starts | Notes |
|---|---|---|---|
| DNS | [CFG-036] | resolution start | includes SSRF check time |
| Connect | [CFG-012] | connect start | TCP established |
| TLS | [CFG-013] | handshake start | certificate verified, fail closed |
| Response headers | [CFG-014] | request bytes written | until full header block received |
| Total transfer | [CFG-015] | request start | resets at each redirect hop; it does NOT span the whole chain |

Timeout violations classify ERR-001 (DNS), ERR-002 (connect), ERR-003 (TLS), ERR-012 (headers/total).

## 3. Redirects

- R-130: Follow 301, 302, 303, 307, 308 up to [CFG-017] hops; method stays GET
  throughout. Each `Location` value (which may be a relative reference) is
  resolved against the current hop's URL per RFC 3986 §5; a hop target that
  is not an acceptable absolute http(s) URL after resolution (disallowed
  scheme, or userinfo [R-001], [R-002]) terminates the chain with outcome
  PERMANENT and error_class ERR-011 (unacceptable target).
- R-131: Each hop: resolve DNS + SSRF check [DOC-16 §2], scope check [R-030],
  robots check [FR-021], and URL-blocklist check ([CFG-037]: a hop target
  matching the blocklist terminates the chain identically to [R-030] —
  outcome PERMANENT, error_class ERR-019, source URL → ST-180, target never
  fetched, hop recorded in `redirect_chain`; the blocklist MUST NOT be
  bypassable by redirects). Each hop request MUST additionally respect the
  target Host's politeness window and concurrency caps ([FR-011] b, c)
  before being sent; while a hop request is in flight it holds one
  concurrency slot on the target Host and one global slot — acquired
  immediately before the hop request is sent and released when the hop
  completes, so no slot is held during a politeness wait ([ERR-018]). A hop target that fails the target Host's robots gate terminates the chain identically to [R-030]: outcome PERMANENT, error_class ERR-017, source URL → ST-180, target never fetched, hop recorded in `redirect_chain`. A hop target whose robots verdict is UNKNOWN (target Host deferred [DOC-08 §2.3]) aborts the chain as outcome RETRYABLE with error_class ERR-010: source URL → ST-150 under normal retry accounting [DOC-13 §3], no request is sent to the target Host during deferral, and the next attempt re-runs the chain from the source. If respecting the target Host's politeness window would delay the hop request by more than [CFG-035] (e.g. an adversarially large `Crawl-delay` [R-102]), the chain is aborted as RETRYABLE with error_class ERR-018 instead of holding a fetch worker and the source Host's concurrency slot for the wait [G-4]: source URL → ST-150, with the target Host's window opening acting as an unclamped floor on `next_attempt_mono` [DOC-13 §3].
- R-132: Redirect loop detection: if any hop URL identity repeats within the chain ⇒ stop, ERR-011.
- R-133: The final hop's URL is recorded as final_url_identity; the original identity keeps its record, linked via `redirect_chain` (ordered list of identities), persisted as JSON on the attempt's `fetch_events` row [DOC-11 §1].

## 4. Response classification

| Status | Class | Action |
|---|---|---|
| 200–299 | success | store payload [FR-043] → ST-130 |
| 304 | success-unchanged | refresh freshness metadata, keep old payload hash [R-121]; if the stored blob no longer exists (retention), treat as cache miss and refetch fully [R-144] |
| 401, 403 | permanent | ST-180, ERR-014 (status recorded in fetch_events.http_status); do NOT retry |
| 404, 410 | permanent | ST-180, ERR-014 |
| 418, 451 | permanent | ST-180, ERR-014 |
| 429 | retryable | honor Retry-After [R-111] → ST-150 |
| other 4xx | permanent | ST-180, ERR-014 |
| 5xx | retryable | → ST-150, increments host consecutive_failures |
| 3xx non-followable (loop/cap/out-of-scope) | per cause | ERR-011, or ERR-015 for out-of-scope targets [R-030] |
| 3xx hop blocked by target Host robots | permanent | chain terminates at that hop; source → ST-180, ERR-017 [R-131] |
| 3xx hop target matches the [CFG-037] blocklist | permanent | chain terminates at that hop; source → ST-180, ERR-019 [R-131] |
| 3xx hop delayed > [CFG-035] by target Host politeness | retryable | chain aborts, ERR-018 [R-131]; source → ST-150, window opening floors `next_attempt_mono` |
| 3xx without parsable `Location` | permanent | ST-180, ERR-011 (missing, unparsable, or non-http(s) target [R-130]) |

## 5. Payload handling

- R-140: Decode Content-Encoding before hashing [FR-024]; unknown encodings ⇒ ERR-013 (retryable once, then permanent).
- R-141: Content-Length vs actual bytes mismatch ⇒ trust actual bytes, log anomaly metric.
- R-142: Charset detection order: HTTP `charset` param → BOM → HTML meta charset in first 1024 bytes → default UTF-8 (strict errors ⇒ replacement char, never fatal).
- R-143: Non-HTML types: sniff Content-Type only; payload stored iff
  [CFG-028]=true and the media type is in the fixed set {`image/*`,
  `application/pdf`, `text/plain`, `application/xml`, `application/rss+xml`,
  `application/atom+xml`} — the **effective allowed list** (empty when
  [CFG-028]=false). Types outside the effective allowed list ⇒ ERR-008
  [DOC-13 §1]; the payload is discarded [FR-041]. A missing `Content-Type`
  header is treated as `application/octet-stream`.
- R-144: A 304 response whose previously stored payload blob no longer exists
  on disk (e.g., removed by retention [DOC-11 §6]) MUST be treated as a cache
  miss: refetch with a full GET and store a fresh payload. The refetch is
  modeled like a redirect hop [R-131]: a new request respecting the Host's
  politeness window and caps, completing the same fetch attempt (no
  additional `attempts` increment). A politeness wait that would exceed
  [CFG-035] aborts the attempt RETRYABLE/ERR-018 exactly like a redirect
  hop [R-131].

## 6. FetchResult contract

Every attempt returns exactly:

```json
{
  "url_identity": "...",
  "outcome": "SUCCESS|UNCHANGED|RETRYABLE|PERMANENT",
  "error_class": "ERR-nnn|null",
  "http_status": 200,
  "final_url_identity": "...",
  "redirect_chain": ["..."],
  "payload_sha256": "...|null",
  "payload_size": 12345,
  "content_type": "text/html; charset=utf-8",
  "validators": {"etag": "...", "last_modified": "..."},
  "timings_ms": {"dns":12,"connect":8,"tls":15,"ttfb":40,"total":180},
  "attempt": 2
}
```

Scheduler consumes outcomes per [DOC-13].

- R-145: For redirect chains, `timings_ms.total` spans the entire chain
  (including per-hop politeness waits); `dns`/`connect`/`tls`/`ttfb` refer to
  the final hop's connection; `payload_sha256`/`payload_size`/`content_type`
  describe the final response.
