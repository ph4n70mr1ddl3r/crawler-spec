---
id: DOC-13
title: Error Taxonomy and Retry Policy
version: 1.33.0
---

# Errors and Retry

## 1. Error classes

| ID | Name | Retryable | Notes |
|----|------|-----------|-------|
| ERR-001 | DNS_FAILURE | yes, except NXDOMAIN | transient DNS resolution failures (timeouts [CFG-036], SERVFAIL, etc. — mapped from DNS timeouts [DOC-09 §2]) are RETRYABLE under the ordinary [§3] procedure: ST-150 with a backoff `next_attempt_mono` when `attempts < [CFG-020]`, or ST-180 with `next_attempt_mono=NULL` when the budget is exhausted; an NXDOMAIN (definitive non-existent domain) is PERMANENT — outcome PERMANENT, error_class ERR-001, source → ST-180 immediately with `urls.last_error_class='ERR-001'`, no retry and no backoff timer — per the security-level rule [R-403] (highest precedence [R-000]); the Host's `consecutive_failures` is incremented only for the transient retryable case, not for NXDOMAIN [R-112] |
| ERR-002 | CONNECT_TIMEOUT / CONN_REFUSED / NET_UNREACH | yes | |
| ERR-003 | TLS_ERROR | yes (once), except cert verification failure | a transient TLS handshake failure (e.g. a handshake timeout [CFG-013], mapped from TLS timeouts [DOC-09 §2]) is RETRYABLE at most once per attempt cycle under the yes-once gate [R-232] and the ordinary [§3] procedure: the first occurrence appends ERR-003 to `urls.once_retried_classes` and lands ST-150 with a backoff `next_attempt_mono` when `attempts < [CFG-020]` (ST-180 with `next_attempt_mono=NULL` at budget exhaustion), and a second occurrence is PERMANENT/ST-180 with `urls.last_error_class='ERR-003'` even below budget [AC-095]; a certificate verification failure (untrusted, expired, or wrong-host certificate — verification is mandatory and fail-closed [DOC-09 §1], [DOC-16 §2]) is PERMANENT immediately — outcome PERMANENT, error_class ERR-003, source → ST-180 with `urls.last_error_class='ERR-003'`, no retry and no backoff timer [§4], no `once_retried_classes` append (a PERMANENT outcome never enters the [§3] RETRYABLE branch) — since a deterministically invalid certificate cannot be repaired by retrying; the Host's `consecutive_failures` is incremented only for the transient retryable case, not for a cert verification failure [R-112], [R-231] |
| ERR-004 | SSRF_BLOCKED | no | security; log at WARN [DOC-16] |
| ERR-005 | HTTP_429 | yes | honor Retry-After [R-111] |
| ERR-006 | HTTP_5XX | yes | |
| ERR-007 | PAYLOAD_TOO_LARGE | no | [FR-023] |
| ERR-008 | UNSUPPORTED_TYPE | no | content type outside the effective allowed list ([R-143]: fixed set when [CFG-028]=true, empty when false); not stored or parsed; ST-180 [FR-041] |
| ERR-009 | PARSE_FAILED | no | recorded on the `pages` row (parse_ok=false, reason); never a FetchResult error_class — parsing is post-fetch [DOC-10 §4] |
| ERR-010 | ROBOTS_DEFERRED | host-level: n/a; URL-level: yes | host-level for page dispatches: no URL state change, the Host defers [DOC-08 §2.3] — no URL-level retry decision applies (hence `n/a` for that case). URL-level only as the redirect-hop abort class [R-131]: the abort is a RETRYABLE outcome governed by the ordinary [§3] retry procedure — source → ST-150 with a backoff `next_attempt_mono` when `attempts < [CFG-020]`, or ST-180 with `next_attempt_mono=NULL` when the budget is exhausted; no request is sent to the target Host during deferral, and the next attempt re-runs the chain [AC-017], [AC-070] |
| ERR-011 | REDIRECT_LOOP / TOO_MANY_REDIRECTS | no | also covers 3xx responses with a missing or unparsable `Location`, and hop targets that are not acceptable absolute http(s) URLs after RFC 3986 §5 resolution (e.g. `ftp:`, `data:`, userinfo) [R-130], [DOC-09 §4] |
| ERR-012 | TIMEOUT_HEADERS / TIMEOUT_TOTAL | yes | |
| ERR-013 | DECODE_FAILED | yes (once) | [R-140] |
| ERR-014 | HTTP_PERMANENT | no | non-retryable permanent HTTP responses: 4xx client errors (401/403/404/410/418/451/other 4xx) and the double-304 case where an unconditional refetch of a 304 with no usable stored payload also returns 304 [R-144]; the specific status (including 304) is recorded in fetch_events.http_status [DOC-09 §4] |
| ERR-015 | REDIRECT_OUT_OF_SCOPE | no | redirect target failed the Scope predicate [R-030]; source URL → ST-180, target never fetched |
| ERR-016 | HEADER_TOO_LARGE | no | response header block exceeds 64 KiB [DOC-16 §3] |
| ERR-017 | REDIRECT_ROBOTS_DISALLOWED | no | redirect hop target failed the target Host's robots gate [R-131]; source URL → ST-180, target never fetched |
| ERR-018 | HOP_RATE_LIMITED | yes | redirect hop whose target Host politeness window would delay the send beyond [CFG-035] (e.g. adversarial Crawl-delay); chain aborts [R-131]; source → ST-150 with `next_attempt_mono` floored at the target Host's window opening **only when `attempts < CFG-020`** — a budget-exhausted abort lands in ST-180 with `next_attempt_mono=NULL`, since the floor never applies to a DEAD record [§3], [AC-085] |
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
  if outcome class ∈ {ERR-003, ERR-013}:   // "yes (once)" classes [R-232] —
                                          // evaluated FIRST, before the budget check
      if class ∈ urls.once_retried_classes: // already retried once this attempt cycle
          → ST-180 (DEAD) — outcome is PERMANENT, error_class := class
      else:
          append class to urls.once_retried_classes  // persisted by [T-2]; fall through
  if attempts ≥ CFG-020:            → ST-180 (DEAD), error_class recorded
  else:
    delay = CFG-022 × CFG-023^(attempts−1)
    delay = delay × (1 + U(−1, +1)×CFG-024)     // full jitter band, seeded by
                                                 // hash(url_identity, attempt) [NFR-006]
    delay = min(delay, CFG-035)                 // clamp applies to the computed backoff only
    if Retry-After present: delay = max(delay, Retry-After as a delta)  // never clamped;
                                                 // harmonized with the host-level rule [R-111]
    state → ST-150, next_attempt_mono = now + delay
    if outcome is ERR-018:                  // hop-abort: the target Host's
        next_attempt_mono = max(next_attempt_mono, target_host.next_allowed_fetch_at)
                                                  // monotonic instant floor; the
                                                  // politeness window opening acts as
                                                  // an absolute floor on the ST-150
                                                  // landing only — never on a
                                                  // budget-exhausted ST-180 landing
on PERMANENT outcome:               → ST-180 immediately
                                     (`urls.last_error_class` recorded)
```

The yes-once-classes gate [R-232] is evaluated before the [CFG-020] budget
check, identically to the redirect final-target upsert path [R-062], [AC-093]:
a repeat ERR-003/ERR-013 is PERMANENT/ST-180 even when `attempts` < [CFG-020],
and a first occurrence appends the class and then falls through to the ordinary
budget/backoff path. The append and the class written on the DEAD landing are
persisted by [T-2]; the list is cleared exactly when `attempts` resets to 0
[R-232]. This procedure governs every ordinary retry landing (FR-052); the
upsert path applies the same precedence via [R-062].

- R-230: Backoff is per-URL; host-level dynamic backoff ([DOC-08 §4]) applies
  additionally and independently at scheduling time.
- R-232: Classes marked "yes (once)" (ERR-003, ERR-013) are retryable at
  most once per attempt cycle and per class: when such an outcome occurs and
  its class is already listed in `urls.once_retried_classes` [DOC-11 §1], the
  outcome is PERMANENT (ST-180, `last_error_class` := that class); otherwise
  the class is appended to the list and the normal §3 retry procedure
  applies. This check is the first gate of the §3 procedure, evaluated before
  the [CFG-020] budget check on both ordinary retry landings [§3] and
  redirect final-target upserts [R-062], [AC-093]; the general budget still
  applies independently once the class check passes. The list is cleared
  exactly when `attempts` is reset to 0 (ST-140→ST-100 [R-052]; operator
  reset §4), so the rule needs no fetch_events history and cannot be affected
  by [CFG-033] retention.
- R-231: Any successful page fetch resets host `consecutive_failures` to 0; the full increment/reset semantics are defined by [R-112] (robots.txt exchanges never modify it). The URL `attempts` counter resets to 0 only when the record leaves ST-140 for ST-100 (recrawl [R-052] or rediscovery refresh [FR-051]) or is operator-reset to ST-100 from ST-180 [DOC-13 §4]; it is never reset by a retry-path success.

## 4. Dead-letter semantics

ST-180 records retain last error class and are excluded from scheduling forever,
except: an operator MAY reset a URL to ST-100 via the runtime API (explicit,
audited action; the reset clears `attempts` to 0 (and with it
`once_retried_classes`, [R-232]), sets `due_at_mono` := now (the reset URL is
immediately due), clears the last error class and any stale `next_attempt_mono`
(a backoff timer must not survive outside ST-150 — the hygiene [R-062]
establishes for upsert landings), and recomputes priority per
[DOC-12 §2]). Automated re-activation of DEAD URLs is forbidden.

## 5. Crash classification

Any crash mid-fetch is equivalent to a RETRYABLE outcome for that URL
(attempt already counted at dispatch [T-1]); the reclassification is
applied at startup by [R-060]: attempts ≥ [CFG-020] ⇒ ST-180 (with
`last_error_class` = NULL — the crash wrote no fetch_event, and no class is
fabricated), else ST-150 with backoff per §3 computed as of restart time.
