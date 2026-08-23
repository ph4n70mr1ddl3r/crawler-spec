---
id: DOC-13
title: Error Taxonomy and Retry Policy
version: 1.1.0
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
| ERR-008 | UNSUPPORTED_TYPE | no | recorded, not fetched again this run |
| ERR-009 | PARSE_FAILED | no | page stored with parse_ok=false |
| ERR-010 | ROBOTS_DEFERRED | n/a | host-level, not URL-level [DOC-08 §2.3] |
| ERR-011 | REDIRECT_LOOP / TOO_MANY_REDIRECTS | no | |
| ERR-012 | TIMEOUT_HEADERS / TIMEOUT_TOTAL | yes | |
| ERR-013 | DECODE_FAILED | yes (once) | [R-140] |

## 2. Retry state

Per URL Record: `attempts` (count of completed fetch attempts), `next_attempt_mono`.
Attempt counting includes the initial attempt. Retry budget [CFG-020] = total
attempts allowed.

## 3. Retry decision procedure (normative)

```
on RETRYABLE outcome:
  if attempts ≥ CFG-020:            → ST-180 (DEAD), error_class recorded
  else:
    delay = CFG-022 × CFG-023^(attempts−1)
    delay = delay × (1 + U(−j, +j)×CFG-024)     // full jitter band, seeded [NFR-006]
    delay = max(delay, Retry-After if present)
    delay = min(delay, CFG-035)
    state → ST-150, next_attempt_mono = now + delay
on PERMANENT outcome:               → ST-180 immediately
```

- R-230: Backoff is per-URL; host-level dynamic backoff ([DOC-08 §4]) applies
  additionally and independently at scheduling time.
- R-231: Any successful fetch resets host `consecutive_failures` to 0. The URL `attempts` counter resets to 0 only on terminal success (ST-140) and on recrawl [R-052]; it is never reset by a retry-path success.

## 4. Dead-letter semantics

ST-180 records retain last error class and are excluded from scheduling forever,
except: an operator MAY reset a URL to ST-100 via the runtime API (explicit,
audited action). Automated re-activation of DEAD URLs is forbidden.

## 5. Crash classification

Any crash mid-fetch is equivalent to a RETRYABLE outcome for that URL
(attempt already counted at dispatch [T-1]); recovery path [R-060].
