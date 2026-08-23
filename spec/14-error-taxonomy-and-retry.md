---
id: DOC-13
title: Error Taxonomy and Retry Policy
version: 1.7.0
---

# Errors and Retry

## 1. Error classes

| ID | Name | Retryable | Notes |
|----|------|-----------|-------|
| ERR-001 | DNS_FAILURE | yes | NXDOMAIN is NOT retried (permanent) |
| ERR-002 | CONNECT_TIMEOUT / CONN_REFUSED / NET_UNREACH | yes | |
| ERR-003 | TLS_ERROR | yes (once) | then permanent; cert verification failures are PERMANENT immediately |
| ERR-004 | SSRF_BLOCKED | no | security; log at WARN [DOC-16] |
| ERR-005 | HTTP_429 | yes | honor Retry-After [R-111] |
| ERR-006 | HTTP_5XX | yes | |
| ERR-007 | PAYLOAD_TOO_LARGE | no | [FR-023] |
| ERR-008 | UNSUPPORTED_TYPE | no | content type outside the effective allowed list ([R-143]: fixed set when [CFG-028]=true, empty when false); not stored or parsed; ST-180 [FR-041] |
| ERR-009 | PARSE_FAILED | no | recorded on the `pages` row (parse_ok=false, reason); never a FetchResult error_class — parsing is post-fetch [DOC-10 §4] |
| ERR-010 | ROBOTS_DEFERRED | n/a | host-level for page dispatches: no URL state change, the Host defers [DOC-08 §2.3]. URL-level only as the redirect-hop abort class [R-131] |
| ERR-011 | REDIRECT_LOOP / TOO_MANY_REDIRECTS | no | also covers 3xx responses with a missing or unparsable `Location`, and hop targets that are not acceptable absolute http(s) URLs after RFC 3986 §5 resolution (e.g. `ftp:`, `data:`, userinfo) [R-130], [DOC-09 §4] |
| ERR-012 | TIMEOUT_HEADERS / TIMEOUT_TOTAL | yes | |
| ERR-013 | DECODE_FAILED | yes (once) | [R-140] |
| ERR-014 | HTTP_4XX_PERMANENT | no | non-retryable client errors (401/403/404/410/418/451/other 4xx); the specific status is recorded in fetch_events.http_status [DOC-09 §4] |
| ERR-015 | REDIRECT_OUT_OF_SCOPE | no | redirect target failed the Scope predicate [R-030]; source URL → ST-180, target never fetched |
| ERR-016 | HEADER_TOO_LARGE | no | response header block exceeds 64 KiB [DOC-16 §3] |
| ERR-017 | REDIRECT_ROBOTS_DISALLOWED | no | redirect hop target failed the target Host's robots gate [R-131]; source URL → ST-180, target never fetched |
| ERR-018 | HOP_RATE_LIMITED | yes | redirect hop whose target Host politeness window would delay the send beyond [CFG-035] (e.g. adversarial Crawl-delay); chain aborts, source → ST-150, target window opening is an unclamped floor on next_attempt_mono [R-131] |
| ERR-019 | REDIRECT_BLOCKLISTED | no | redirect hop target matches the [CFG-037] URL blocklist; chain terminates at that hop, source → ST-180, target never fetched, hop recorded in `redirect_chain` [R-131] — the blocklist is not bypassable by redirects |

## 2. Retry state

Per URL Record: `attempts` (count of started fetch attempts — incremented
inside the dispatch transaction [T-1], so an attempt interrupted by a crash
still counts [R-053]), `next_attempt_mono`.
Attempt counting includes the initial attempt. Retry budget [CFG-020] = total
attempts allowed.

## 3. Retry decision procedure (normative)

```
on RETRYABLE outcome:
  if attempts ≥ CFG-020:            → ST-180 (DEAD), error_class recorded
  else:
    delay = CFG-022 × CFG-023^(attempts−1)
    delay = delay × (1 + U(−1, +1)×CFG-024)     // full jitter band, seeded by
                                                 // hash(url_identity, attempt) [NFR-006]
    delay = min(delay, CFG-035)                 // clamp applies to the computed backoff only
    delay = max(delay, Retry-After if present)  // Retry-After — or, for ERR-018
                                                 // hop aborts, the target Host's window
                                                 // opening [R-131] — is a floor and is
                                                 // never clamped; harmonized with the
                                                 // host-level rule [R-111]
    state → ST-150, next_attempt_mono = now + delay
on PERMANENT outcome:               → ST-180 immediately
```

- R-230: Backoff is per-URL; host-level dynamic backoff ([DOC-08 §4]) applies
  additionally and independently at scheduling time.
- R-232: Classes marked "yes (once)" (ERR-003, ERR-013) have an effective
  retry budget of 2 total attempts (initial + 1 retry), regardless of
  [CFG-020]; further occurrences are permanent.
- R-231: Any successful page fetch resets host `consecutive_failures` to 0; the full increment/reset semantics are defined by [R-112] (robots.txt exchanges never modify it). The URL `attempts` counter resets to 0 only when the record leaves ST-140 for ST-100 (recrawl [R-052] or rediscovery refresh [FR-051]); it is never reset by a retry-path success.

## 4. Dead-letter semantics

ST-180 records retain last error class and are excluded from scheduling forever,
except: an operator MAY reset a URL to ST-100 via the runtime API (explicit,
audited action; the reset clears `attempts` to 0, clears the last error
class, and recomputes priority per [DOC-12 §2]). Automated re-activation of
DEAD URLs is forbidden.

## 5. Crash classification

Any crash mid-fetch is equivalent to a RETRYABLE outcome for that URL
(attempt already counted at dispatch [T-1]); recovery path [R-060].
