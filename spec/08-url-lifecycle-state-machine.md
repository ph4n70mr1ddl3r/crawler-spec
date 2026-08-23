---
id: DOC-07
title: URL Lifecycle State Machine
version: 1.8.0
---

# URL Lifecycle State Machine

## 1. States

| ID | Name | Terminal? | Meaning |
|----|------|-----------|---------|
| ST-100 | QUEUED | no | Passed filters; awaiting scheduling. |
| ST-110 | SCHEDULED | no | Dispatched; holding politeness slot. |
| ST-120 | FETCHING | no | HTTP request in flight. |
| ST-130 | FETCHED | no | Payload stored; extraction pending. |
| ST-140 | DONE | yes | Parsed, extracted, artifacts stored. |
| ST-150 | RETRY_WAIT | no | Retryable failure; backoff timer running. |
| ST-180 | DEAD | yes | Retries exhausted or permanent error. |
| ST-190 | EXCLUDED | yes | Never crawled; reason code required. |

Reason codes for ST-190: `OUT_OF_SCOPE`, `ROBOTS_DISALLOW`, `ROBOTS_UNKNOWN_TIMEOUT`,
`TRAP_PARAM`, `TRAP_PATH_BUDGET`, `DEPTH_LIMIT`, `CAP_REACHED`, `BLOCKLIST`.

## 2. Transitions

```
(creation)          ─► ST-100   filters passed [FR-003]
(creation)          ─► ST-190   scope/trap/blocklist filter failed [FR-004],
                                [DOC-06 §5], or enqueue-time cap reached [FR-005]
ST-100 ─► ST-110                scheduler dispatch [FR-012], atomic
ST-100 ─► ST-190                robots DISALLOW at gate time [FR-031],
                                dispatch-time domain cap [FR-011(e)]
                                (`CAP_REACHED`), or robots-unknown
                                threshold [R-103] (`ROBOTS_UNKNOWN_TIMEOUT`)
ST-110 ─► ST-120                robots ALLOW confirmed, request sent
ST-110 ─► ST-190                robots DISALLOW at send-time re-check [R-054]
ST-110 ─► ST-100                send-time robots UNKNOWN (deferral) compensation [R-054]
ST-120 ─► ST-130                success — 2xx with payload stored [FR-043],
                                or 304 UNCHANGED reusing the existing payload
                                [R-121], [R-144]
ST-110 ─► ST-150                crash recovery reclassification [R-060],
                                [DOC-13 §5] (applies from ST-120
                                identically; request-sent status unknown,
                                conservatively retryable)
ST-120 ─► ST-150                retryable failure [DOC-13 §3]
ST-120 ─► ST-180                permanent failure or retry budget exhausted
ST-150 ─► ST-100                backoff elapsed, attempts < [CFG-020]
                                (due_at_mono := next_attempt_mono)
ST-150 ─► ST-180                attempts = [CFG-020]
                                (defensive: unreachable in practice — the
                                budget is evaluated at failure time
                                [DOC-13 §3], so ST-150 records always hold
                                attempts < [CFG-020])
ST-150 ─► ST-190                host robots-unknown timeout [R-103]
ST-130 ─► ST-140                extraction complete
ST-140 ─► ST-100                recrawl due [FR-050], or rediscovery refresh
                                [FR-051] (attempts reset to 0)
ST-180 ─► ST-100                operator reset via runtime API [DOC-13 §4]
                                (audited; attempts := 0, last error class cleared)
```

No other transitions exist. Any observed other transition is a defect.

Note on the robots paths: the robots gate [FR-011(d)] is normally evaluated
during dispatch selection, while the record is ST-100 (ST-100→ST-190). The
ST-110 paths cover verdicts that change after the dispatch transaction [T-1]
but before the request is sent (e.g. a robots cache refresh between commit
and send): DISALLOW ⇒ ST-110→ST-190; UNKNOWN (Host deferred) ⇒ compensated
back to ST-100 [R-054]. In every case the held concurrency slot is released
exactly once [R-051].

## 3. State-dependent rules

- R-050: Only ST-100 records are visible to the Scheduler.
- R-051: ST-110/ST-120 records own one inflight unit each against host/global caps; caps are released exactly once, on the first transition out of {ST-110, ST-120} (to ST-100, ST-130, ST-150, ST-180, or ST-190 — including the robots paths [FR-031], [R-054] and the dispatch-time cap gate [FR-011(e)]).
- R-052: ST-140→ST-100 transitions (recrawl due [FR-050] and rediscovery
  refresh [FR-051]) reset `attempts=0` and set `due_at_mono` per their
  trigger; priority is recomputed per [DOC-12 §2] (R-201).
- R-053: `attempts` is incremented exactly once per fetch attempt, inside the
  dispatch transaction [FR-012], [T-1]; it is therefore already counted when a
  crash mid-fetch is classified as retryable [DOC-13 §5]. Exception: the
  send-time robots re-check aborts [R-054] roll the increment back — no
  request was sent, so no attempt occurred.
- R-054: Send-time robots re-check [DOC-08 §3]: if the applicable robots
  verdict for a dispatched URL changed after [T-1] and before the request is
  sent, the dispatch is aborted — no request is sent, hence no fetch_event
  [FR-025] and no attempt ([R-053]; the [T-1] increment is rolled back in
  both cases below):
  - changed to DISALLOW ⇒ the record moves ST-110→ST-190/`ROBOTS_DISALLOW`
    with slot release [R-051];
  - changed to UNKNOWN (Host deferred) ⇒ the dispatch is compensated: the
    record returns to ST-100, the concurrency slot is released [R-051], and
    the record is reconsidered after deferral expiry [R-103] — at the
    [CFG-040] threshold it is excluded `ROBOTS_UNKNOWN_TIMEOUT` like any
    gated record.
  The politeness advance of [T-1] is never rolled back in either case (the
  robots exchange consumed the window).

## 4. Recovery and redirect completion

- R-060: On startup, every record in {ST-110, ST-120} is reclassified per
  the crash rule [DOC-13 §5]: the attempt was already counted at dispatch
  [T-1] and whether the request was sent is unknowable, so the outcome is
  conservatively RETRYABLE — attempts ≥ [CFG-020] ⇒ ST-180 (DEAD,
  `last_error_class` := NULL: the crash wrote no fetch_event and no ERR-nnn
  class is fabricated); otherwise ST-150 with `next_attempt_mono` =
  restart time + backoff computed per [DOC-13 §3] (jitter seeded by
  hash(url_identity, attempt), as for any retry). Resetting directly to
  ST-100 would (a) exceed the Retry Budget — a record whose final allowed
  attempt was in flight at crash time would be re-dispatched — and (b)
  retry immediately with no per-URL backoff, contradicting [DOC-13 §5].
  Because dispatch advanced `next_allowed_fetch_at` transactionally
  [FR-012], politeness cannot be violated by this reclassification.
  Host rows are rebuilt conservatively in the same recovery pass: `inflight` is
  recomputed as the count of records still in {ST-110, ST-120} after the
  reclassification (i.e., 0 for a clean recovery), restoring INV-3 [NFR-011].
- R-061: ST-130 records at startup resume extraction (idempotent: re-running
  extraction overwrites artifacts deterministically).
- R-062: On completion of a redirect chain, the system MUST upsert a URL Record
  for the chain's final target identity, with depth equal to the source record's
  depth (redirects do not count toward depth [DEC-009]),
  `discovered_from` = source identity, and state per the fetch outcome
  (ST-130 on success, then normal progression; ST-150/ST-180 on failure).
  The upsert never modifies the target's `attempts` (the attempt is
  accounted on the source record [T-1]). Intermediate hops are persisted on
  the source's `redirect_chain` [R-133] and receive no URL Records. The
  final target is exempt from the trap filters ([DOC-06 §5] items 2–4) at
  this creation point (it was already fetched); scope, robots, SSRF, and
  the URL blocklist were verified per hop [R-030], [R-131].
