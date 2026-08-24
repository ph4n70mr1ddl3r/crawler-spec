---
id: DOC-09
title: Fetching Specification (HTTP Behavior)
version: 1.14.0
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
| Response headers | [CFG-014] | request bytes written | until the final status line's full header block received; interim `1xx` blocks do not satisfy it [§4] |
| Total transfer | [CFG-015] | request start | resets at each redirect hop; it does NOT span the whole chain |

Timeout violations classify ERR-001 (DNS), ERR-002 (connect), ERR-003 (TLS), ERR-012 (headers/total).

## 3. Redirects

- R-130: Follow 301, 302, 303, 307, 308 up to [CFG-017] hops; method stays GET
  throughout. Each `Location` value (which may be a relative reference) is
  resolved against the current hop's URL per RFC 3986 §5; a hop target that
  is not an acceptable absolute http(s) URL after resolution (disallowed
  scheme, or userinfo [R-001], [R-002]) terminates the chain with outcome
  PERMANENT and error_class ERR-011 (unacceptable target). A `3xx` status
  outside the followable set (e.g. `300`) likewise terminates the attempt
  PERMANENT/ERR-011, with or without a `Location` — no follow is defined
  for it (a `304` is not such a case: it is classified earlier, as
  success-unchanged [§4], [R-144]).
- R-131: Each hop: resolve DNS + SSRF check [DOC-16 §2], scope check [R-030],
  robots check [FR-021], and URL-blocklist check ([CFG-037]: a hop target
  matching the blocklist terminates the chain identically to [R-030] —
  outcome PERMANENT, error_class ERR-019, source URL → ST-180, target never
  fetched, hop recorded in `redirect_chain`; the blocklist MUST NOT be
  bypassable by redirects). A hop target that fails the target Host's robots gate terminates the chain identically to [R-030]: outcome PERMANENT, error_class ERR-017, source URL → ST-180, target never fetched, hop recorded in `redirect_chain`. A hop target whose robots verdict is UNKNOWN aborts the chain as outcome RETRYABLE with error_class ERR-010 — UNKNOWN because the target Host is deferred [DOC-08 §2.3]; an UNKNOWN returned merely because a robots acquisition is in flight on the target Host is instead awaited per [R-105] (bounded by the robots exchange's timeouts, no Host unit held [R-051]). In the ERR-010 case: source URL → ST-150 under normal retry accounting [DOC-13 §3], no request is sent to the target Host during deferral, and the next attempt re-runs the chain from the source.

  Hop dispatch protocol (release-before-acquire [R-051] — a waiting task
  holds no Host unit, so politeness and capacity waits can never deadlock
  [G-4]; without it, two cross-host chains A→B and B→A under [CFG-009]=1
  would each hold their source Host's unit while waiting forever for the
  other's):

  1. When a redirect response is received, the per-Host unit held for the
     responding Host is released (that request is finished). The task's
     single global unit is unaffected — it spans [T-1]..[T-2] [DOC-00], so
     hops never wait on global capacity and never hold a second global
     unit.
  2. Holding no Host unit, the target's hop gates are evaluated (first
     paragraph of this rule: DNS+SSRF, scope, robots, blocklist). A
     refusal aborts the chain with the outcome stated there — holding no
     Host unit, so nothing remains to release at completion beyond the
     global unit. A robots verdict of UNKNOWN merely because an
     acquisition is in flight on the target Host is awaited here per
     [R-105] (no Host unit held).
  3. Before the next hop request is sent, the target Host's politeness
     window and per-Host concurrency cap [CFG-009] MUST be respected. Waits
     hold no Host unit. If respecting the politeness window would delay the
     send by more than [CFG-035] (e.g. an adversarially large `Crawl-delay`
     [R-102]), the chain is aborted as RETRYABLE with error_class ERR-018
     instead of holding a fetch worker for the wait [G-4]: source URL →
     ST-150, with the target Host's window opening acting as an unclamped
     floor on `next_attempt_mono` [DOC-13 §3]. A capacity wait (all
     [CFG-009] target-Host units in use) needs no such threshold: the
     holders are in-flight requests that complete within their timeouts,
     and the waiting task holds no Host unit, so the wait is bounded and
     cannot cycle.
  4. Immediately before the hop request is sent, one unit is acquired on
     the target Host and the window is advanced identically to [FR-012]
     (`next_allowed_fetch_at = max(next_allowed_fetch_at, now) +
     EffectiveDelay`); while the request is in flight the task holds that
     one unit and no other Host unit [R-051].
- R-132: Redirect loop detection: a hop target whose identity equals the
  original identity or repeats any earlier hop target's identity within the
  chain ⇒ stop, ERR-011 (the check fires at the first such repetition —
  A→B→A is caught when the second hop target `A` is seen, not only on a
  later B).
- R-133: The chain's outcome URL is recorded as final_url_identity; the original identity keeps its record, linked via `redirect_chain` (ordered list of identities), persisted as JSON on the attempt's `fetch_events` row [DOC-11 §1]. `redirect_chain` lists, in order, every hop target the chain acted on — each `Location` target it followed, and each it refused at a gate or limit (loop [R-132], redirect cap [R-130], scope [ERR-015], robots [ERR-017], blocklist [ERR-019], SSRF [ERR-004], robots deferral [ERR-010], rate-limit abort [ERR-018]) — and never contains the requested identity as such (the list holds hop targets only; a loop-refused hop target that happens to equal the original identity is still recorded — it is a refused hop target, not the requested URL [R-132]); on any chain that received a final response the followed final target is its last element. It is empty for a no-redirect fetch. `final_url_identity` is the URL of the last response received in the attempt: the identity itself for a no-redirect fetch, the redirecting hop's URL for a pre-send abort, and the final target's URL when a final response arrived.

## 4. Response classification

| Status | Class | Action |
|---|---|---|
| 1xx (interim) | — (never an outcome) | interim responses are consumed by the transport (RFC 9110 §15.2); the attempt continues and the [CFG-014] header timer keeps running until the final status line's header block arrives — a stream yielding no final status before the timer aborts ERR-012 [§2] |
| 200–299 | success | store payload [FR-043] → ST-130 |
| 304 | success-unchanged | refresh freshness metadata, keep old payload hash [R-121]; if no usable stored payload exists (blob removed by retention, or none ever stored — [R-144]), treat as cache miss and refetch fully [R-144] |
| 401, 403 | permanent | ST-180, ERR-014 (status recorded in fetch_events.http_status); do NOT retry |
| 404, 410 | permanent | ST-180, ERR-014 |
| 418, 451 | permanent | ST-180, ERR-014 |
| 429 | retryable | honor Retry-After [R-111] → ST-150 |
| other 4xx | permanent | ST-180, ERR-014 |
| 5xx | retryable | → ST-150, increments host consecutive_failures |
| 3xx non-followable (loop/cap/out-of-scope) | per cause | ERR-011, or ERR-015 for out-of-scope targets [R-030] |
| 3xx hop blocked by target Host robots | permanent | chain terminates at that hop; source → ST-180, ERR-017 [R-131] |
| 3xx hop target matches the [CFG-037] blocklist | permanent | chain terminates at that hop; source → ST-180, ERR-019 [R-131] |
| 3xx hop target fails SSRF/egress policy | permanent | chain terminates at that hop; source → ST-180, ERR-004, referring Host flagged `suspicious=true`; target never fetched [R-400], [R-402] |
| 3xx hop delayed > [CFG-035] by target Host politeness | retryable | chain aborts, ERR-018 [R-131]; source → ST-150, window opening floors `next_attempt_mono` |
| 3xx without parsable `Location` | permanent | ST-180, ERR-011 (missing, unparsable, or non-http(s) target [R-130]) |
| 3xx outside the followable set (e.g. 300) | permanent | ST-180, ERR-011, with or without a `Location` [R-130]; a `304` is not this case — the success-unchanged row above applies |

## 5. Payload handling

- R-140: Decode Content-Encoding before hashing [FR-024]; unknown encodings ⇒ ERR-013 (retryable once, then permanent).
- R-141: Content-Length vs actual bytes mismatch ⇒ trust actual bytes, log anomaly metric.
- R-142: Charset detection order: HTTP `charset` param → BOM → HTML meta charset in first 1024 bytes → default UTF-8 (strict errors ⇒ replacement char, never fatal).
- R-143: Non-HTML types: sniff Content-Type only; payload stored iff
  [CFG-028]=true and the media type is in the fixed set {`image/*`,
  `application/pdf`, `text/plain`, `application/xml`, `text/xml`,
  `application/rss+xml`, `application/atom+xml`} — the **effective allowed list** (empty when
  [CFG-028]=false). Types outside the effective allowed list ⇒ ERR-008
  [DOC-13 §1]; the payload is discarded [FR-041]. A missing `Content-Type`
  header is treated as `application/octet-stream`.
- R-144: A 304 response with no usable stored payload MUST be treated as a
  cache miss: refetch with a full unconditional GET and store a fresh
  payload. Two sub-cases: (a) the previously stored payload blob no longer
  exists on disk (e.g., removed by retention [DOC-11 §6]); (b) no payload
  was ever stored for the URL — a 304 on a first fetch or to an
  unconditional request (servers MUST NOT send it, but MUST NOT is not a
  guarantee). The refetch is modeled like a redirect hop [R-131] (unit
  transfer, politeness window, caps): a new request completing the same
  fetch attempt (no additional `attempts` increment); a politeness wait that
  would exceed [CFG-035] aborts the attempt RETRYABLE/ERR-018 exactly like
  a redirect hop [R-131]. If the unconditional refetch also returns 304,
  the outcome is PERMANENT with error_class ERR-014 — the server cannot
  satisfy any request for this resource, so retrying cannot help (this
  guard bounds the refetch to one per attempt).

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
