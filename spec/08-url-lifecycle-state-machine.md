---
id: DOC-07
title: URL Lifecycle State Machine
version: 1.18.0
---

# URL Lifecycle State Machine

## 1. States

| ID | Name | Terminal? | Meaning |
|----|------|-----------|---------|
| ST-100 | QUEUED | no | Passed filters; awaiting scheduling. |
| ST-110 | SCHEDULED | no | Dispatched; holding its global unit and at most one Host unit [R-051]. |
| ST-120 | FETCHING | no | HTTP request in flight. |
| ST-130 | FETCHED | no | Payload stored; extraction pending. |
| ST-140 | DONE | yes | Parsed, extracted, artifacts stored. |
| ST-150 | RETRY_WAIT | no | Retryable failure; backoff timer running. |
| ST-180 | DEAD | yes | Retries exhausted or permanent error. |
| ST-190 | EXCLUDED | yes | Never crawled; reason code required. |

Reason codes for ST-190: `OUT_OF_SCOPE`, `ROBOTS_DISALLOW`, `ROBOTS_UNKNOWN_TIMEOUT`,
`TRAP_PARAM`, `TRAP_PATH_BUDGET`, `DEPTH_LIMIT`, `CAP_REACHED`, `BLOCKLIST`.

*Terminal* in the table above means the record's current fetch cycle is
complete and has no failure-path exit. ST-140 is terminal with respect to its
fetch cycle: it leaves ST-140 only via the two ST-100 re-entry paths —
recrawl [FR-050] and rediscovery refresh [FR-051] (the records [FR-051]
calls "terminal-success") — or via the [R-062] upsert, which moves it
directly to a new fetch-outcome state (ST-130/ST-150/ST-180) without passing
through ST-100. Where a rule means {ST-180, ST-190} by `terminal`
(e.g. [R-062]), it names the states explicitly.

## 2. Transitions

```
(creation)          ─► ST-100   filters passed [FR-003]
(creation)          ─► ST-190   scope/trap/blocklist filter failed [FR-004],
                                [DOC-06 §5], or enqueue-time cap reached [FR-005]
(creation)          ─► ST-130/ST-150/ST-180
                                redirect final-target upsert [R-062]: the target
                                record is created at chain completion directly
                                in the fetch-outcome state (no ST-100 phase —
                                every per-hop gate was verified in flight [R-131])
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
ST-110 ─► ST-180                crash recovery reclassification with the
                                retry budget exhausted [R-060], [DOC-13 §5]
                                (likewise applies from ST-120 identically;
                                subsumed there by the edge below)
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
                                (audited; attempts := 0, due_at_mono := now,
                                last error class and any stale
                                next_attempt_mono cleared, priority
                                recomputed per [DOC-12 §2])
{ST-100, ST-130, ST-140, ST-150,
 ST-180, ST-190} ─► ST-130/ST-150/ST-180
                                redirect final-target upsert onto a
                                pre-existing record [R-062]: the fetch-outcome
                                state replaces the record's current state (the
                                same rule that creates new records directly in
                                the outcome states above). Never from
                                {ST-110, ST-120} — an independent fetch in
                                flight is untouched by the upsert. A same-state
                                landing (ST-150→ST-150, ST-180→ST-180,
                                ST-190→ST-190) is not a transition: no
                                metric is emitted for it.
```

No other transitions exist. Any observed other transition is a defect.

Note on the robots paths: the robots gate [FR-011(d)] is normally evaluated
during dispatch selection, while the record is ST-100 (ST-100→ST-190). The
ST-110 paths cover verdicts that change after the dispatch transaction [T-1]
but before the request is sent (e.g. a robots cache refresh between commit
and send): DISALLOW ⇒ ST-110→ST-190; UNKNOWN (Host deferred) ⇒ compensated
back to ST-100 [R-054]. In every case the held units are released
exactly once [R-051].

## 3. State-dependent rules

- R-050: Only ST-100 records are visible to the Scheduler.
- R-051: ST-110/ST-120 records own one global unit each, held from [T-1]
  until the first transition out of {ST-110, ST-120}, plus at most one
  per-Host unit: the Host of their most recent request (initially the
  identity's Host via [T-1]). Host units transfer at redirect hops per
  [R-131]: the previous Host's unit is released when its response is
  received, and the next Host's unit is acquired immediately before the next
  request is sent — so a task never holds two Host units and holds none
  while waiting (politeness or capacity waits therefore cannot deadlock
  [G-4]). All units the record still holds are released exactly once, on
  the first transition out of {ST-110, ST-120} (to ST-100, ST-130, ST-150,
  ST-180, or ST-190 — including the robots paths [FR-031], [R-054]).
  Exclusions that fire on an ST-100 candidate before [T-1] — the
  dispatch-time cap gate [FR-011(e)], selection-time robots exclusions
  ([FR-031] at gate time, [R-103]) — release nothing: the record holds no
  units before [T-1] (decrementing there would corrupt the counters
  [R-212], [INV-3]).
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
    with unit release [R-051];
  - changed to UNKNOWN (Host deferred) ⇒ the dispatch is compensated: the
    record returns to ST-100 with `due_at_mono` := now (immediately due;
    the UNKNOWN verdict suppresses candidacy until the verdict changes),
    the units are released [R-051], and the
    record is reconsidered after deferral expiry [R-103] — at the
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
  Host rows are rebuilt in the same recovery pass: `inflight` is reset to 0
  for every Host — exact, not merely conservative: every {ST-110, ST-120}
  record has just been reclassified out of the unit-owning states [R-051],
  and robots exchanges in flight at crash died with the process; under
  [R-051] unit transfers a crashed record's Host unit sat on the Host of its
  most recent request, which is not recoverable from persisted state, so 0
  (the count of unit-owning records) is the only sound rebuild — restoring
  INV-3 [NFR-011].
- R-061: ST-130 records at startup resume extraction (idempotent: re-running
  extraction overwrites artifacts deterministically).
- R-062: On completion of a redirect chain, the system MUST upsert a URL Record
  for the chain's final target identity, with depth equal to the source record's
  depth (redirects do not count toward depth [DEC-009]),
  `discovered_from` = source identity, and state per the fetch outcome
  (ST-130 on success, then normal progression; ST-150/ST-180 on failure).
  The upsert applies only when the chain reached its final target — the
  final request was actually sent (a final response arrived, or a
  transport failure was classified against it). A chain aborted before
  sending to its would-be next hop (scope [ERR-015], robots [ERR-017],
  URL blocklist [ERR-019], SSRF [ERR-004], robots deferral [ERR-010], an
  over-threshold politeness wait [ERR-018], or loop/cap exhaustion
  [ERR-011]) has no fetched final target: the outcome is recorded on the
  source record only, and the unfetched hop URL receives no URL Record
  (each hop rule's "target never fetched"). The upsert never modifies
  the target's `attempts` (the attempt is accounted on the source record
  [T-1]). Intermediate hops are persisted on
  the source's `redirect_chain` [R-133] and receive no URL Records. The
  final target is exempt from the trap filters ([DOC-06 §5] items 2–4) at
  this creation point (it was already fetched); scope, robots, SSRF, and
  the URL blocklist were verified per hop [R-030], [R-131].

  On chain success, [T-2] commits two `pages` rows in the same transaction:
  the source's (with `final_url_identity` = the target identity [FR-044]) and
  the target's (with `final_url_identity` = itself); both records move to
  ST-130 and progress to ST-140 via their own extraction passes — identical
  payload and resolution base ⇒ identical artifacts, upsert-idempotent
  [R-157].

  If a URL Record for the target identity already exists, the creation-time
  fields — `depth`, `discovered_from`, `source_run_id`, `raw_first_seen`,
  `is_seed` — are NOT rewritten: a pre-existing record keeps its own
  provenance and seed fields, and the upsert's field effects are exactly
  those this rule and [T-2] name (state, `last_seen_at`,
  `last_error_class`, the `exclude_reason`/`next_attempt_mono` clearing,
  `once_retried_classes`, and the completion-time fields [T-2] owns).
  The upsert's state effect depends on that record's state. Terminal records (ST-180, ST-190)
  are overwritten with the fetch outcome: this records a completed,
  gate-verified fetch (scope, robots, SSRF, and blocklist were re-checked per
  hop [R-131]) and is neither the forbidden automated re-activation of
  [DOC-13 §4] nor a rediscovery under [FR-051]. A record in {ST-110, ST-120} — an independent fetch in
  flight — MUST NOT have its state or `attempts` modified (overwriting it
  would corrupt its unit accounting [R-051]): the chain records `last_seen_at`
  and its `pages` row only, and the concurrent attempt's own [T-2] governs
  the record. Any other pre-existing record (ST-100, ST-130, ST-140, ST-150)
  takes the fetch-outcome state as above. An upsert that moves a record out
  of ST-190 clears `exclude_reason` (the exclusion no longer applies — the
  column is defined only for ST-190 [DOC-11 §1]); a landing outside ST-150
  clears `next_attempt_mono` (a stale backoff timer must not survive into
  ST-130/ST-180).

  Failure upserts set `last_error_class` to the outcome's class; success
  upserts clear it (mirroring [DOC-13 §3]'s accounting, where only DEAD and
  PERMANENT outcomes leave a durable class). A class set by a RETRYABLE
  upsert is transient: the target's own next successful completion clears it
  [T-2], so a record recovered by its own fetch never carries a stale failure
  class. A RETRYABLE-outcome upsert
  lands in ST-150 — with `next_attempt_mono` mirroring the source's (one
  chain, one retry schedule: the source's next attempt re-runs the chain
  [R-131] and re-upserts) — only when the record's `attempts` <
  [CFG-020]; a record whose own budget is already exhausted (e.g. a
  budget-exhausted DEAD record overwritten by the chain) takes ST-180
  instead. Without this guard the upsert could strand a record in ST-150
  forever: the ST-150→ST-100 promotion requires `attempts` < [CFG-020]
  [DOC-12 §1], and the defensive ST-150→ST-180 edge is otherwise
  unreachable [§2] — so no ST-150 record ever holds `attempts` ≥
  [CFG-020], preserving the invariant asserted by [DOC-12 §1].

  Yes-once classes ([R-232]) are accounted on the target exactly as on the
  source — one chain, one schedule: a RETRYABLE outcome whose class is
  ERR-003 or ERR-013 is evaluated against the target's own
  `once_retried_classes`; when the class is not yet listed there it is
  appended (mirroring the source's bookkeeping), and when it is already
  listed the outcome is PERMANENT for the target (ST-180,
  `last_error_class` := that class), independently of where the source
  record lands. Success upserts leave the list to the normal clearing rule
  ([R-232]: cleared when `attempts` resets to 0).
