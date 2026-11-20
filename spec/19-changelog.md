---
id: DOC-18
title: Changelog
version: 1.33.0
---

# Changelog

## Unreleased — 2026-11-20 (bug hunt: DOC-09 §4 304 row Class cell still blanket-labeled every 304 success-unchanged, contradicting the row's own PERMANENT/ERR-014 double-304 Action, and AC-108 test)

### Correctness fixes

- The bug hunts that qualified the [DOC-09 §4] 304 row's Action cell, [R-121]
  [DOC-08 §5], the [R-130] parenthetical [DOC-09 §3], the [DOC-13 §1]
  `ERR-014` definition, and the [DOC-07 §2] transition table for the [R-144]
  double-304 cache-miss outcome (adding [AC-101], [AC-104], [AC-107]) never
  touched the **Class** cell of the [DOC-09 §4] 304 row itself. That cell still
  read `success-unchanged` — a blanket outcome label false for the one 304
  outcome the spec mandates be PERMANENT: a 304 with no usable stored payload
  whose single permitted unconditional refetch [R-144] also returns 304 must
  land ST-180 with `error_class='ERR-014'` [AC-059], [AC-101]. The row is
  evaluated top-down and its Class column is the outcome class (compare the
  `permanent`, `retryable`, `per cause` and `— (never an outcome)` cells of the
  other rows, and the `FetchResult` outcome enum [DOC-09 §6]); a literal
  implementation reading that Class cell classified the double-304 as
  SUCCESS/UNCHANGED — reusing a payload hash that by that outcome's own premise
  does not exist, resetting the Host's `consecutive_failures` [R-112], [R-231]
  even though a PERMANENT outcome must leave it unchanged, and self-
  contradicting the very same row's Action cell and the [DOC-07 §2]
  ST-120→ST-180 permanent-failure edge. The Class cell now states the
  conditional outcome — UNCHANGED in the ordinary case, PERMANENT/`ERR-014` on
  the [R-144] double-304 cache miss — and the 304 cross-reference in the '3xx
  outside the followable set' row no longer calls it unconditionally the
  'success-unchanged row'. New [AC-108] pins the row's classification in both
  the ordinary UNCHANGED case and the double-304 PERMANENT case, including the
  host-counter side effect.

## Unreleased — 2026-11-20 (bug hunt: DOC-07 §2 transition table still sent every 304 to ST-130/UNCHANGED, contradicting the R-144 double-304 PERMANENT/ERR-014 landing, and AC-107 test)

### Correctness fixes

- The bug hunt that qualified [R-121] [DOC-08 §5] and the [DOC-09 §4] 304
  row for the [R-144] double-304 cache-miss outcome, and added [AC-104],
  fixed the normative honesty rule and the response-classification table but
  never touched the [DOC-07 §2] state-transition diagram. That diagram's
  ST-120→ST-130 success edge still read "or 304 UNCHANGED reusing the existing
  payload [R-121], [R-144]" — a blanket claim false for the one 304 outcome
  the spec mandates be PERMANENT. It sent every 304 to ST-130/UNCHANGED even
  when no usable stored payload exists (nothing to reuse), and mis-cited
  [R-144], the very rule that mandates a PERMANENT/`ERR-014` landing on a
  second 304, as authority for the UNCHANGED landing. A literal implementation
  of that edge returned UNCHANGED/ST-130 for a first-fetch 304 whose
  unconditional refetch [R-144] also returned 304 — reusing a payload hash
  that by the outcome's own premise does not exist, resetting the Host's
  `consecutive_failures` [R-112], [R-231] even though a PERMANENT outcome must
  leave it unchanged, and self-contradicting the same table's ST-120→ST-180
  permanent-failure edge [AC-059], [AC-101], [AC-104]. The edge now classifies
  a 304 exactly like [R-121] and the [DOC-09 §4] row: UNCHANGED/ST-130 only
  when a usable stored payload exists, otherwise an [R-144] cache miss whose
  second 304 is PERMANENT/`ERR-014` routed to ST-180 via the permanent-failure
  edge. New [AC-107] pins the transition table's agreement with the normative
  304 treatment in both the ordinary UNCHANGED case and the double-304
  PERMANENT case, including the host-counter side effect.

## Unreleased — 2026-11-20 (bug hunt: DOC-09 §4 429 and 5xx rows mandated unconditional ST-150, ignoring the retry-budget ST-180 landing, and AC-106 test)

### Correctness fixes

- The [DOC-09 §4] response-classification table states a landing state in
  every row, and the [R-131] `ERR-018` row was qualified in an earlier bug
  hunt so a budget-exhausted hop abort lands in ST-180 with a NULL timer
  rather than an unconditional ST-150. The two ordinary retryable rows were
  left unqualified: `429` read "honor `Retry-After` [R-111] → ST-150" and
  `5xx` read "→ ST-150, increments host `consecutive_failures`". Both
  contradict the normative [DOC-13 §3] procedure, under which any RETRYABLE
  outcome with `attempts` ≥ [CFG-020] lands in ST-180 (DEAD) with
  `urls.last_error_class` recorded, and [FR-052]/[AC-024], which pin the
  budget-exhausted ST-180 landing for `429`. With [CFG-020]=3, a third
  consecutive `429` or `5xx` following the table literally re-arms a backoff
  timer on a record whose `attempts` equals the budget, stranding it in ST-150
  (the ST-150→ST-100 promotion requires `attempts < [CFG-020]` [DOC-12 §1])
  and violating the "non-NULL only in ST-150" field hygiene [AC-085], while
  re-dispatching beyond the budget. Both rows now condition the ST-150 landing
  on `attempts < [CFG-020]` and name the budget-exhausted ST-180 landing (the
  `5xx` row retains the [R-112] host-counter increment on either landing,
  since the outcome stays RETRYABLE); new [AC-106] pins the table's agreement
  with [DOC-13 §3] and [AC-024] in both the within-budget and
  budget-exhausted cases.

## Unreleased — 2026-11-19 (bug hunt: R-121 said all 304 responses count as success, contradicting the R-144 double-304 PERMANENT/ERR-014 outcome, and AC-104 test)

### Correctness fixes

- The bug hunt that broadened `ERR-014` [DOC-13 §1] to cover the [R-144]
  double-304 cache-miss outcome, and added [AC-101], fixed the taxonomy row
  and the [DOC-09 §4] 304 row but never touched the normative honesty rule
  [DOC-08 §5], [R-121]. That rule still read "304 responses count as
  success with unchanged payload" — a blanket claim false for the one 304
  outcome the spec mandates be PERMANENT: a 304 with no usable stored
  payload whose single permitted unconditional refetch [R-144] also returns
  304 must land ST-180 with `error_class='ERR-014'` [AC-059], [AC-101]. A
  literal implementation of R-121 in the response classifier returned
  UNCHANGED/ST-130 for the second 304, reusing a payload hash that by the
  rule's own premise does not exist, and reset the Host's
  `consecutive_failures` [R-112], [R-231] even though a PERMANENT outcome
  must leave it unchanged — directly contradicting [R-144], [DOC-09 §4]
  and [AC-101]. The same blanket claim survived in the [DOC-09 §3]
  parenthetical of [R-130] ("a 304 is … classified earlier, as
  success-unchanged"). R-121 now distinguishes the two sub-cases — a 304
  with a usable stored payload is success-unchanged, while a 304 cache miss
  follows [R-144] to a PERMANENT/`ERR-014` landing on the second 304 — and
  states the [R-112] counter consequence; [R-130]'s parenthetical now points
  at both sub-cases instead of asserting that every 304 is
  success-unchanged. New [AC-104] pins both the ordinary UNCHANGED/304 case
  and the double-304 PERMANENT case, including the host-counter side effect.

- [AC-105] The `state_transitions_total` metric correctly distinguishes creation events from state-transition events involving ST-190 records: it emits `{from=creation,to=ST-190}` exclusively for ingestion-time ST-190 creation (scope/trap/blocklist filter failure [FR-004] or enqueue-time cap reached [FR-005]), and emits `{from=ST-100,to=ST-190}`, `{from=ST-110,to=ST-190}`, or `{from=ST-150,to=ST-190}` for later ST-190 state transitions (robots re-check [R-054], robots-unknown timeout sweep [R-103], dispatch-time domain-cap gate [FR-011(e)]). It emits no `{from=creation,to=ST-190}` for ST-190 record overwrites via [R-062] upserts (those overwrite existing records and are not record-creating transitions). An implementation that emits `{from=creation,to=ST-190}` for later transitions or upsert overwrites, or that fails to distinguish creation from transition, violates [AC-052] and [AC-098].

## Unreleased — 2026-11-19 (bug hunt: R-121 said all 304 responses count as success, contradicting the R-144 double-304 PERMANENT/ERR-014 outcome, and AC-104 test)

### Correctness fixes

- The bug hunt that added `[AC-102]` qualified the two `ERR-018`
  class-definition summaries — the `[DOC-13 §1]` taxonomy row and the
  `[DOC-09 §4]` classification row — so the hop-abort politeness floor on
  `next_attempt_mono` applies only to a RETRYABLE ST-150 landing and a
  budget-exhausted abort lands in ST-180 with a NULL timer. It did not touch
  the normative hop-dispatch rule itself: `[DOC-09 §3]`, `[R-131]` step 3
  (untouched since its introduction) still read "source URL → ST-150, with
  the target Host's window opening acting as an unclamped floor on
  `next_attempt_mono` `[DOC-13 §3]`". That clause is the source the two
  summaries were copied from, so the defect survived in the rule that
  [FR-021] and every redirect-hop implementation follow: with `[CFG-020]=1`
  an implementation following step 3 literally lands the budget-exhausted
  `ERR-018` abort in ST-150 with a floored, non-NULL backoff timer,
  contradicting `[DOC-13 §3]`, `[AC-085]`, `[AC-102]` and the "non-NULL only
  in ST-150" field-hygiene invariant `[DOC-12 §1]`, `[DOC-11 §1]`, and
  stranding a record whose `attempts` equals `[CFG-020]` in ST-150 (the
  ST-150→ST-100 promotion requires `attempts < [CFG-020]`). Step 3 now
  conditions the ST-150 landing and the window-opening floor on
  `attempts < [CFG-020]` and sends a budget-exhausted abort to ST-180 with
  `next_attempt_mono=NULL`, and new `[AC-103]` pins the step-3 narration's
  agreement with `[DOC-13 §3]` in both the within-budget and
  budget-exhausted cases.

## Unreleased — 2026-11-19 (bug hunt: ERR-018 class summaries still mandated unconditional ST-150 and a DEAD-record backoff floor, and AC-102 test)

### Correctness fixes

- The earlier bug hunt that fixed the `ERR-018` hop-abort floor (changelog
  entry "ERR-018 backoff floor on ST-180") re-indented the `[DOC-13 §3]`
  pseudocode so the `max(next_attempt_mono,
  target_host.next_allowed_fetch_at)` floor applies only inside the `else:`
  (ST-150) branch, and added `[AC-085]` to assert a budget-exhausted
  `ERR-018` abort reaches ST-180 with a NULL `next_attempt_mono`. That fix
  never touched the two class-definition summaries: the `[DOC-13 §1]`
  taxonomy row still read "chain aborts, source → ST-150, target window
  opening is an unclamped floor on `next_attempt_mono`", and the `[DOC-09
  §4]` classification row still read "source → ST-150, window opening floors
  `next_attempt_mono`". Both therefore described `ERR-018` as an
  unconditional ST-150 landing with an unconditional floor, contradicting
  `[DOC-13 §3]`, `[AC-085]`, and the "non-NULL only in ST-150" field-hygiene
  invariant `[DOC-13 §4]`, `[R-062]`, `[DOC-12 §1]`, `[DOC-11 §1]`: with
  `[CFG-020]=1` an implementation following either summary would write a
  backoff timer onto a DEAD ST-180 record. Both rows are now qualified so the
  floor applies only when `attempts < [CFG-020]` and a budget-exhausted abort
  lands in ST-180 with `next_attempt_mono=NULL`, and new `[AC-102]` pins the
  agreement of the two summaries with the `[DOC-13 §3]` procedure in both the
  within-budget and budget-exhausted cases.

## Unreleased — 2026-11-19 (bug hunt: ERR-014 defined as 4xx-only but mandated for the double-304 outcome, and AC-101 test)

### Correctness fixes

- [DOC-13 §1] defined `ERR-014` as `HTTP_4XX_PERMANENT`, 'non-retryable
  client errors (401/403/404/410/418/451/other 4xx)'. But [R-144] — added
  later — mandates `ERR-014` for a different case: when a 304 with no usable
  stored payload is refetched unconditionally and that refetch also returns
  304, the outcome is PERMANENT with `error_class=ERR-014`. A 304 is a 3xx,
  and the [DOC-09 §4] classification table explicitly says 'a 304 is not
  this case' for the permanent-3xx row, routing it to the success-unchanged
  row instead. So the only normative definition of `ERR-014` excluded the
  one 3xx outcome for which it is the required class: an implementation
  following the taxonomy would refuse to record `ERR-014` for the
  double-304 and fail [AC-059]. `ERR-014` is now named `HTTP_PERMANENT` and
  defined status-neutrally to cover both 4xx client errors and the [R-144]
  double-304 case, with the status (including 304) recorded in
  `fetch_events.http_status`; the [DOC-09 §4] 304 row now states the
  PERMANENT/`ERR-014` landing explicitly, and new [AC-101] pins the outcome
  and the broadened taxonomy definition.

## Unreleased — 2026-11-19 (bug hunt: AC-008 said the dispatch-time domain-cap gate does not trigger on the first five dispatch attempts, and AC-100 test)

### Correctness fixes

- [AC-008] asserted, for a domain with 5 ST-100 records and [CFG-006]=2, that
  "the dispatch-time gate [FR-011(e)] check does NOT trigger on the first 5
  dispatch attempts (since ST-100 is excluded from the per-domain check and the
  domain has no records in {ST-130,ST-140,ST-110,ST-120})". That rationale is
  only true *before* the first dispatch: [T-1] immediately moves the dispatched
  record to ST-110, so `inflight(D)` becomes 1, and [T-2] completion moves it to
  ST-140, so `successes(D)` becomes 1. After two successful dispatches
  `successes(D)+inflight(D)=2 ≥ [CFG-006]`, so the third dispatch attempt MUST
  be excluded [FR-005(2)], [FR-011(e)] — exactly as [AC-007] and [AC-078] (the
  latter an identical fixture) assert. The clause therefore made [AC-008]
  self-contradictory: its own next sentence says a subsequent dispatch after
  two successes is excluded, i.e. the third attempt, which is within the first
  five it had just declared exempt. It also contradicted [AC-006]'s bound of
  exactly two successes per Registrable Domain. Commit `bb896122` had already
  fixed the older "6th dispatch attempt" wording but left this clause intact,
  so a literal implementation of the criterion would dispatch attempts 3–5,
  overshoot the per-domain budget, and fail the very ACs pinning it. [AC-008]
  now states the live accounting and the first-two/ third-attempt split
  explicitly, and new [AC-100] pins the step-by-step sum, the ST-100→ST-190 /
  `CAP_REACHED` landing of the third attempt, and the failure mode of the
  former clause.

## Unreleased — 2026-11-19 (bug hunt: R-021 resolves relative page links against the final URL instead of a valid `<base href>`, and AC-099 test)

### Correctness fixes

- [R-021] opens by stating that a valid `<base href>` overrides the final URL
  as the resolution base, and [FR-042]/[AC-040] resolve the page's relative
  links against that extraction base. Its middle sentence, however, read "A
  relative `href` is first resolved against the final URL per RFC 3986 §5; the
  result is valid only if it is an acceptable absolute http(s) URL". The older
  changelog entry "[R-021] now defines `<base href>` validity" confirms the
  intended subject is a relative **`<base href>` value**, which must be
  resolved against the final URL to obtain and validate the absolute base.
  Read literally as a rule about a page link's relative `href`, it directs the
  implementation to resolve every relative link against the final URL even
  when a valid `<base href>` is present — the very behaviour the rule's own
  first sentence forbids. A fixture with a valid relative base therefore
  produces links rooted at the final URL instead of the base, failing [AC-040]
  and contradicting [FR-042], [R-020] and [R-021]'s first clause. [R-021] now
  separates the two steps explicitly: resolve a relative `<base href>` value
  against the final URL to form the effective base, then resolve every relative
  link against that effective base (against the final URL only when no valid
  base is present), with an invalid base ignored. New [AC-099] pins both steps
  and the invalid-base fallback, and records that a literal implementation of
  the former clause fails the valid-base case while still passing the
  invalid-base case.

## Unreleased — 2026-11-19 (bug hunt: robots exclusions mislabeled as ingestion/creation events in metrics and AC-098 test)

### Correctness fixes

- Commit `138d934` (the `state_transitions_total`/`AC-052` clarification) and
  the earlier `exclusions_total` mapping fix added `robots DISALLOW` to the
  `creation→ST-190` filter-outcome list and to the `urls_discovered_total{excluded}`
  ingestion-event list in [DOC-15 §1], mirrored in [AC-052] and [AC-090].
  Both placements are false under [DOC-07 §2]: a robots verdict is never
  evaluated during ingestion — the gate runs during dispatch selection, where a
  DISALLOW moves an ST-100 record to ST-190/`ROBOTS_DISALLOW` [FR-031], and a
  verdict change after [T-1] takes the ST-110→ST-190 send-time path [R-054];
  a continuously deferred Host sweeps gated ST-100/ST-150 records to
  ST-190/`ROBOTS_UNKNOWN_TIMEOUT` [R-103]. None of these is a record creation
  or an ingestion event, so labelling the first `from=creation` emits a
  transition label the state machine does not define and counting it in
  `urls_discovered_total{excluded}` breaks that metric's exact four-bucket
  partition of ingestion events [AC-052], [AC-091]. The same overstatement made
  [AC-092] claim a one-to-one mapping between `exclusions_total` reason codes
  and `urls_discovered_total{excluded}` assignments: `exclusions_total` counts
  all ST-190 entries, creations and later transitions alike, so the
  transition-only reasons (`ROBOTS_DISALLOW` at the gate,
  `ROBOTS_UNKNOWN_TIMEOUT` via the sweep, dispatch-time `CAP_REACHED`) have no
  `excluded` counterpart. [DOC-15 §1] now confines `creation→ST-190` to the
  genuine ingestion filter outcomes, drops gate-time robots DISALLOW from the
  `excluded` bucket, and states that `exclusions_total` and
  `urls_discovered_total{excluded}` are related but not one-to-one; [AC-052],
  [AC-090] and [AC-092] are corrected to match. New [AC-098] pins the real
  labels (`ST-100→ST-190`, `ST-110→ST-190`, `ST-100/ST-150→ST-190`), that these
  events move no `urls_discovered_total` bucket, and that a literal
  implementation of the former enumeration fails both the transition-label
  check and the partition check.

### Completeness fixes

- Commit `5f707c1` announced "New [AC-091]" but appended its text to [AC-090]
  rather than defining a standalone `- AC-091:` criterion, leaving [AC-091]
  cited by [AC-052], [AC-090], [AC-092], [AC-094] and this new [AC-098] with no
  definition to resolve to. The exact-partition assertion is restored as a real
  [AC-091] criterion (with the gate-time robots exclusion carved out per
  [AC-098]), and [AC-090] now points to it instead of duplicating its body.

## Unreleased — 2026-11-19 (bug hunt: RFC 5952 leading-zero suppression missing from IPv6 normalization step 3 and AC-097 test)

### Correctness fixes

- The v1.33.0 fix that made [DOC-06 §2] step 3 canonicalize IPv6 literals
  enumerated only three operations — lowercase every hex digit, compress the
  longest zero run, and retain the brackets — and so still omitted RFC 5952
  §4.1's suppression of leading zeros within each 16-bit field. That made the
  algorithm's own canonical-form claim, [R-003]'s examples, and the just-added
  [AC-096] internally unsatisfiable: lowercasing plus zero-run compression
  turns `[2001:0DB8::1]` into `[2001:0db8::1]` and
  `[2001:0DB8:0000:0000:0000:0000:0000:0001]` into `[2001:0db8::1]`, neither
  equal to `[2001:db8::1]`. A literal implementation of §2 step 3 therefore
  still produced two URL Identities, two URL Records, two
  `host_key`/Registrable-Domain values, and a split [CFG-006] per-domain cap
  for what [R-003] and [AC-096] assert is one resource. Step 3 now explicitly
  suppresses leading zeros in each 16-bit field (`0DB8`→`db8`, `0011`→`11`,
  an all-zero field becoming `0` before compression), [R-003] names the
  suppression sub-step, and the note records that RFC 3986 parsing does
  neither compression nor suppression. New [AC-097] isolates the missing
  sub-step with a fully expanded literal that has no compressible zero run,
  so it fails only on leading zeros even when compression is implemented —
  distinct from [AC-096]'s combined compression case.

## Unreleased — 2026-11-19 (bug hunt: IPv6 RFC 5952 canonicalization missing from normative normalization §2 and AC-096 test)

### Correctness fixes

- The [DOC-06 §2] "Normalization algorithm (normative)" — the exhaustive
  procedure [AC-001]'s normalization vector and every URL-Identity computation
  follow — lowercased the hostname (step 3) but never canonicalized IPv6
  literals, even though [R-003] requires IPv6 literals to be canonicalized to
  their RFC 5952 compressed form so that equivalent spellings share one URL
  Identity. RFC 3986 parsing alone does not compress IPv6 addresses, so a
  literal implementation of §2 produced two distinct identities for
  `[2001:0db8:0000:0000:0000:0000:0000:0001]` and `[2001:db8::1]`: two URL
  Records instead of one, two `host_key`/Registrable-Domain values, and a
  split [CFG-006] per-domain cap that could admit two separate batches of
  successes. [AC-057]'s existing example (`[2001:0DB8::1]` vs
  `[2001:db8::1]`) varied only hex-letter case, which step 3's lowercasing
  already handles, so it could not catch the gap. Step 3 now performs RFC 5952
  compressed-lowercase canonicalization (leftmost-longest zero-run compression,
  no `::` for a single field, brackets retained) before the identity and Host
  key are formed, with a note that RFC 3986 parsing alone is insufficient;
  [R-003] now cross-references §2 step 3 and lists the expanded form. New
  [AC-096] pins an expanded mixed-case literal and its compressed form to one
  identity/record/Host/domain key, and records why [AC-057]'s case-only
  example is insufficient.

## Unreleased — 2026-11-19 (bug hunt: R-232 once-retried gate missing from normative §3 retry procedure and AC-095 test)

### Correctness fixes

- The [DOC-13 §3] "Retry decision procedure (normative)" — the procedure that
  [FR-052] and every ordinary retry landing point to — had no check of
  `urls.once_retried_classes`. Its `on RETRYABLE outcome` block went straight
  to the [CFG-020] budget test and, when budget remained, landed the record in
  ST-150 with backoff. That is wrong for the "yes (once)" classes [ERR-003],
  [ERR-013]: [R-232] makes a repeat occurrence PERMANENT (ST-180,
  `last_error_class` := that class) regardless of remaining budget, and
  [AC-024] pins "two ERR-003 outcomes … reaches DEAD at the second
  occurrence". A literal implementation of the normative pseudocode would
  therefore retry a second ERR-003/ERR-013 in ST-150 instead of dead-lettering
  it, violating [R-232] and failing [AC-024]. The gap also left the ordinary
  fetch path inconsistent with the redirect final-target upsert path, where
  [AC-093] had just pinned the identical once-retried-classes-before-budget
  precedence in [R-062]. §3 now opens its RETRYABLE branch with the [R-232]
  gate — already-listed class ⇒ PERMANENT/ST-180 with the class recorded;
  otherwise append the class and fall through to the normal budget/backoff
  path — with a note that the gate precedes the budget check on both paths;
  [R-232] now states the same precedence and its application to both landings.
  New [AC-095] pins the ordinary (non-upsert) behavior: first ERR-003 appends
  the class and lands ST-150, second ERR-003 with budget remaining is
  PERMANENT/ST-180, a plain class retries normally, and a successes-driven
  `attempts` reset clears the list so one retry is available again.

## Unreleased — 2026-11-19 (bug hunt: ingested/duplicate is_seed contradiction and AC-094 test)

### Correctness fixes

- The [DOC-15 §1] `ingested` bucket definition contained a residual clause that
  listed "an existing record's `is_seed` flag updated idempotently" as an
  `ingested` event. That operation is exactly seed re-injection [FR-006] /
  config-seed re-ingestion [NFR-012], which the `duplicate` bucket of the same
  table, [AC-032], [AC-058] and [AC-090] all classify as `duplicate` — [AC-090]
  says "not `ingested`". The clause also contradicted `ingested`'s own
  "excluding events that result in a `duplicate` … bucket assignment" qualifier,
  so a literal implementation would double-count or mis-bucket seed re-ingestion
  and break [AC-052]/[AC-091]'s exact four-bucket partition. The clause is
  removed and `ingested` now explicitly states that an idempotent `is_seed`
  update on a pre-existing record is `duplicate`, not `ingested`. New [AC-094]
  pins this: re-ingestion onto an existing identity increments `duplicate`
  exactly once, while creation of a new record (including an [R-062] upsert that
  creates its target) remains `ingested`.

## Unreleased — 2026-11-18 (bug hunt: urls_discovered_total bucket pins lost in simplification)

### Correctness fixes

- The [DOC-15 §1] `urls_discovered_total` definition lost four load-bearing
  pins when commit `5ac463b` (2026-11-15, "Review fixes: ... Broke down
  `urls_discovered_total` into four concise buckets") replaced the detailed
  bucket semantics with a short bullet list. The deleted text was not
  redundant: each clause had been added by an earlier bug hunt precisely
  because its absence let counters diverge across conformant implementations
  under R-240's closed-enum discipline — `ingested` also counts URL Records
  created by redirect final-target upserts [R-062] (v1.16.0), runtime-seed
  rejects violating [FR-002] count as `dropped` even though they have no URL
  Identity (v1.18.0), seed re-injection [FR-006] and [R-062] upserts onto
  pre-existing records count as `duplicate` so the four buckets partition
  every ingestion event exactly (v1.19.0), and `excluded` means an audit
  ST-190 record with a reason code, not a silent drop (`deeb358`,
  2026-11-13). After the simplification the `dropped` bullet read only
  "no-record discards (unacceptable URLs per [R-001]/[R-002] …)" — those
  rules are *discovery-time* discards, so a literal implementation would not
  count an FR-006 runtime API reject — and the `duplicate` bullet read only
  "rediscoveries and upserts onto pre-existing records", omitting seed
  re-injection and the explicit exact-partition guarantee; the "partitions
  every ingestion event exactly" clause dropped from the column header and
  the "not silently dropped" qualifier vanished from `excluded`. This reopens
  the very divergences those changelog entries say were fixed. The full
  bucket semantics and citations are restored. New [AC-090] pins the two most
  fragile cases: a runtime-seed [FR-002] reject increments `dropped` despite
  recording nothing, and a seed re-injection onto an existing identity
  increments `duplicate`, so a literal implementation of the former bullets
  fails and AC-052's exact-partition assertion is again testable.

- New [AC-091] verifies the four-bucket partition is exact and that
  `excluded` and `dropped` are disjoint: `excluded` captures URLs producing
  an ST-190 audit record with a reason code, while `dropped` captures URLs
  producing no-record discards (unacceptable URLs per [R-001]/[R-002],
  runtime-seed rejects per [FR-006], OUT_OF_SCOPE with [CFG-038]=false,
  per-page candidate-cap overflow per [R-159]); the four buckets partition
  every ingestion event exactly, each event increments exactly one bucket,
  and no event increments more than one bucket or zero buckets.

 ## Unreleased — 2026-11-18 (bug hunt: exclusions_total metric mapping)

### Correctness fixes

- The [DOC-15 §1] `exclusions_total` metric was ambiguous in its relationship
  to the `urls_discovered_total{excluded}` bucket and the `OUT_OF_SCOPE` handling
  under [CFG-038]. A literal implementation could miscount reason codes or
  double-count entries across the two metrics, violating [R-240]'s closed-enum
  discipline. New [AC-092] pins the exact mapping: reason codes in
  `exclusions_total` correspond directly to `urls_discovered_total{excluded}`
  bucket entries; `OUT_OF_SCOPE` drops under [CFG-038]=false create no ST-190
  record and are excluded from `exclusions_total`, being counted only by
  `urls_discovered_total{dropped}` [FR-004]. This prevents the class of
  divergences seen in v1.18.0–v1.19.0 where counter semantics were unspecified.

## Unreleased — 2026-11-18 (bug hunt: once_retried_classes evaluation precedence and AC-093 test)

### Correctness fixes

- New [AC-093] pins the exact interaction between the once-retried-classes check and the retry budget during redirect final-target upserts: if the class is already listed in `once_retried_classes`, the outcome is PERMANENT (ST-180) regardless of [CFG-020]; if the class is not yet listed, it is appended per [R-232] and the normal budget check applies (ST-150 when `attempts` < [CFG-020], ST-180 otherwise). This evaluation order — once-retried-classes check first, budget check second — is now explicitly stated in [R-062] and prevents divergent implementations where RETRYABLE outcomes are landed in ST-150 even when the class is already listed, which would contradict [R-232] and violate [R-062]'s accounting; new [AC-093] tests this behavior and ensures [AC-052]'s four-bucket partition exactness is preserved.

  ## Unreleased — 2026-11-18 (bug hunt: last_error_class schema comment ST-180-only)

## Unreleased — 2026-11-18 (bug hunt: last_error_class schema comment ST-180-only)

### Correctness fixes

- The [DOC-11 §1] `urls.last_error_class` schema comment described the column
  as "error class to record on ST-180 per [DOC-13 §3]", implying a non-NULL
  class only ever accompanies a DEAD record. That is false under the spec's
  own completion rules: [T-2] and [R-062] require a failure upsert to SET the
  target's class to the outcome's class even when it lands in ST-150 (a
  RETRYABLE chain outcome), where the class stays transient until the
  target's own next successful completion clears it [AC-061(d)], [AC-084];
  ordinary RETRYABLE completions leave it unchanged, and an operator DEAD
  reset clears it [DOC-13 §4]. Following the comment literally would omit or
  clear the class on a RETRYABLE→ST-150 upsert landing — the exact defect the
  v1.16.0 changelog records as fixed ("Stale `last_error_class` survived
  recovery"), and the behavior [AC-084] fails. Git history pins the
  regression: commit `5ac463b` (2026-11-15, "Review fixes: ... DOC-11:
  Simplified last_error_class column comment") replaced the accurate comment
  — which named the RETRYABLE/ST-150 upsert case, the success clear, and the
  operator-reset/crash NULL cases — with the ST-180-only summary, and also
  dropped the line's leading indentation. The same commit is the source of
  the NFR-015 and ERR-018 regressions fixed immediately above. The comment
  now restores the outcome-based rule and its citations. New [AC-089] pins
  the observable behavior: a RETRYABLE ERR-002 upsert onto an ST-140 target
  leaves a non-NULL class in ST-150 until the target's own success clears it,
  so a literal implementation of the former comment fails [AC-084] and
  [R-062].

## Unreleased — 2026-11-18 (bug hunt: NFR-015 starvation bound ignores global-unit pool)

### Correctness fixes

- [DOC-05 NFR-015] promised "every due URL whose gates pass is dispatched
  within one scheduling-loop iteration of becoming due [R-210]" — an absolute
  one-iteration guarantee. [DOC-12 R-210] is explicitly narrower: dispatch is
  guaranteed within one loop iteration "provided the global-unit pool is not
  full," and [R-212] states that a dispatch finding the pool full at [T-1]
  commit time commits nothing, leaving the record in ST-100 until a unit-release
  wake. A due, all-gates-passing URL at a moment when [CFG-010] is saturated is
  therefore *not* dispatched in that iteration, so NFR-015's literal claim is
  false — and following it would over-commit the global pool in breach of
  [FR-011(b)], [AC-011] and [INV-3]. Git history pins the regression: commit
  `5ac463b` (2026-11-15, "Review fixes: ... starvation freedom ...") amended
  R-210 from "it MUST be dispatched within one loop iteration; gates are the
  ONLY reason to wait" to the qualified form with the global-pool proviso and
  the [R-212] parenthetical, but left NFR-015 (introduced in `c5b8626`, v1.8.0)
  untouched. Under [R-000] both are level-(3) statements, but [DOC-12] is a
  topic document and [DOC-05] a general one, so R-210/R-212 govern and NFR-015
  was the erroneous text. NFR-015 now carries the same "provided the
  global-unit pool is not full" proviso, cites [R-212], and states that a
  [FR-011(b)] deferral keeps the record in ST-100 until the next wake rather
  than exempting it from [CFG-010]. The same commit also left R-210's
  "gates are the ONLY reason to wait" absolute unqualified beside its own
  global-pool exception and a stray closing bracket (`)]`); R-210 is now
  scoped with "subject to that proviso" and the stray bracket removed. New
  [AC-088] pins the observable behavior: with [CFG-010]=1 and two due,
  gate-passing URLs on different Hosts, one dispatch commits in the first
  iteration and the second stays ST-100 until the completion wake releases the
  unit, so a literal implementation of the former unconditional NFR-015 that
  dispatches both fails [FR-011(b)]/[AC-011].

## Unreleased — 2026-11-18 (bug hunt: path-shape trap cap counted records)

### Correctness fixes

- [DOC-06 §5] filter 3 and [DOC-14 CFG-030] both summarized the path-shape
  guard as "at most [CFG-030] URLs may share one shape per Host." That bound
  is false under the spec's own counting rule: once [CFG-030] records share a
  shape, every further candidate with that shape is *excluded* to
  ST-190/`TRAP_PATH_BUDGET`, and [R-041] requires excluded URLs to keep their
  ST-190 records while [R-042] requires every record on the Host — "including
  excluded ones" — to contribute its shape. The number of URL Records sharing
  the shape therefore grows past [CFG-030] without bound (501, 502, … at the
  default) even though no more than [CFG-030] are ever admitted. Git history
  pins the regression: commit `b4b0a1b` (v1.6.0) fixed the off-by-one
  comparison (`>` → `≥` on existing records) and introduced both "at most
  [CFG-030] URLs" summaries in the same change, without reconciling them with
  [R-042]'s "including excluded ones" rule, which had existed since `fe5351c`
  (v1.4.0); the v1.6.0 changelog repeats the same erroneous conclusion
  ("so at most [CFG-030] URLs share a shape"). Under [R-000] the summaries
  are level-(4) prose while [R-041]/[R-042] are level-(3) rules, and their
  "any state / including excluded ones" wording is also the more specific
  statement, so the summaries were the erroneous text. Both are now qualified
  to "at most [CFG-030] non-excluded URLs may be admitted per shape," with an
  explicit note that the ST-190 records themselves carry and count toward the
  shape so only admissions are bounded — deliberately unlike the global cap
  [CFG-005], whose ST-190 audit records are not counted [FR-005(1)(A)]. New
  [AC-087] asserts that with [CFG-030]=3 the 4th–6th candidates become
  ST-190/`TRAP_PATH_BUDGET` records that still share and count toward the
  shape (six records on one shape), so a literal implementation of the former
  "at most [CFG-030] URLs may share one shape" clause that suppresses those
  records or drops them from the counter fails.

## Unreleased — 2026-11-18 (bug hunt: NFR-004 one-shot fetch bound)

### Correctness fixes

- The [DOC-05 §NFR-004] `CFG-025=0` case asserted "each URL is fetched at
  most once — a single generation per URL." Read literally, this forbids a
  second fetch of a URL identity under *any* configuration, but the spec
  mandates exactly such a second fetch when `[CFG-021]=true`: [FR-051] moves a
  rediscovered terminal-success (ST-140) record back to ST-100, [R-052]
  explicitly states that this rediscovery-refresh edge "is still an edge for
  one-shot records (`[CFG-025]=0`, `[R-221]` no-store) — only the recrawl-due
  path is suppressed," [DOC-12 §1] repeats that the refresh "still promotes
  such a record, independently of recrawl eligibility," [DOC-11 §1] describes
  the resulting same-run second fetch ("a same-run refetch ever occurs (e.g.,
  a rediscovery refresh under `[CFG-021]=true` [FR-051]) ... UPSERTS (replaces)
  the prior row"), and [AC-072] makes the same ST-140 re-admission the terminal
  fate of a one-shot record. Under [R-000] a numbered NFR is level (3), but
  within that level a topic document overrides a general one: [DOC-07]/[DOC-11]/
  [DOC-12] therefore govern over the general [DOC-05], so NFR-004's absolute
  claim is the erroneous statement. Git history confirms the regression:
  commit `2ea28a5` ("Spec review v1.21.0 ... restructure NFR-004's disjoint
  generation-bound cases") introduced the unqualified "at most once" wording
  while splitting the cases, whereas [DOC-11 §1] had carried the correct
  qualified form — "fetched at most once **per recrawl cycle**" — since commit
  `3ac6169` (v1.1.0). The clause is now qualified to "at most once per fetch
  cycle — a single generation per cycle, not per URL identity," naming the
  rediscovery refresh, the operator DEAD reset, and the same-attempt [R-144]
  refetch as permitted re-fetches and reaffirming the one-blob-per-body
  [DEC-005] bound. New [AC-086] asserts that a `[CFG-025]=0`, `[CFG-021]=true`
  rediscovery performs the second fetch (upserting the same-run `pages` row)
  while a never-rediscovered one-shot record stays terminal, so a literal
  implementation of the former "at most once" clause fails.

## Unreleased — 2026-11-18 (bug hunt: ERR-018 backoff floor on ST-180)

### Correctness fixes

- The [DOC-13 §3] retry-decision pseudocode applied the ERR-018 hop-abort
  floor (`next_attempt_mono = max(next_attempt_mono,
  target_host.next_allowed_fetch_at)`) as a top-level `if outcome is
  ERR-018:` statement, evaluated after the `if attempts ≥ CFG-020` budget
  test. An ERR-018 abort is RETRYABLE, so when the budget is already
  exhausted the record is sent to ST-180 (DEAD) at line 45 and the
  unconditional floor then wrote a non-NULL `next_attempt_mono` onto that
  DEAD record. This contradicts the field-hygiene invariant — "a backoff
  timer must not survive outside ST-150" and `next_attempt_mono` "is
  non-NULL only in ST-150" — asserted by [DOC-13 §4], [R-062], [DOC-07 §2],
  [DOC-12 §1] and the [DOC-11 §1] schema comment ("for ST-150 backoff"), and
  with the ST-150-only scope of the floor described in [R-131] and [AC-055]
  (which explicitly contemplates ERR-018 reaching ST-180 when the window
  never opens within the retry budget). Git history confirms the regression:
  commit `5ac463b` ("DOC-13: Clarified ERR-018 hop-abort floor description")
  dedented the block out of the `else:` branch that previously confined it
  to the ST-150 landing. The floor is now re-indented into that `else:`
  branch with an explicit "ST-150 landing only — never on a budget-exhausted
  ST-180 landing" note, so it applies only where a backoff timer exists. New
  [AC-085] asserts a budget-exhausted ERR-018 abort reaches ST-180 with a
  NULL `next_attempt_mono` while a within-budget abort still floors it on
  ST-150, so a literal implementation of the former top-level clause fails.

## Unreleased — 2026-11-18 (bug hunt: R-062 upsert `last_error_class`)

### Correctness fixes

- The field-effects summary appended to the [DOC-07 §2] upsert transition
  diagram claimed that on "state changes outside ST-180" the upsert
  "clears `last_error_class`". A RETRYABLE redirect-chain upsert lands in
  ST-150 — outside ST-180 — yet [R-062] §4 explicitly says failure upserts
  SET `last_error_class` to the outcome's class (success upserts clear it),
  corroborated by [T-2] and [AC-061(d)], both of which rely on a transient
  class surviving a RETRYABLE upsert until the target's own next success
  clears it. The summary therefore prescribed the opposite of the normative
  rule for the common RETRYABLE→ST-150 case. The summary now states the
  outcome-based rule, aligns the `next_attempt_mono` clause with §4's
  "landing outside ST-150" wording, and repairs the "upsi rt's" typo from
  the bad edit that introduced the clause. New [AC-084] asserts a
  RETRYABLE upsert leaves a non-NULL class until the target's own success
  clears it, so a literal implementation of the former summary fails.

## Unreleased — 2026-11-18 (bug hunt: R-052 ST-140→ST-100 reset)

### Correctness fixes

- [R-052] was garbled by an earlier review edit that inserted the note "the
  redirect final-target completion upsert [R-062] is not a rediscovery":
  the insertion consumed the original rule subject "ST-140→ST-100
  transitions (recrawl due [FR-050] and rediscovery refresh [FR-051])",
  leaving the fragment "[R-062] is not a rediscovery: [FR-050] and
  rediscovery refresh [FR-051]) reset `attempts=0` ...". Read literally,
  this credits the [R-062] upsert with the `attempts` reset and `due_at_mono`
  install that belong only to actual ST-140→ST-100 transitions, directly
  contradicting [R-062] ("the upsert never modifies the target's
  `attempts`"), [R-231], and [R-232] (the once-retried list clears exactly
  when `attempts` resets to 0). It also carried a stray `)` and a
  self-citation to [R-052] from the [FR-051] wording. The rule now restores
  the "ST-140→ST-100 transitions" subject, states explicitly that the
  [R-062] upsert never resets `attempts`, and drops the self-reference.
  New [AC-083] exercises an upsert onto a pre-existing ST-140 record and
  asserts `attempts`, priority, and `once_retried_classes` are preserved,
  contrasting with the genuine recrawl/rediscovery promotions.

## Unreleased — 2026-11-18 (bug hunt: gate-time robots unit release)

### Correctness fixes

- [R-051] listed "the robots paths [FR-031], [R-054]" among the transitions
  out of {ST-110, ST-120} that release held Host/global units, but [FR-031]
  is the selection-time robots DISALLOW exclusion, which fires on an ST-100
  candidate before [T-1] and therefore holds no units. The same rule's
  no-unit clause already named [FR-031] as releasing nothing, so R-051 was
  self-contradictory; a conformant implementation following the release list
  would decrement counters it never incremented, corrupting [INV-3] and
  [FR-011(b)]. The unit-release list now names only the send-time re-check
  path [R-054], and the DOC-07 note on the robots paths was reconciled.
- [AC-009] described the selection-time robots-DISALLOW exclusion as firing
  "with units released per [R-051]" — the exact inverse of R-051's no-unit
  clause — and cited "AC-011's cap gate" although AC-011 is the concurrency
  invariant and the cap gate is [FR-011(e)] (AC-007/AC-008). AC-009 now
  states that the gate-time exclusion holds and releases no units, keeps the
  termination argument against the correct cap gate, and distinguishes the
  send-time [R-054] path. New [AC-082] asserts no counter change on the
  gate-time path and exactly one release on the send-time path.

## Unreleased — 2026-11-18 (bug hunt: global cap scope)

### Correctness fixes

- [FR-005(1)(A)] had scoped the enqueue-time global cap [CFG-005] to "domain D
  (the candidate's Registrable Domain)", making a config named `max_urls_total`
  and tabled as a "total records" resource cap [DOC-16 §3] behave as a second
  per-domain cap. A conformant implementation following that wording could hold
  [CFG-005] non-EXCLUDED records on *each* domain, leaving the crawl-wide total
  unbounded, and it contradicted NFR-004 ([CFG-005] distinct payloads per
  generation), [AC-006] (counts all non-EXCLUDED records), and the changelog's
  note that [CFG-005], unlike [CFG-006], has no dispatch-time gate. The clause
  now states that [CFG-005] counts non-EXCLUDED records across the whole crawl,
  with only the [R-062] upsert exemption retained. New [AC-081] exercises a
  multi-domain fixture to detect a per-domain mis-implementation, and the
  "other domains" clauses in [AC-034] and [AC-080] (which described the
  per-domain cap, not the global one) were reconciled with system-wide
  enforcement.
- Removed stray `+` diff markers accidentally committed onto the start of
  [AC-077], [AC-079], and [AC-080], which broke their markdown list items.

## 1.33.0 — 2026-10-29 (review pass v33: correctness, completeness, consistency)

### Correctness fixes

- [R-062] cannot produce an ST-190 landing: its fetch-outcome states are
  exactly ST-130, ST-150, and ST-180. The ST-190→ST-190 same-state landing
  re-added in v1.32 was therefore unreachable (a chain whose final target is
  robots-DISALLOWed aborts at the hop with ERR-017 and never reaches the
  upsert). The same-state list is back to ST-130→ST-130, ST-150→ST-150, and
  ST-180→ST-180; an existing ST-190 record is always overwritten
  (ST-190→ST-130/ST-150/ST-180) and its `exclude_reason` cleared.
  [DOC-07 §2], [DOC-07 §4], and AC-061(e) are aligned.
- The DOC-07 §2 same-state note now says only [T-2]'s URL-record completion
  fields apply to the target; the transaction's host-side effects
  (`hosts.inflight−1`, host counters) belong to the source attempt, so an
  upsert target is never double-decremented.
- [FR-050] said the 304-unchanged multiplier "shortens" the recrawl
  interval; [DOC-12 §4] doubles it (up to 4×). The clause now says the
  multiplier lengthens the interval and delays the next recrawl.
- [R-221] and [AC-072] offered an "operator reset" as a way to re-admit a
  no-store ST-140 record, but the operator reset is DEAD-only (ST-180→ST-100
  [DOC-13 §4]); it cannot leave ST-140. Both now name only the rediscovery
  refresh ([CFG-021]=true [FR-051]) and state the reset's scope explicitly.
- [R-141] now compares `Content-Length` against the body bytes received on
  the wire *before* Content-Encoding decode; the decoded Payload size is
  what `payload_size`/`payload_sha256` describe [FR-024]. Previously
  "actual bytes" was ambiguous and could flag every compressed response.
- [R-021] now defines `<base href>` validity: a relative `href` is resolved
  against the final URL, the result must be an acceptable absolute http(s)
  URL ([R-001], [R-002]), and an invalid base is ignored.
- [DEC-004] said every redirect target creates a URL Record; [R-062]
  creates one only for the final target, while intermediate hops live in
  `redirect_chain` [R-133]. The decision now matches the rule.
- NFR-004 jitter assumption fix: the specification previously claimed
  "jitter only shrinks intervals" while the 304 multiplier (≤4×) only grows
  them, so the bound holds for every URL. Jitter (CFG-026, up to 0.9) can both
  shrink and grow intervals by a factor of (1−[CFG-026]); the divisor
  (1−[CFG-026]) in the retention formula provides a conservative worst-case
  bound ensuring the at-most claim holds for every URL. Corresponding test
  AC-074 added to [DOC-17] verifying this correction.

### Completeness additions

- New [T-5] send-time robots abort transaction: the state change
  (ST-190/`ROBOTS_DISALLOW` or the ST-100 compensation with
  `due_at_mono` := now), the [T-1] `attempts` rollback, and the source
  Host's `inflight` decrement commit atomically, with no `fetch_event` and
  no `pages` row. [R-054], [AC-016], the DOC-07 send-time transitions, and
  the glossary's global-unit interval now cite it.
- [INV-5] now carries the [R-062] note v1.32's changelog described but did
  not actually add: an upsert onto a pre-existing record is not a
  rediscovery; it updates `last_seen_at`, and its state/scheduling effects
  are exactly R-062's; a record in {ST-110, ST-120} keeps its state,
  `attempts`, and scheduling (on chain success the target's `pages` row
  still commits).
- [R-062] now states the creation values of the non-nullable target fields
  it left implicit: `source_run_id` = the Run performing the chain,
  `raw_first_seen` = the final `Location` value after RFC 3986 §5
  resolution, and `is_seed` = false.
- [T-2] now states that [CFG-025]=0 also leaves `due_at_mono` NULL,
  matching the [DOC-11 §1] schema comment for one-shot records.
- [AC-029] no longer calls a non-HTML type "allowed" when [CFG-028]=false
  (the effective allowed list is empty [R-143]).

- [FR-005] clarified the enqueue-time global cap [CFG-005] excludes redirect final-target upserts [R-062] bounded by in-flight chains, and the per-domain cap [CFG-006] excludes ST-100 records (only counts successes+inflight in {ST-130,ST-140,ST-110,ST-120}); dispatch-time gate [FR-011(e)] is authoritative, with ST-100‑queued URLs bypassing the enqueue-time per-domain check but remaining subject to dispatch-time enforcement per AC-007/AC-008. ST-190 audit records remain excluded from the [CFG-005] budget. Redirect final-target upserts count toward the target domain's per-domain cap upon successful chain completion.
- [FR-005] dispatch-time gate text clarified: removed redundant "Note" stating candidate resides in ST-100 at dispatch time, replaced with clean statement that the gate is authoritative across all states, per AC-007/AC-008/AC-077. ST-100‑queued and ST-150‑gated URLs bypass the enqueue-time per-domain check but remain subject to dispatch-time enforcement. ST-190 audit records remain excluded from the [CFG-005] budget.
- [FR-005] dispatch-time gate explicitly invokes the FR-005(1)(B) per-domain budget condition, ensuring the gate's authoritative scope across ST-100/ST-150 states is grounded in the same evaluation as the enqueue-time check, maintaining consistent behavior between both exclusion points.
### Versioning

- KB version 1.32.0 → 1.33.0; touched documents (DOC-00, DOC-02, DOC-03,
  DOC-04, DOC-06, DOC-07, DOC-09, DOC-11, DOC-12, DOC-17, DOC-18, README)
  bumped accordingly.

## 1.32.0 — 2026-10-29 (review pass v32: consistency, completeness)

### Completeness additions

- The [R-062] upsert edge in [DOC-07 §2] named only three same-state
  landings (ST-130→ST-130, ST-150→ST-150, ST-180→ST-180), but its source
  set includes ST-190 and "fetch-outcome state replaces the record's
  current state" admits ST-190→ST-190 — e.g. a chain whose failure class
  matches the target's ST-190 reason. The same-state list now names all
  four, with [T-2]'s field effects applying regardless, and AC-061(e)
  aligned.
- [DOC-07 §4] (R-062) now states that a same-state ST-190 landing
  preserves `exclude_reason` (the reason is final [R-031], [R-041]) and
  that ST-190 carries no backoff timer to clear, so the landing affects
  only `last_seen_at` plus the failure bookkeeping [T-2] sets.
- [INV-5] gained a cross-reference note that [R-062] upserts onto
  pre-existing records are not rediscoveries: they update `last_seen_at`
  and, on chain success, commit the target's `pages` row only, and never
  change state, `attempts`, or scheduling of a record in {ST-110, ST-120}
  ([DOC-03] Data flow invariants).

### Consistency fixes

- [FR-050]'s "unless freshness headers dictate otherwise per [DOC-12 §4]"
  was a pointer to a section that never invokes freshness headers (only
  the 304 multiplier and [R-221] no-store); the clause now names both
  header-driven behaviors explicitly so the FR is self-contained.
- [R-051]'s unit-release list ended at "including the robots paths"
  without noting that [R-062] same-state upsert landings carry no units
  and release nothing; the note is now in the rule.
- [DOC-08 §4] R-101: the "no other character is special" clause now says
  no *other position* of `$` is special, removing the self-contradiction
  with the terminal-`$` anchor it had just defined.
- [DOC-03] Deployment model's graceful-shutdown sentence now cites
  [AC-050], pinning the exit-code-0 guarantee to its acceptance
  criterion.

### Versioning

- KB version 1.31.0 → 1.32.0; touched documents (DOC-03, DOC-04, DOC-07,
  DOC-08, DOC-17, DOC-18, README) bumped accordingly.

## 1.31.0 — 2026-10-28 (review pass v31: completeness, consistency)

### Completeness additions

- [R-221] (one-shot `no-store` responses) and [R-222] (retention-deleted
  records are never re-scheduled for recrawl) had no acceptance criterion,
  and R-221's one-shot behavior was not pinned to the state machine — the
  ST-140→ST-100 edge in [DOC-07 §2] read as firing for every recrawl-due
  record, which is false for a one-shot record that never installs a
  `due_at_mono`. New AC-072 (no-store: stored/extracted normally, `due_at_mono`
  left NULL, ST-140→ST-100 due-time promotion suppressed, rediscovery refresh
  and operator reset still apply) and AC-073 (retention-deleted URL Record:
  no recrawl re-scheduling, rediscovery may re-create it).
- The ST-140→ST-100 edge in [DOC-07 §2] now names the one-shot carve-out
  ([R-221], [CFG-025]=0) explicitly, matching [DOC-12 §4]. The matching
  due-time promotion in [DOC-12 §1] states the same suppression and the
  rediscovery-refresh exception, keeping the loop model and the state
  machine in agreement.
- [R-052] clarified that a rediscovery refresh remains an edge even for
  one-shot records — only the recrawl-due path is suppressed [DOC-12 §4].
- [DOC-12 §4] R-221 now states the mechanical basis: T-2 leaves
  `due_at_mono` NULL for a no-store success; R-222 now states that a
  retention sweep deleting a URL Record removes its `due_at_mono` with it.
- [DOC-11 §1] `urls.due_at_mono` schema comment and the [T-2] transaction
  now note the [R-221] NULL-due-time case so the column's semantics are
  complete in both places that write it.

### Consistency fixes

- [DOC-11 §1] the `urls` schema block had inconsistent indentation introduced
  by a prior edit (two rows four-space indented, the closing paren
  misaligned); normalized to the block's three-space style.
- [DOC-00] the Crawl Session term cited [DOC-03] bare; the Run lifecycle it
  anchors is defined in the unnumbered "Deployment model" section, so the
  reference now names the section [DOC-03 Deployment model] (the
  section-qualified form already used by [DOC-11 §1]).

### Versioning

- KB version 1.30.0 → 1.31.0; touched documents (DOC-07, DOC-11, DOC-12,
  DOC-17, DOC-18, README) bumped accordingly. DOC-00 carries a
  citation-only fix and keeps its last behavioral version.

## 1.30.0 — 2026-10-28 (review pass v30: consistency, housekeeping)

### Consistency fixes

- [DOC-18 §1.26.0] the versioning line listed DOC-09 twice in the
  touched-documents parenthetical (a copy artifact from the two separate
  DOC-09 edits in that pass); the duplicate entry was removed.

### Housekeeping

- [README] `last-updated` was stale at 2026-10-27; aligned to the release
  date.

### Versioning

- KB version 1.29.0 → 1.30.0; touched documents (README, DOC-18) bumped
  accordingly.

## 1.29.0 — 2026-10-27 (review pass v29: consistency)

### Consistency fixes

- [DOC-09 §4] the ERR-018 (hop politeness wait > [CFG-035]) row did not state
  the worker-release guarantee that [AC-055] requires ("no fetch worker is held
  for an over-threshold wait"), unlike the header-size row fixed in v1.28.0.
  Added the guarantee to the row so the cap's disposition matches the
  acceptance criterion.

### Versioning

- KB version 1.28.0 → 1.29.0; touched documents (DOC-09, README, DOC-18)
  bumped accordingly.

## 1.28.0 — 2026-10-27 (review pass v28: consistency)

### Consistency fixes

- [DEC-007] said an unreachable/broken robots.txt should "temporarily treat the
  entire Host as disallowed", but [DOC-08 §2.3] specifies the Host becomes
  DEFERRED (fail closed — no page fetches) and retried with backoff, not a
  DISALLOW verdict. Reworded DEC-007 to "deferred (fail closed — no page
  fetches)" so the decision matches the enforced behavior.
- [AC-064] called the `sid` glob "unanchored", contradicting [R-043] (patterns
  are implicitly anchored at both ends). Reworded to "(implicitly anchored,
  whole-string)".
- [README] KB version was stale at 1.26.0 while the changelog stood at 1.27.0;
  aligned to the current KB version.
- [DOC-09 §4] header-size row now states that no fetch worker is held on
  rejection, matching [AC-063].

### Versioning

- KB version 1.27.0 → 1.28.0; touched documents (README, DOC-02, DOC-17,
  DOC-09, DOC-18) bumped accordingly.


## 1.27.0 — 2026-10-27 (review pass v27: completeness)

### Completeness additions

- DOC-09 §4's response-classification table had no row for the 64 KiB header-size cap: error class ERR-016 `HEADER_TOO_LARGE` was declared in [DOC-16 §3], defined in [DOC-13 §1], and cited by [AC-063] as enforced in [DOC-09 §4], but no enforcement row existed there. Added the row so the cap's disposition (outcome PERMANENT, error_class ERR-016, state ST-180, no payload persisted) is specified exactly where the acceptance criterion points.

### Versioning

- KB version 1.26.0 → 1.27.0; touched documents (DOC-09, DOC-18) bumped accordingly.

## 1.26.0 — 2026-10-27 (review pass v26: correctness, completeness, consistency)

### Correctness fixes

- The blocklist glob language was undefined: [CFG-029] and [CFG-037] said
  "glob patterns" but never defined metacharacters, anchoring, case
  sensitivity, or escaping — two conformant implementations could disagree
  on a security-relevant blocklist, and [CFG-037] is enforced on redirect
  hops [R-131]. New [R-043] pins the language: whole-string, case-sensitive
  match; `*`, `?`, and `\` escapes only; no character classes or implicit
  wildcards; empty pattern matches only the empty string. [CFG-029]/[CFG-037]
  cite it; AC-064/AC-067 exercise matching.
- Runtime seeds on a new domain/Host were not always in scope: the Scope
  predicate derives the seed set from existing `is_seed` records [FR-006],
  but a runtime seed's own record is created only after the predicate passes
  — under SEED_DOMAINS/SEED_HOSTS a well-formed new-domain seed would be
  recorded OUT_OF_SCOPE and never fetched, contradicting "behaves identically
  to a config seed". New [R-032] includes the seed itself in the set before
  evaluation (PREFIX_LIST remains fixed by [CFG-039]/[V-4]); [FR-006] and
  AC-058 aligned.
- [R-062] said "both records move to ST-130" on a successful redirect chain,
  contradicting its own rule that an independently in-flight target record
  ({ST-110, ST-120}) is untouched; and for a failing chain it told the
  in-flight target to record "its `pages` row", although failure completions
  insert no page row. The success sentence now carries the target-in-flight
  exception, and the exception is scoped to "on chain success" with state,
  `attempts`, and `last_error_class` untouched. The same-state-landing list
  also omitted ST-130→ST-130 (a pre-existing extraction-pending target).
- [R-402] marked "the referring URL" ST-180 on an SSRF-blocked redirect. For
  a block at hop *n* > 1 the referring URL is an intermediate hop with no URL
  Record, while [DOC-09 §4] marks the source record. R-402 now names the
  source URL Record for the state change and the Host of the redirecting hop
  for the `suspicious` flag; AC-069 aligned.
- [DOC-13 §3]'s ERR-018 floor was dimensionally wrong: it took
  `max(delay, target_Host.next_allowed_fetch_at)`, mixing a duration with an
  absolute monotonic instant (the same class as the v1.7.0 NFR-004 unit
  error). The floor is now applied to `next_attempt_mono` as an absolute
  instant; Retry-After remains a converted, never-clamped delay floor.
- The extraction completion path had no storage transaction: [T-2] inserts
  `pages` rows at fetch completion, before parsing, yet `pages.title`,
  `noindex`, `parse_ok`, and `parse_reason` are extraction outputs and
  nothing wrote them. New [T-4] commits the page-artifact upsert/materialized
  metadata and ST-130→ST-140 transition atomically and idempotently; [T-2]
  now states the extraction-owned columns may be left at defaults. AC-041
  verifies title/noindex persistence.
- The artifacts 2 MiB cap was unenforceable for large non-list fields:
  truncation covered only headings/main_text/outlinks, so a huge
  title/description/meta string could exceed the cap even after those were
  emptied. The fixed truncation order now continues through the remaining
  variable-length strings, and `word_count` is recomputed from the final
  stored `main_text` so it never describes discarded text. AC-071 added.
- `outlinks` was ambiguous about deduplication, ordering, and which
  candidates it contains, although AC-040 demands byte-identical artifacts.
  It is now the deduplicated ([R-154]), document-ordered set of
  anchor-derived candidates including nofollow and later-filtered ones, with
  anchor text per [R-156]; non-anchor sources are explicitly excluded.
- [DOC-15 §1]'s `from=creation` description omitted creation→ST-100, the
  most common record-creating transition, although [DOC-07 §2] lists three
  creation arrows. All three are now enumerated, and AC-052 checks the label.
- [DOC-09 R-145] left timings undefined for a chain aborted before its next
  hop was sent; the phase timers are now the last exchange actually
  performed, with `total` spanning the attempt to that point.

### Completeness additions

- Redirect-hop robots re-check: the hop gate was evaluated before the
  politeness/capacity wait, and step 4 sent without re-reading the verdict, so
  a robots change during a (possibly [CFG-035]-long) wait could still send.
  Step 4 now re-reads the verdict immediately before send — the hop analogue
  of [R-054] — aborting DISALLOW as ERR-017 and UNKNOWN as ERR-010. New
  AC-070.
- [V-7]: [CFG-044] (`robots_defer_backoff_start_s`) could exceed [CFG-040]
  (`robots_defer_max_s`), making the "initial backoff capped by the maximum"
  rule ill-defined. Startup now requires `CFG-044 ≤ CFG-040`; AC-060 covers
  it.
- AC-056(b) now covers the userinfo case that [R-130] already classifies.

### Consistency fixes

- AC-012's "Allow longer-match wins on tie" was garbled; it now states the
  actual [R-101] rule (longest value wins; Allow wins an exact-length tie).
- README's AC categorical scheme claimed 060+ was "security/redirects", but
  AC-060–AC-071 also cover configuration validation, resource caps, trap
  filters, and extraction caps; the scheme now says "cross-cutting
  conformance".

### Versioning

- KB version 1.25.0 → 1.26.0; touched documents (README, DOC-04, DOC-06,
  DOC-07, DOC-09, DOC-10, DOC-11, DOC-13, DOC-14, DOC-15, DOC-16, DOC-17,
  DOC-18) bumped accordingly.

## 1.25.0 — 2026-10-26 (housekeeping)

### Correctness fixes

- Citation fixes: [FR-023] classified oversized payloads as PERMANENT/ERR-007 but pointed at [DOC-09 §6] (the FetchResult field contract); the class is defined in [DOC-13 §1], so the cite now points there. [AC-020] cited [DOC-09 §6] for redirect-cap ERR-011; that rule is [R-130] (§3), so the cite now points there. [R-062] attached [R-157] (extraction determinism) to "upsert-idempotent", which this rule itself defines; [R-157] now credits the identical-artifacts claim and idempotency is attributed to the rule.

### Versioning

- KB version 1.24.0 → 1.25.0; touched documents (README, DOC-04, DOC-07, DOC-17, DOC-18) bumped accordingly.

## 1.24.0 — 2026-08-29 (housekeeping)

### Housekeeping

- Fixed the README `last-updated` date, which had gone stale (2026-08-25 → 2026-08-29) after the v1.23.0 release.

### Versioning

- KB version 1.23.0 → 1.24.0; touched documents (README, DOC-18) bumped accordingly.

## 1.23.0 — 2026-08-29 (review pass v23: completeness)

### Completeness additions

- Added AC-063 through AC-069 to close acceptance-criterion gaps for behaviors
  that were fully specified but untested: header-size rejection (ERR-016,
  [DOC-16 §3]); all four trap-filter reason codes (TRAP_PARAM, TRAP_PATH_BUDGET,
  DEPTH_LIMIT, BLOCKLIST, [DOC-06 §5]); redirect-out-of-scope termination
  (ERR-015, [R-030]); and SSRF-block redirect with host suspicious-flag
  (ERR-004 + R-402). All four trap filters and ERR-016/ERR-015 had normative
  rules and ST-190/Error classifications but no verifiable AC, so a conformant
  implementation could pass all existing ACs while violating them.

### Consistency additions

- Documented the categorical AC numbering scheme in the README conventions
  section (001–006 ingestion, 010–019 politeness/scheduling, 020–029
  fetching/errors, 030–033 state machine, 040–043 extraction/storage,
  050–059 operations, 060+ security/redirects) so the intentional gaps are
  explicit rather than ambiguous.

### Versioning

- KB version 1.22.0 → 1.23.0; touched documents (DOC-00, DOC-17, DOC-18) bumped
  accordingly.

## 1.22.0 — 2026-08-25 (review pass v22: consistency, completeness)

### Consistency fixes

- The defensive ST-150→ST-180 edge stranded a stale `next_attempt_mono`,
  contradicting the field-hygiene invariant it sits under: [DOC-12 §1]
  asserts `next_attempt_mono` is non-NULL only in ST-150, and every other
  exit from ST-150 states its clearing (the due-time promotion, the
  [R-103] sweep, the operator reset, upsert landings outside ST-150) —
  but the machine's own legal edge (unreachable in practice, yet listed,
  so bound by AC-052's legality check) did not. An implementation
  exercising the defensive branch would carry a backoff timer into ST-180.
  The edge now clears the column like every sibling exit.

### Completeness additions

- Robots-exchange redirect hops had no pinned unit accounting: §2.2 sends
  the exchange with one global + one per-Host unit and follows redirects
  "per [DOC-09 §3] hop rules", but nothing said the per-Host unit
  transfers at each hop. An implementation could hold the original Host's
  unit through a cross-host robots redirect and its subsequent politeness/
  capacity waits, so per-host inflight attribution — which AC-011 counts
  robots exchanges toward — diverged across conformant implementations.
  Pinned: [R-131]'s hop dispatch protocol governs (release-before-acquire;
  waits hold no Host unit).
- R-133's `redirect_chain` refusal enumeration omitted the [ERR-011]
  refusals that have a usable target identity: a non-followable final 3xx
  status with a parsable `Location`, and a `Location` resolving to an
  unacceptable target (non-http(s) scheme, userinfo). Whether the
  stopped-at target was recorded was unspecified — audit divergence of
  exactly the class the loop-refusal precedent ([R-132], v1.13.0)
  eliminated for loops. Pinned: recorded, as a refused hop target; a
  `Location` missing or too broken to yield an identity leaves nothing to
  record; the last-element sentence now also covers refusal-terminal
  chains (the old "received a final response" phrasing presupposed a
  followed target). AC-056(b) extended to verify the recording.

### Versioning

- KB version 1.21.0 → 1.22.0; touched documents (DOC-07, DOC-08, DOC-09,
  DOC-17) bumped accordingly.

## 1.21.0 — 2026-08-25 (review pass v21: correctness, consistency, completeness)

### Correctness fixes

- AC-057's example IP-literal seed was `http://93.184.216.34/` — a globally
  routable address (example.com's former A record). DOC-17's preamble promises
  hermetic fixtures ("a local HTTP test server + recorded responses"), and
  IN_SCOPE URLs are enqueued and fetched, so a suite following the example
  would contact the real internet — non-deterministic and offline-breaking.
  The example is now `http://127.0.0.1/`, served by the local fixture under
  the [CFG-042]=true escape hatch [R-405]; scope matching is lexical over
  Registrable Domains, never DNS-dependent, so the AC's verdicts are
  unchanged.

### Consistency fixes

- NFR-004's generation-bound sentence contradicted itself on [CFG-027]=0:
  "else one" was the complement of "recrawl and retention are both enabled",
  so it covered [CFG-027]=0 — the very case the same sentence's parenthetical
  had just exempted ("this NFR claims no bound"). Restructured into explicit,
  disjoint cases: both enabled ⇒ ceil(...) generations; [CFG-025]=0 ⇒ one
  generation per URL; [CFG-027]=0 ⇒ keep forever, no bound claimed whatever
  [CFG-025] is.

### Completeness additions

- DOC-16 §3 declares a hard per-page resource cap ("URLs per page: 1000
  extracted + overflow flag", enforced pre-allocation), but DOC-10 §2 — which
  owns extraction — pinned no cap on the combined candidate set from its six
  sources; the only 1000 there was the outlinks artifact cap, which bounds
  the recorded list, not ingestion. An adversarial page could therefore flood
  discovery unboundedly while nominally violating the declared cap, and two
  conformant implementations would diverge. New R-159 pins it: after
  resolution, normalization, filtering ([R-153], [R-155]) and within-page
  deduplication ([R-154]), at most 1000 candidates per page proceed to
  ingestion — first 1000 in document order, overflow dropped with `truncated`
  set. The truncated-flag definition includes the new cap ([DOC-10 §3]), the
  urls_discovered_total{dropped} definition covers cap overflow — keeping the
  four-bucket partition exact ([DOC-15 §1]) — and AC-062 verifies the cap.

### Versioning

- KB version 1.20.0 → 1.21.0; touched documents (DOC-05, DOC-10, DOC-15,
  DOC-16, DOC-17) bumped accordingly.

## 1.20.0 — 2026-08-25 (review pass v20: correctness, consistency, completeness)

### Correctness fixes

- Retention could manufacture the exact dangling reference [INV-2] forbids:
  the blob-deletion step fired whenever no `pages` row referenced a blob, but
  an in-flight attempt may be about to commit one — a freshly staged write in
  the [T-3] crash window, or a 304 completion [R-144] re-affirming a hash
  whose last referencing row the same sweep had just deleted. Blob deletion
  now additionally requires that no fetch attempt is in flight
  ({ST-110, ST-120}), re-evaluated every sweep ([R-502], [DOC-11 §6]);
  crash-orphaned blobs remain collectible at the first post-restart sweep.
  AC-031 extended.

### Consistency fixes

- R-212 justified gate (e)'s race-freedom with mechanisms that do not bear
  the weight (C1 ingestion serialization and the [R-103] sweep — neither
  covers worker-side [T-2] completions). The load-bearing fact is now stated:
  a concurrent completion never raises successes(D)+inflight(D) (a success
  converts the record's own inflight unit into a success, net 0; a failure
  lowers the sum), so only [R-062] target upserts can lift the sum mid-window,
  and [FR-005] sanctions exactly that excess; an in-transaction re-check of
  (e) would wrongly convert that sanctioned transient excess into ST-190
  exclusions.
- T-2 attributed host counters to "the Host of the final response" —
  undefined for response-less outcomes such as timeouts; aligned with
  [R-112]'s "the exchange that produced the outcome".
- AC-032 demanded "zero duplicate ingestions", which the v1.19.0 metric pin
  makes impossible: config-seed re-ingestion counts as
  urls_discovered_total{duplicate} by definition [DOC-15 §1]. AC-032 now
  states the [NFR-012] property directly — no re-created records, rediscovery
  semantics only — and routes the counter expectation to the duplicate
  bucket.

### Completeness additions

- bytes_downloaded_total's "every response body received" basis was unpinned
  for robots.txt exchanges: bodies received through the same fetch machinery
  but not page fetches, so implementations could diverge on counting them
  (R-240's closed-enum discipline). Pinned: included, classed by their
  Content-Type like any body [DOC-15 §1].
- exclusions_total read as either a live stock ("ST-190 records") or a
  monotonic counter; the gauge excluded_count already carries the live count.
  Pinned: cumulative entries into ST-190, monotonic [DOC-15 §1].
- The DEAD/EXCLUDED retention sweep could delete `is_seed=true` records,
  silently shrinking the Crawl Scope set [FR-006] derives from that flag —
  permanent for runtime-injected seeds, which no startup re-ingests. Seed
  records are now exempt from the step ([DOC-11 §6]). AC-043 extended.

### Versioning

- KB version 1.19.0 → 1.20.0; touched documents (DOC-11, DOC-12, DOC-15,
  DOC-17) bumped accordingly.

## 1.19.0 — 2026-08-25 (review pass v19: correctness, consistency, completeness)

### Correctness fixes

- R-051 counted the dispatch-time cap gate [FR-011(e)] among unit-release
  cases, but that exclusion fires on an ST-100 candidate before [T-1]
  ([DOC-12 §3]): the record holds no units yet, and a literal implementation
  following R-051's list would decrement counters it never incremented —
  corrupting [INV-3]/[FR-011(b)] accounting. R-051 now states the no-unit
  fact for pre-dispatch exclusions explicitly.
- R-231 claimed `attempts` "resets to 0 only when the record leaves ST-140
  for ST-100", denying the operator DEAD reset's documented `attempts := 0`
  ([DOC-13 §4], [DOC-07 §2] edge) — an implementation treating R-231 as
  exhaustive would preserve exhausted budgets across resets. The reset path
  is now listed (same defect class as the v1.16.0 R-201 narrowing).

### Consistency fixes

- Priority recomputation points were enumerated inconsistently: [FR-006]
  recomputes priority when ingestion sets `is_seed=true` on a pre-existing
  record, while [R-201]/[DOC-12 §1] declared their three-point list
  exhaustive ("never recomputed mid-cycle"). The seed-flag case joined both
  enumerations.
- INV-5 said rediscovery "MAY update `last_seen_at`", but every [FR-051]
  branch mandates the update — the invariant understated a MUST.
- The `next_attempt_mono` field-hygiene principle ("a backoff timer must not
  survive outside ST-150", [DOC-13 §4]/[R-062]) was violated by two legal
  exits that left stale timers behind: the ST-150→ST-100 promotion
  ([DOC-12 §1]) and the [R-103] robots-unknown sweep (ST-150→ST-190). Both
  now clear the column; with this and the existing writers,
  `next_attempt_mono` is non-NULL only in ST-150. AC-027 and AC-053
  extended to verify it.

### Completeness additions

- `urls_discovered_total{duplicate}` had no defined trigger — the last
  unpinned bucket of that counter (same defect class as the v1.16–v1.18
  pins): whether a rediscovery, a seed re-injection hit, or an upsert onto
  a pre-existing record incremented it was unspecified and counts could
  diverge across conformant implementations (R-240's closed-enum
  discipline). Pinned: `duplicate` = ingestion events finding a
  pre-existing record ([FR-051]/[INV-5], seed re-injection [FR-006],
  [R-062] upserts onto pre-existing records), making the four buckets an
  exact partition of every ingestion event [DOC-15 §1].
- `bytes_downloaded_total`'s counting basis was undefined for discarded
  bodies: an [ERR-008]-rejected type or an [ERR-007] mid-stream abort could
  plausibly count or not, diverging implementations. Pinned: every received
  body counts (abort bodies up to the abort; a `304` body contributes
  nothing) [DOC-15 §1]. AC-052 extended.

### Versioning

- KB version 1.18.0 → 1.19.0; touched documents (DOC-03, DOC-07, DOC-08,
  DOC-12, DOC-13, DOC-15, DOC-17) bumped accordingly.

## 1.18.0 — 2026-08-25 (review pass v18: correctness, consistency, completeness)

### Correctness fixes

- FR-042 mandated resolving every discovered link "against the final response
  URL (post-redirect)", directly contradicting [R-021]/[R-153], which let a
  valid `<base href>` override the final URL as resolution base — an
  implementation following FR-042 verbatim would mis-resolve outlinks on
  base-href pages and violate [R-157]'s determinism contract for them.
  FR-042 now resolves against the extraction base ([R-020] with the [R-021]
  override); `discovered_from` remains the page's final URL identity, glossed
  as provenance rather than resolution base. AC-040 extended with a
  `<base href>` fixture.

### Consistency fixes

- The `page_artifacts.final_url_identity` column comment cited only [R-020]
  as "the link-resolution base", while its owning definition [DOC-10 §3] and
  the determinism framing treat the base as [R-020]/[R-021] (the payload's
  own `<base href>` participates). Comment aligned.
- The operator DEAD reset ([DOC-13 §4], [DOC-07 §2] edge label) left a stale
  `next_attempt_mono` in place — inert while the record sits in ST-100, but
  contradicting the field-hygiene principle [R-062] pinned for upsert
  landings ("a stale backoff timer must not survive" outside ST-150). Both
  statements now clear it; the reset's `once_retried_classes` clearing is
  cross-referenced to [R-232].

### Completeness additions

- `urls_discovered_total{dropped}` was ambiguous for runtime-seed rejects:
  [FR-006] demands rejection "identically to discovery-time discards"
  ([R-001]/[R-002]), but whether those rejects increment the dropped bucket
  (they have no URL Identity, yet the counter needs none) was unspecified and
  counts could diverge across conformant implementations (R-240's
  closed-enum discipline). Pinned: they count as `dropped` [DOC-15 §1].

### Versioning

- KB version 1.17.0 → 1.18.0; touched documents (DOC-04, DOC-07, DOC-11,
  DOC-13, DOC-15, DOC-17) bumped accordingly.

## 1.17.0 — 2026-08-25 (review pass v17: correctness, consistency, completeness)

### Correctness fixes

- Retention contradicted its own "0 = keep forever" parameters: [DOC-14]
  defines [CFG-027]=0, [CFG-033]=0, and [CFG-043]=0 as *keep forever*
  (and [NFR-004] reasons explicitly about the [CFG-027]=0 case), but every
  §6 deletion condition was stated as a bare threshold — "older than 0
  days" / "`fetch_ts` < now − 0" — which would purge the entire store on
  each sweep at those settings. Each of the three steps now states its
  zero-disables-deletion semantics. AC-043 extended.

### Completeness additions

- T-2 enumerated `last_error_class` writers incompletely: it listed only
  PERMANENT-sets / success-clears / ordinary-RETRYABLE-unchanged, but a
  RETRYABLE outcome that exhausts the budget enters ST-180 and sets the
  class ([DOC-13 §3]'s DEAD branch, echoed by the schema comment and
  [DOC-13 §4]'s "retain last error class"). An implementer reading only
  T-2 could leave the column NULL or stale on budget exhaustion. T-2 now
  names the DEAD branch; AC-024 extended to assert the recorded class.
- The counting basis of `exclusions_total` was ambiguous for OUT_OF_SCOPE
  drops under [CFG-038]=false: they produce no ST-190 record, so whether
  they increment a metric whose Meaning is "ST-190 reasons" was undefined
  and counts could diverge across conformant implementations (R-240's
  closed-enum discipline). Pinned: the series counts ST-190 records by
  reason code and is exposed regardless of [CFG-038]; drops are counted
  only by `urls_discovered_total{dropped}` [DOC-15 §1]; CFG-038's note
  aligned.

### Versioning

- KB version 1.16.0 → 1.17.0; touched documents (DOC-11, DOC-14, DOC-15,
  DOC-17) bumped accordingly.

## 1.16.0 — 2026-08-25 (review pass v16: correctness, consistency, completeness, unambiguity)

### Correctness fixes

- Stale `last_error_class` survived recovery: [R-062]/AC-061(d) require a
  RETRYABLE redirect-chain upsert to set the target's `last_error_class`, but
  no success path ever cleared it — [T-2] owned the column "on failures" only,
  and [DOC-13 §3] writes it only on DEAD/PERMANENT. A record upserted into
  ST-150 with the chain's class, then promoted ST-150→ST-100 and fetched
  successfully by itself, carried that stale failure class through
  ST-130/ST-140 indefinitely — contradicting the column's documented meaning
  ("the outcome that entered ST-180") and reporting a failure that no longer
  exists. [T-2] now owns `last_error_class` on both branches (PERMANENT sets,
  successful outcomes clear, ordinary RETRYABLE outcomes leave unchanged);
  the schema comment states both writers; R-062 notes the class's transience;
  AC-061(d) extended to verify the clear-after-recovery.
- R-201 was narrower than every other assignment of priority recomputation:
  [DOC-12 §1], DOC-07's operator-reset edge label, FR-051, and [DOC-13 §4]
  recompute on recrawl due, rediscovery refresh, and operator reset, while
  the normative rule named only recrawl — an implementation treating R-201 as
  the rule would skip recomputation on two legal paths (same defect class as
  the v1.4.0 R-231 unification). R-201 now lists all three.
- DOC-16's front-matter version was not bumped in the 1.15.0 pass although its
  §5 was edited there ("touched documents … bumped accordingly"); repaired to
  1.15.0.

### Completeness additions

- `urls_discovered_total` was undefined for URL Records created by redirect
  final-target upserts: they are not [FR-003]/[FR-004] ingestions, so whether
  they increment the counter was unspecified and counts could diverge across
  conformant implementations (R-240's closed-enum discipline). Pinned:
  upsert creations count as `ingested` [DOC-15 §1].
- ERR-018's host-counter treatment was defined only implicitly: R-112
  enumerates exactly which RETRYABLE classes increment
  `consecutive_failures` (ERR-018 absent ⇒ no increment) and calls out
  ERR-010 explicitly, but ERR-018 — also RETRYABLE — went unmentioned,
  inviting a plausible mis-implementation that increments it. The exclusion
  is now explicit, with the rationale (the wait reflects the target Host's
  own Crawl-delay, and the last exchange ran against the redirecting Host).
- R-141 still said "log anomaly metric" without naming it; the rule now names
  `content_length_mismatch_total` [DOC-15 §1] (the metric table already
  back-referenced R-141).

### Consistency fixes

- FR-030 glossed `robots_state` = INITIAL as "before any first fetch to that
  Host", but a Host whose robots acquisition failed has had fetch activity
  while legitimately remaining INITIAL — the operative criterion is the
  authoritative verdict ([R-106]). Gloss aligned.
- The `pages` primary-key comment justified same-run refetch upserts with an
  "operator-triggered refresh" — no such action exists in the operator API
  surface ([DOC-16 §5]: inject seeds, DEAD reset, drain). Replaced with the
  defined cause: rediscovery refresh under [CFG-021]=true [FR-051].

### Versioning

- KB version 1.15.0 → 1.16.0; touched documents (DOC-04, DOC-07, DOC-08,
  DOC-09, DOC-11, DOC-12, DOC-15, DOC-17) bumped accordingly; DOC-16 repaired
  to 1.15.0 for the missed prior-pass bump.

## 1.15.0 — 2026-08-25 (review pass v15: correctness, consistency, completeness, unambiguity)

### Correctness fixes

- DOC-07 §1's terminal note asserted that ST-140 "re-enters ST-100 only via
  recrawl [FR-050], rediscovery refresh [FR-051], **or the [R-062] upsert**",
  but the upsert never lands a record in ST-100 — its edges go directly to
  fetch-outcome states ([DOC-07 §2]). The note now separates ST-140's exits:
  the two ST-100 re-entry paths (recrawl, rediscovery refresh) versus the
  upsert's direct move to ST-130/ST-150/ST-180 without passing through
  ST-100.

### Completeness additions

- DOC-10 §4 mandates a parse-failure reason string "recorded on the `pages`
  row only", but the `pages` schema had no column to store it in.
  `pages.parse_reason` (NULL when `parse_ok=true`) now carries the ERR-009
  detail [DOC-11 §1].
- R-310's decompression-bomb abort named no error class; it is now pinned to
  the over-cap classification of [FR-023] — outcome PERMANENT, error_class
  ERR-007, no partial payload persisted. AC-022 extended with the
  decoded-size case.

### Consistency fixes

- DOC-09 §2's total-transfer timer note ("resets at each redirect hop; it
  does NOT span the whole chain") read as contradicting [R-145]'s
  "`timings_ms.total` spans the entire chain": one governs the enforcement
  timer, the other the reported metric. The note now states both explicitly.
- DOC-16 §5 said operator actions are served by the HTTP listener "when
  [CFG-034] is set", omitting [R-406]'s disablement on non-loopback binds;
  now scoped to loopback addresses with the cross-reference added.
- DOC-08 §2.3's deferral-backoff parenthetical buried the [CFG-040] cap after
  a clause about streak clearing ("...is cleared when an authoritative
  verdict is obtained — cap [CFG-040]"), making the cap's attachment
  ambiguous; restructured so the cap attaches to the backoff and the
  streak-clearing reads cleanly (no semantic change).
- R-211's wake-source clause contained a garbled verb form ("creates or
  re-due-s a candidate"); rewritten as "creates a candidate, makes a record
  due".

### Versioning

- KB version 1.14.0 → 1.15.0; touched documents (DOC-07, DOC-08, DOC-09,
  DOC-11, DOC-12, DOC-16, DOC-17) bumped accordingly.

## 1.14.0 — 2026-08-24 (review pass v14: correctness, consistency, completeness, unambiguity)

### Correctness fixes

- Global-concurrency TOCTOU on [FR-011(b)]: the gate was verified only at
  selection time, but robots exchanges acquire a global unit from outside
  the dispatch loop — a hop-gate query on a fetch worker can initiate an
  acquisition [R-105], [R-131] — so a robots acquisition landing between
  selection and [T-1] could push global inflight past [CFG-010].
  [DOC-08 §2.2] had made the robots side's acquisition atomic, but nothing
  said the page side's was. New R-212: the scheduling loop is the sole
  executor of [T-1] (dispatches strictly serialized), the global counter is
  in-memory and mutated only atomically (acquisitions are
  check-and-increments shared by dispatch and robots exchanges; releases
  are decrements), and a dispatch finding the pool full at commit time
  commits nothing — the record stays ST-100 and is re-selected on the next
  wake (unit-releasing events are wake sources [R-211]).

### Completeness additions

- Runtime seed injection onto a pre-existing record was undefined:
  [FR-006] said injections "behave identically to config seeds", but for
  an identity that already has a URL Record nothing said whether `is_seed`
  flips — and the scope set ([DOC-06 §4]) is derived from seed records, so
  an operator injecting a seed to expand scope could silently get no
  expansion. Ingesting a seed onto an existing record (config re-ingestion
  or injection) now sets `is_seed=true` idempotently, recomputes priority,
  and otherwise follows rediscovery semantics [FR-051] — injection never
  re-activates DEAD/EXCLUDED records [DOC-13 §4]. AC-058 extended.
- Interim `1xx` responses had no classification for page fetches: a stream
  that ends (or stalls) without a final status matched no row of the
  DOC-09 §4 table. New row: `1xx` is never an outcome — interim responses
  are consumed by the transport, the [CFG-014] header timer runs until the
  final status line's header block, and its expiry classifies ERR-012
  (robots already treated a final `1xx` as deferral [DOC-08 §2.3]); the
  §2 timer note aligned.
- [R-062] creation-time vs. overwrite fields: the rule assigns `depth` and
  `discovered_from` (and, implicitly, `source_run_id`, `raw_first_seen`,
  `is_seed`) to the target record but never said whether an upsert onto a
  pre-existing record rewrites them — an implementer could clobber the
  record's provenance (e.g. replace a discovered record's depth-0 lineage).
  These fields are now written only at creation; a pre-existing record
  keeps them, and the upsert's field effects are exactly those R-062/[T-2]
  name. New AC-061(g).
- Glossary **Host** did not define its port component: "a
  `(scheme, hostname, port)` triple" is ambiguous about whether the port is
  the URL's explicit port or the scheme default — `http://h/` and
  `http://h:80/` must share one Host key for robots, politeness, and scope.
  The port is now defined as the *effective* port (explicit if present,
  else the scheme default).
- [CFG-025], [CFG-027], [CFG-043] carried no explicit lower bound
  ("0 = never/keep forever" implies but does not state ≥0); the range
  column now states it so [V-1] validation is total for every key.

### Consistency fixes

- Every reference of the form "[DOC-08 §2.2]"/"§2.3"/"§2.4" (glossary,
  [DOC-09], [DOC-11], [DOC-13], [DOC-17]) pointed at subsections that did
  not exist as headings — DOC-08 §2 was a numbered list, so the anchors
  were unresolvable and an implementer had to guess that "§2.3" meant
  list item 3. The four items are now real subsections (§2.1 Cache lookup,
  §2.2 Fetch, §2.3 Status interpretation, §2.4 Cache stores), in-text
  "per item 3"-style references were normalized to the new anchors, and
  R-105's stale historical aside ("previously its initiator was implicit in
  §2.1's 'else fetch'") was dropped.
- [R-000] now states the specific-over-general tiebreak the README prose
  asserted but the normative rule omitted: within a level, the more
  specific document wins; R-nnn rules rank with FR/NFR except the security
  and politeness blocks already claimed by levels (1)–(2).
- [DOC-07 §1]'s ST-110 description ("holding its global unit and
  source-Host unit") contradicted [R-051]'s unit-transfer rule once a
  chain has hopped (the held unit belongs to the Host of the most recent
  request, not the source's). Now "at most one Host unit [R-051]".
- Unresolvable named section references normalized: "[DOC-03 §Deployment]"
  → "[DOC-03 Deployment model]" and "[DOC-01 §Success]" → "[DOC-01
  Success criteria]" (the headings are unnumbered).

### Versioning

- KB version 1.13.0 → 1.14.0; all touched documents bumped accordingly.

## 1.13.0 — 2026-08-24 (review pass v13: correctness, completeness, consistency, unambiguity)

### Correctness fixes

- The DOC-07 §2 transition list omitted every [R-062] upsert-overwrite
  edge: R-062 (and AC-026, AC-061) require a redirect final-target upsert
  to *replace the state of a pre-existing record* — e.g. ST-190/`CAP_REACHED`
  →ST-130 on success, ST-180→ST-150 on a retryable outcome, ST-140→ST-180 on
  a permanent one — but the machine listed only the creation arrow for new
  target records while declaring "no other transitions exist … any observed
  other transition is a defect". An R-062-conformant upsert onto an existing
  record would itself be a "defect", and AC-052's only-legal-pairs metric
  check would fail on it (same defect class as the v1.8/v1.11/v1.12
  missing-edge fixes). Grouped edge added, with the no-metric rule for
  same-state landings (ST-150→ST-150, ST-180→ST-180, ST-190→ST-190) so the
  metric check stays decidable. New AC-061(e).
- NFR-004's disk bound was false at the [CFG-027]=0 edge: the
  generations-retained formula ceil([CFG-027]×86400 / …) evaluates to 0
  when retention is disabled, while [CFG-027]=0 in fact means *keep
  forever* — pages never expire, so growth across recrawl generations is
  unbounded. The bound is now scoped to [CFG-025]>0 *and* [CFG-027]>0, with
  the keep-forever case explicitly excluded from the NFR's claim (operator
  choice, not a system property).
- R-132 loop detection did not include the original identity in the
  repetition check, so A→B→A was only catchable on a *later* B — yet AC-021
  requires detection "at first repetition". The rule now fires when a hop
  target equals the original identity or any earlier hop target. This in
  turn exposed a contradiction in R-133: `redirect_chain` "never [contains]
  the original identity", yet the loop-refused hop target *is* the original
  identity and AC-015/AC-020 require refused hops to be recorded. R-133 now
  says the list holds hop targets only (never the requested URL as such) and
  a loop-refused target equal to the original identity is still recorded.
  AC-021 extended to pin both.

### Completeness additions

- robots.txt status interpretation had no row for a final response that is
  neither 2xx, 4xx, 5xx/network-error, nor decode-failure, and not a
  followable redirect — concretely a spurious `304` (the robots request
  sends no validators, but servers MUST NOT is not a guarantee [R-144]) or
  a `300`. Such a verdict was undefined, so implementations could diverge
  (e.g. treat 304 as "reuse cache", which no rule defines). DOC-08 §2.3 now
  routes every such final status to UNKNOWN/deferral, fail closed
  [DEC-007]. AC-013 extended.
- Page-fetch classification had the mirror gap: a `3xx` outside the
  followable set {301, 302, 303, 307, 308} — e.g. `300`, with or without a
  `Location` — matched no row of the DOC-09 §4 table (only
  missing/unparsable `Location` was covered). R-130 and the table now
  terminate such an attempt PERMANENT/ERR-011, with the `304` carve-out
  noted (success-unchanged per §4).
- [R-062] field hygiene on overwrite upserts was unspecified: moving a
  record out of ST-190 left a stale `exclude_reason` (the schema defines
  the column only for ST-190 [DOC-11 §1]), and landing outside ST-150 left
  a stale `next_attempt_mono`. Both are now cleared by rule. New AC-061(e).
- [R-062] yes-once accounting ([R-232]) was unspecified for the target
  record: a chain failing ERR-003/ERR-013 on its final hop appends the class
  to the source's `once_retried_classes`, but nothing said whether the
  upserted target's list is touched — leaving the target's own later retry
  cycle with undefined once-only semantics. The mirror rule is now
  normative: the class is evaluated against (and appended to) the target's
  own list; already-listed ⇒ PERMANENT for the target. New AC-061(f).
- T-2's transaction set said only "urls.state update" while several rules
  make [T-2] the commit point for other URL columns: the recrawl
  `due_at_mono` and `consecutive_unchanged` [DOC-12 §4] (R-211 already
  asserted this), and on failures `last_error_class` and
  `once_retried_classes` [DOC-13 §3], [R-232]. The set now names them, so
  crash-atomicity coverage is explicit rather than inferred.
- DOC-11 §5's index list omitted every index needed by the DEAD/EXCLUDED
  url-record retention scan (the §6 sweep keys on `last_seen_at` else
  `updated_at` filtered by state). Added (state, updated_at) and
  (state, last_seen_at).

### Consistency fixes

- R-240 claimed "all labeled values come from closed enums in this KB",
  but `inflight_per_host{host}` and `host_next_allowed_ms{host}` label on
  host keys — not a closed enum. The rule now scopes the closed-enum
  guarantee to everything except the per-host series' `host` label
  (cardinality bounded by contacted Hosts).

### Versioning

- KB version 1.12.0 → 1.13.0; all touched documents bumped accordingly.

## 1.12.0 — 2026-08-24 (review pass v12: correctness, completeness, consistency, unambiguity)

### Correctness fixes

- The DOC-07 §2 transition list was missing the crash-recovery edge
  ST-110→ST-180: [R-060] reclassifies every {ST-110, ST-120} record at
  startup and sends budget-exhausted ones to ST-180, but the machine listed
  the crash edge from ST-110 only to ST-150 while declaring every unlisted
  transition a defect — an R-060-conformant restart would itself be a
  "defect", and AC-052's only-legal-pairs metric check would fail on it
  (same defect class as the v1.8.0 missing-edge fixes). Edge added; AC-030
  extended to kill during either in-flight state.
- [R-062] retryable upserts could strand a record in ST-150 forever:
  overwriting a terminal ST-180 record whose `attempts` = [CFG-020] (the
  normal state of a budget-exhausted DEAD record) with a chain's RETRYABLE
  outcome produced an ST-150 record holding attempts ≥ [CFG-020] — but the
  ST-150→ST-100 promotion requires attempts < [CFG-020] [DOC-12 §1], the
  defensive ST-150→ST-180 edge is unreachable [DOC-07 §2], and retention
  deletes only DEAD/EXCLUDED records, so the record could never leave
  ST-150. A RETRYABLE-outcome upsert now lands in ST-180 when the target's
  own budget is exhausted, restoring the invariant that ST-150 records
  always hold attempts < [CFG-020]. New AC-061(b).
- T-2 hop-abort unit accounting risked a double decrement: it said the
  [ERR-018]/[ERR-010]/[ERR-017]/[ERR-019] aborts "release exactly the unit
  held, on the Host whose response carried the unfollowable redirect" —
  but under [R-051]/[R-131] release-before-acquire that unit was already
  released when the redirect response arrived (R-131's own robots-await
  note says "no Host unit held"), and R-131 never pinned where the hop
  gate checks sit relative to the release. The protocol now fixes the
  order — release at response receipt, then gate evaluation holding no
  unit, then the wait, then acquire immediately before the send — and T-2
  decrements the unit of the most recent request still in flight (none
  for pre-send aborts).
- [R-401] vs. the test harness: the fixture server had to bind port
  80/443 (privileged ports) because no escape hatch existed for the port
  restriction — [CFG-042] relaxed only R-400.2 — making every
  local-fixture AC impractical in unprivileged CI despite NFR-009.
  [CFG-042]=true now also relaxes [R-401] (test-only, WARN, and
  production-forbidden unchanged) [R-405].

### Completeness additions

- `robots_state` was never assigned by any rule: the enum
  (INITIAL|OK|ALLOW_ALL|DEFERRED) existed in the schema, but nothing mapped
  outcomes to states or states to gate verdicts — e.g. the verdict after a
  deferral expires but before the [R-105] retry completes was undefined.
  New R-106: parsed 2xx ⇒ OK, 4xx ⇒ ALLOW_ALL, deferral ⇒ DEFERRED,
  authoritative verdict clears DEFERRED; the gate returns UNKNOWN iff
  robots_state ∈ {INITIAL, DEFERRED}. AC-013 asserts the transitions.
- [R-062] scope for aborted chains was undefined: "on completion of a
  redirect chain … the chain's final target" did not say whether a chain
  aborted before sending to its would-be next hop ([ERR-004]/[ERR-015]/
  [ERR-017]/[ERR-019]/[ERR-010]/[ERR-018]/loop/cap) upserts a target
  record — every hop rule says "target never fetched", yet R-062's
  failure states (ST-150/ST-180) invited the opposite reading. Now: the
  upsert applies only when the final request was sent; pre-send aborts
  record the outcome on the source only. New AC-061(c).
- [R-062] failure upserts left the target's `last_error_class` and
  `next_attempt_mono` undefined. Now: failure upserts set
  `last_error_class` to the outcome's class (success clears it), and a
  retryable upsert's `next_attempt_mono` mirrors the source's — one
  chain, one retry schedule, since the source's next attempt re-runs the
  chain [R-131] and re-upserts. New AC-061(d).
- [R-133] `redirect_chain` membership was ambiguous (does the list include
  the followed final target, or only blocked hops?): AC-015 requires
  never-fetched blocked hops in the list, so the symmetric definition is
  now normative — every hop target the chain acted on (followed, or
  refused at a gate/limit), never the original identity, empty for a
  no-redirect fetch. `final_url_identity` is likewise pinned: the URL of
  the last response received (self for no-redirect; the redirecting hop
  for a pre-send abort; the final target otherwise). AC-020 extended.
- DOC-09 §4 lacked a row for SSRF-blocked redirect hops — [R-400]/[R-402]
  define the outcome but the response-classification table omitted it
  (permanent, ERR-004, referring Host flagged suspicious, target never
  fetched). Row added.
- [R-141] referenced an undefined "anomaly metric" (R-240's closed-enum
  discipline has no such series). New `content_length_mismatch_total`
  counter [DOC-15 §1].
- Hardcoded cadences parameterized per DOC-14's every-tunable-has-a-CFG-id
  rule (same class as the v1.9.0 CFG-044 fix): new CFG-045
  `run_summary_every` (was "N=1000" [DOC-15 §4]) and CFG-046
  `retention_sweep_interval_s` (was "runs hourly" [DOC-11 §6]); the
  retention job is also identified as an in-process maintenance task
  [DEC-002].
- The operator DEAD reset [DOC-13 §4] and the [R-054] UNKNOWN compensation
  did not set `due_at_mono` (immediate candidacy was only inferable from
  the stale pre-dispatch value): both now set `due_at_mono` := now.
- ST-150→ST-100 retry promotions left priority handling unspecified (a
  determinism-relevant ordering input): they preserve the record's
  priority; priority is recomputed only on [R-052] transitions and the
  operator reset [DOC-12 §1].

### Consistency fixes

- `text/xml` was absent from [R-143]'s stored-non-HTML set while
  `application/xml` was included — both are common XML media types and
  RSS/Atom feeds are frequently served as `text/xml`. Added, with the
  [DOC-15 §1] `content_type_class` xml note aligned.
- AC-050's "in-flight complete ≤ total_transfer_timeout" is unsatisfiable
  for multi-hop chains ([CFG-015] resets per hop [DOC-09 §2]; inter-hop
  waits ≤ [CFG-035] are legal [R-131]) — and DOC-03's drain wording had
  the same over-tight bound. Both now state the real bound: completion
  within the task's bounded timers.
- NFR-004's "at most [CFG-005] distinct payloads per recrawl generation"
  now carries the [R-062] bounded-exemption qualifier ([FR-005]) — the
  total can exceed [CFG-005] by in-flight-chain final targets.
- Crawl Session "lineage" was undefined although the term anchors the
  Run-spanning semantics [DOC-03], [DOC-11 §1]: a Session is now the
  sequence of Runs over one Metadata Store lineage; the first Run against
  a fresh (empty) store begins a new Session [DOC-00].

### Versioning

- KB version 1.11.0 → 1.12.0; all touched documents bumped accordingly.

## 1.11.0 — 2026-08-24 (review pass v11: correctness, completeness, consistency, unambiguity)

### Correctness fixes

- The DOC-07 transition list was incomplete and self-contradictory: it
  showed only two record-creating arrows ((creation)→ST-100,
  (creation)→ST-190) and declared "no other transitions exist", yet the
  [R-062] redirect final-target upsert creates records directly in the
  fetch-outcome states (ST-130/ST-150/ST-180) — an implementer following
  DOC-07 literally would reject those transitions as defects, and
  [DOC-15 §1] hardcoded "the two record-creating transitions", leaving
  the upsert's creation metrics unspecified. Added the creation arrow
  (with its no-ST-100-phase rationale: every per-hop gate was verified in
  flight [R-131]) and aligned DOC-15.
- FR-005 contradicted itself: the enqueue-time clause asserted "the total
  of non-EXCLUDED records never exceeds [CFG-005]", while the same
  requirement exempts [R-062] final-target upserts from [CFG-005] (bounded
  by in-flight chains) — so the total CAN exceed the cap by exactly those
  upserts. The parenthetical is now qualified with the exemption.
- robots.txt TTL was the only scheduler-consumed expiry measured against
  wall-clock time (`robots_fetched_at`), violating [DEC-012] (scheduler
  time is monotonic) and breaking replay determinism [NFR-006], [AC-033]:
  every other gate-relevant expiry (politeness window, deferral, backoff,
  due times) is monotonic, but a wall-clock TTL drifts under virtual time
  and clock jumps. New `hosts.robots_fetched_at_mono` column is the TTL
  basis ([DOC-08 §2.1]); the TEXT field remains as an audit timestamp
  only [DOC-11 §1].
- FR-031's exclusion timing was undefined in the dispatch loop: the
  [DOC-12 §3] pseudocode waits for and picks only gate-PASSING
  candidates, so a due record whose robots verdict is DISALLOW that is
  never selected (e.g. outranked by higher-priority work on other Hosts,
  or its Host's window never opens) would linger in ST-100 forever —
  never dispatched, never excluded, invisible to the [R-103] sweep
  (UNKNOWN timeouts only) and to retention (which deletes only
  DEAD/EXCLUDED records). §3 now requires the ST-100→ST-190/
  `ROBOTS_DISALLOW` exclusion at verdict-evaluation time, with the same
  termination argument as gate (e).

### Completeness additions

- R-105's trigger list omitted the post-deferral case: after
  `robots_deferred_until_mono` expires, some event must re-initiate
  robots acquisition, but only INITIAL and TTL-expiry triggers were
  enumerated (the §2.3 retry's initiator was implicit in §2.1's "else
  fetch"). The first gate query after a deferral expires is now an
  explicit trigger.
- Robots redirect-hop failure causes were under-enumerated: "terminates
  without a final response" listed only redirect-cap exhaustion, SSRF
  block, and ERR-018 — omitting redirect loops [R-132], missing/
  unparsable/non-http(s) `Location` [R-130], and URL-blocklist matches.
  All are now listed, and the [CFG-037] blocklist is explicitly applied
  to robots redirect hops (R-131's not-bypassable rule, robots fetches
  included; previously unstated whether it applied at all).
- Robots-exchange unit acquisition atomicity: [DOC-08 §2.2] said the
  exchange "holds one per-Host and one global unit while in flight" but
  not how they are acquired. Now: both units are acquired atomically
  before the request is sent (mirroring [T-1]); while waiting for
  capacity the exchange holds no units — ruling out a capacity-wait
  cycle against a saturated global pool by the same release-before-
  acquire argument as [R-131].
- `robots_deferred_until_mono` is now explicitly cleared together with
  `robots_deferred_since_mono` and the streak counter when an
  authoritative verdict is obtained (a stale expiry value could
  otherwise mis-key R-211's sleep list).
- R-131 vs R-105 interplay: a hop robots check returning UNKNOWN merely
  because an acquisition is in flight on the target Host is AWAITED
  (bounded by the robots exchange's timeouts, no Host unit held) —
  ERR-010 aborts apply only to actual deferral. Previously "verdict is
  UNKNOWN" read as an unconditional abort, wasting a retry attempt on a
  healthy Host whose robots.txt was simply still downloading.
- R-102: multiple `Crawl-delay` lines in the applicable group — the
  first one applies (error-tolerant parsing ignores the rest).
- `meta_robots` tokenization defined ([DOC-10 §3]): tokens split on
  commas/whitespace; `none` ≡ `noindex,nofollow`; unknown tokens ignored
  — previously `none` and separator handling were unstated, yet
  [FR-045]'s noindex/nofollow behavior keys off this artifact.
- New AC-060: the configuration validation rules [V-1]–[V-6] had no
  acceptance criterion (V-4/V-5 were only partially covered via
  AC-002/AC-005); it verifies abort-before-any-I/O with an offending-key
  error for every V-rule.

### Consistency fixes

- "Terminal" was used in two senses: the DOC-07 §1 table marks ST-140
  Terminal?=yes (yet it has outgoing transitions), while [R-062] writes
  "Terminal records (ST-180, ST-190)". §1 now defines the term (fetch-
  cycle terminal, no failure-path exits) and notes that rules meaning
  {ST-180, ST-190} name the states explicitly.
- INV-1 said hop chains are "persisted on the source record's
  `redirect_chain`" — the column lives on the attempt's `fetch_events`
  row [R-133], [DOC-11 §1]; wording fixed.
- DOC-12 §4: "any full 200 resets it" → any full (non-304) 2xx success
  (a 204/206-style success also ends an unchanged streak; "200" read
  literally excluded them).
- FR-032 carried a duplicated trailing "[DOC-08 §4]" cross-reference;
  removed.

### Versioning

- KB version 1.10.0 → 1.11.0; all touched documents bumped accordingly.

## 1.10.0 — 2026-08-23 (review pass v10: correctness, completeness, consistency, unambiguity)

### Correctness fixes

- Redirect-hop concurrency accounting could deadlock (R-131/T-2/R-051):
  the model had a task hold its source Host's [T-1] unit for the whole
  chain AND acquire a target-Host unit per hop AND a second global unit
  per hop. Two cross-host chains A→B and B→A under [CFG-009]=1 each hold
  their source Host's unit while waiting for the other's — a wait cycle
  with no timeout covering slot waits (fetch timers only run during HTTP
  exchanges); a same-host redirect self-deadlocks at [CFG-009]=1, and small
  [CFG-010] deadlocks on the phantom second global unit (a chain runs in
  one worker and cannot consume two). Replaced with release-before-acquire
  unit transfers [R-051]: one global unit per task spanning [T-1]..[T-2]
  ([DOC-00] glossary now defines Global/Per-Host Concurrency in task/request
  terms); at most one Host unit per task, released when its response is
  received and re-acquired immediately before the next request, so a
  waiting task holds no Host unit and waits cannot cycle. T-2 decrements
  exactly the one unit still held (Host of the most recent request — which
  is also [R-112]'s counter-attribution Host); hop window advance made
  explicit (identical to [FR-012]); R-060's inflight rebuild restated as an
  exact reset-to-0 (a crashed record's unit Host is unrecoverable under
  transfers, and every unit-owning record is reclassified anyway). AC-011,
  AC-026, AC-055 aligned, incl. a new A↔B deadlock fixture.
- EffectiveDelay unit mismatch [DOC-08 §4]: `max(CFG-007 /*ms*/,
  crawl_delay(h) /*seconds*/, backoff_ms(h))` compared a raw seconds value
  against millisecond terms — a `Crawl-delay: 5` would be honored as 5 ms
  (same dimensional class as the v1.7.0 NFR-004 fix). The formula now
  converts: `1000 × crawl_delay_s(h)`, with all three terms annotated in
  milliseconds; FR-032 and AC-014 aligned (AC-014 previously wrote
  "CFG-007=5s" for a ms-valued parameter).

### Completeness additions

- New R-105: robots acquisition is single-flight per Host and lazily
  initiated (first gate query against an INITIAL Host, or first query after
  TTL expiry [R-104]); concurrent queries await the in-flight exchange;
  acquisition completion is a scheduler wake source. Previously nothing
  defined who initiates the robots fetch or deduplicated concurrent gate
  queries, and R-211's wake list covered only revalidation completions.
- Robots deferral-backoff streak persisted: `hosts.robots_defer_failures`
  column added [DOC-11 §1] — the "×2 per consecutive failure" escalation
  [DOC-08 §2.3] otherwise resets on restart (same phantom-state class as
  the v1.9.0 once_retried_classes fix).
- R-144 generalized to all 304s with no usable stored payload: retention-
  deleted blobs (already covered) plus first-fetch/unconditional 304s
  (previously undefined); an unconditional refetch that also returns 304 is
  PERMANENT/ERR-014 — the guard bounds the refetch to one per attempt
  (otherwise a broken server could loop it). New AC-059 (the R-144 path
  had no acceptance criterion at all).
- FR-005(1): the enqueue-time [CFG-005] check and record insertion are one
  transaction (ingestion serialized in C1) — concurrent discoveries could
  otherwise overshoot the total (TOCTOU; [CFG-005], unlike [CFG-006], has
  no dispatch-time gate to catch an overshoot).
- New V-6: [CFG-003] must be a non-empty subset of {http, https} (an empty
  set silently crawls nothing).
- DOC-15 §1: `urls_discovered_total` gains outcome `dropped` for no-record
  discards ([R-001]/[R-002], OUT_OF_SCOPE with [CFG-038]=false) — R-240's
  closed-enum rule had no value for them.

### Consistency fixes

- R-211 attributed recrawl-due installation to extraction completion
  (ST-130→ST-140), contradicting [DOC-12 §4]/[T-2] ("computed at fetch
  completion"); [T-2] itself was not a wake source (with an empty
  ST-100/ST-150 frontier the loop could sleep past a recrawl due time
  installed by a completion), and first-time robots acquisitions were not
  covered by "revalidation completion". Wake sources now: fetch completion
  [T-2], extraction completion (can make a record due immediately), robots
  acquisition/revalidation completion ([R-104], [R-105]).
- AC-055 "no worker slot … held during the wait" contradicted the accepted
  design (waits ≤ [CFG-035] legitimately hold the task's worker — that is
  why ERR-018 exists as a threshold); the real invariants are: no Host unit
  held during any wait, and no worker held for an over-threshold wait.

### Versioning

- KB version 1.9.0 → 1.10.0; all touched documents bumped accordingly.

## 1.9.0 — 2026-08-23 (review pass v9: correctness, completeness, consistency, unambiguity)

### Correctness fixes

- R-062 redirect final-target upsert was undefined for pre-existing records,
  with two hazardous readings: overwriting a record in {ST-110, ST-120} (an
  independent fetch in flight) would corrupt its slot accounting [R-051] and
  `attempts`, while blanket-applying FR-051's "never re-activated" rule would
  discard a completed, gate-verified fetch. Now: terminal records are
  overwritten with the fetch outcome (all per-hop gates were re-checked
  [R-131]; success clears `last_error_class`) — explicitly not the forbidden
  automated re-activation [DOC-13 §4] nor a rediscovery [FR-051 gained the
  disambiguating cross-reference]; in-flight records keep their state and
  `attempts` (the chain records `last_seen_at` and its `pages` row only);
  other non-terminal records take the outcome state. AC-026 extended.
- Redirect-chain success accounting was unstated: T-2 described a single
  `pages` insert and R-062 said only "state per the fetch outcome" — leaving
  open whether the source, the target, or both get page rows and ST-130
  transitions. Now both rows commit in one [T-2] (source's with
  `final_url_identity` = target [FR-044]; target's with itself), and both
  records progress ST-130→ST-140 (identical payload + resolution base ⇒
  identical, upsert-idempotent artifacts [R-157]).
- R-232 ("yes (once)" classes) was ambiguous and unimplementable as written:
  "effective retry budget of 2 total attempts" reads both as a per-class
  occurrence rule and as an attempts-based cutoff (which would permanently
  fail a first-occurrence ERR-003 appearing at attempts=2), and no persisted
  state existed to count occurrences — fetch_events are retention-bounded
  [CFG-033]. Now: per class, at most one retry per attempt cycle, tracked in
  the new `urls.once_retried_classes` column, cleared exactly when `attempts`
  resets to 0; a repeat occurrence ⇒ PERMANENT. AC-024 extended.

### Completeness additions

- R-103 covered only records gated at the threshold expiry: discoveries and
  ST-140→ST-100 recrawl promotions arriving on a Host already past [CFG-040]
  of continuous deferral had no defined treatment. The sweep condition is now
  re-evaluated on every scheduler iteration; AC-027 extended.
- Robots-exchange slot lifecycle was dangling: §2.2 acquires a politeness
  advance plus a concurrency slot, but neither [T-1] nor [T-2] covers robots
  exchanges (no URL Record participates). The advance/acquire/release
  mapping is now stated explicitly (release when the exchange completes and
  its verdict/cache update commits).
- `urls.last_fetch_mono` had no writer (phantom column, same class as the
  v1.2.0 schema-gap fixes): now set to the attempt completion time in [T-2].
- New CFG-044 `robots_defer_backoff_start_s` (default 60): the robots-
  deferral backoff start was a hardcoded constant violating DOC-14's
  every-tunable-has-a-CFG-id rule (same class as the v1.1.0 CFG-035/036
  fixes); the ×2 doubling factor is now labeled fixed.
- CFG-031 boost prefixes: the matched string was never stated — now the
  case-sensitive URL-identity prefix [DOC-12 §2].
- [CFG-029] parameter-name extraction defined [DOC-06 §5]: split the query
  on `&`; name = text before the first `=`, or the whole token when no `=`.
- DOC-15: `state_transitions_total` gains the `from=creation` label for the
  two record-creating transitions (R-240's closed-enum rule had no value for
  them); counter lifetime across restarts defined (reset per Run;
  `run_summary` events and the `runs` table are the durable aggregates).
- R-221: `Cache-Control: no-store` now explicitly affects recrawl
  eligibility only in v1 — the payload is fetched, stored [FR-043], and
  extracted normally (the storage question was left implicit).

### Consistency fixes

- DOC-12 §3 pseudocode: "gates pass for c.host" contradicted gate (d) being
  per-URL [FR-011]; the wait condition now reads "gates [FR-011] a–d pass"
  with the per-Host (a–c) vs per-URL (d) split noted.

### Versioning

- KB version 1.8.0 → 1.9.0; all touched documents bumped accordingly.

## 1.8.0 — 2026-08-23 (review pass v8: correctness, completeness, consistency, unambiguity)

### Correctness fixes

- Crash recovery could exceed the Retry Budget and skipped retry backoff
  ([R-060] vs [DOC-13 §5]): resetting in-flight {ST-110, ST-120} records
  directly to ST-100 — a state that is immediately due — (a) re-dispatches a
  record whose final allowed attempt was in flight at crash time, so
  `attempts` would exceed [CFG-020] (violating the glossary Retry Budget
  invariant and FR-052), and (b) retries immediately with no per-URL backoff,
  contradicting §5's "equivalent to a RETRYABLE outcome". R-060 now
  reclassifies at startup per [DOC-13 §5]: budget exhausted ⇒ ST-180, else
  ST-150 with backoff per §3 (jitter seeded by hash(url_identity, attempt),
  as for any retry). The ST-110→ST-150 edge was added to the state machine
  (crash while ST-110 had no retryable exit; ST-120 used the existing edge);
  DOC-03's deployment text and AC-030 aligned. Crash-exhausted DEAD records
  carry `last_error_class` = NULL — the crash wrote no fetch_event and no
  class is fabricated (§5).
- Missing legal transition ST-100→ST-190/`ROBOTS_UNKNOWN_TIMEOUT` [R-103]:
  the sweep covers gated records in ST-100 or ST-150, but the state machine
  listed only the ST-150 edge and declares every unlisted transition a
  defect — a compliant R-103 sweep of queued records would itself be a
  "defect", and AC-027 exercises exactly that path. The ST-100→ST-190 edge
  label now includes the robots-unknown threshold; AC-027 extended to cover
  both gated states and to require the sweep to fire at the threshold expiry.
- NFR-004 generation bound ignored recrawl jitter: ceil([CFG-027] × 86400 /
  [CFG-025]) divides by the base interval, but jitter ([CFG-026], up to 0.9)
  only ever shrinks intervals — a URL jittered to the minimum retains
  1/(1−jitter)× more generations, so the "at most" disk bound was understated
  up to 10× at range extremes (same dimensional-analysis class as the v1.7.0
  fix). Divisor corrected to the minimum jittered interval
  [CFG-025] × (1 − [CFG-026]).
- T-2 released only one host slot for cross-host redirect chains: [R-131]
  (since v1.6.0/v1.7.0) holds one concurrency slot per in-flight hop on the
  target Host plus the source Host's [T-1] slot, but the completion
  transaction decremented "hosts.inflight−1" once — leaking a target-Host
  slot per cross-host completion until restart (same leak class as the v1.3.0
  R-051 fix). T-2 now decrements once per held slot and assigns host-counter
  attribution (final hop's Host), with the matching rule added to [R-112];
  AC-026 extended to verify both releases.
- `hosts.crawl_delay_s` was INT, truncating fractional `Crawl-delay` values
  (e.g. 0.5) and violating [R-102]'s "honored exactly as received" — with
  [CFG-007]=0 a truncated 0.5 s delay under-waits. Column is now REAL
  (seconds, exactly as received).

### Completeness additions

- `urls.last_error_class` column added [DOC-11 §1]: [DOC-13 §4] requires DEAD
  records to "retain last error class" and the operator reset to clear it
  (verified by AC-053), but no column existed and `fetch_events` (the only
  error-class bearer) are retention-bounded [CFG-033] — the value was
  unrecoverable after retention. The PERMANENT path of [DOC-13 §3] now names
  the column it writes.
- FR-006 recorded runtime seeds violating [FR-002] as ST-190 "with the
  applicable reason", but no reason code exists for invalid URLs and an
  unparseable URL has no URL Identity to store — [R-001]/[R-002] mandate
  discard-with-no-record. [FR-002] violations are now rejected with an error
  response and never recorded; [V-4] violations keep the
  ST-190/`OUT_OF_SCOPE` treatment (iff [CFG-038]=true). AC-058 aligned.
- [R-211] wake/sleep gaps (same class as the v1.6.0/v1.7.0 fixes):
  (a) the [R-103] threshold expiry (`robots_deferred_since_mono + [CFG-040])
  was not a sleep target — the sweep could fire a full deferral-backoff
  period late; (b) extraction completion (ST-130→ST-140) installs recrawl
  due times that may precede the current sleep target — with no ST-100/ST-150
  work pending, the loop could sleep past them; (c) robots revalidation
  completion ([R-104]) can gate or un-gate candidates but was not a wake
  source. All three added; R-103 now names the scheduling loop as the sweep
  actor.
- Robots fetches that terminate without a final response (redirect cap
  exhausted [R-130], hop SSRF block [R-400], [ERR-018] hop-wait abort) had no
  defined verdict: now a network-class failure ⇒ UNKNOWN/deferral, fail
  closed [DEC-007] [DOC-08 §2.2].
- R-144: a politeness wait > [CFG-035] during a 304 cache-miss refetch now
  aborts RETRYABLE/ERR-018 exactly like a redirect hop — the "modeled like
  [R-131]" phrase left the abort case implicit.
- New V-5 [DOC-14]: [CFG-018] must match the UA Token pattern [R-120] and
  [CFG-019] (when set) must be a valid email — both were described in the
  parameter table but never validated at startup.
- `<area href>` added as a discovery source [DOC-10 §2] — image-map links, a
  standard HTML link vector the candidate table omitted.
- Artifacts JSON truncation made deterministic [DOC-10 §3], [DOC-16 §3]:
  `headings` capped at 1000 entries (it was the one unbounded list, and is
  added to the `truncated` flag's cap list); when the serialized JSON still
  exceeds the 2 MiB row cap, fields are tail-truncated in a fixed order
  (headings → main_text → outlinks) — the old "truncate deterministic fields"
  named no order, so two conformant implementations could diverge and break
  [R-157]/AC-040.

### Consistency fixes

- PREFIX_LIST matching precision [DOC-06 §4], [CFG-039]: scheme/host compare
  case-insensitively (candidates are normalized lowercase; entries are used
  verbatim) and the path verbatim/case-sensitively; an entry with an empty
  path matches every path on its host while an entry path of `/` matches only
  the root — the surprising asymmetry is now stated instead of left to be
  inferred.
- DOC-12 §3 pseudocode: "pick c = highest-priority candidate" now says
  "gate-passing candidate" — the wait condition admits candidates by gates,
  and the pick must be from that same set (otherwise gates would be
  re-evaluated implicitly and [R-210]'s one-iteration bound is ambiguous).
- NFR-015: "within one scheduling round" → "one scheduling-loop iteration"
  (terminology drift vs [R-210]).
- DOC-06 §5 filter 2: "drop URLs …" → "exclude URLs …" — "drop" suggested
  no-record discard, contradicting the → ST-190/`TRAP_PARAM` outcome and
  [R-041] (same wording class as the v1.7.0 R-143 fix).
- DOC-09 §1: "no request headers beyond those specified" now exempts
  transport-mandated headers (`Host`/`:authority`, `Content-Length`,
  connection management) — as written, even HTTP itself was forbidden.
- `word_count` defined as the count of whitespace-separated tokens in
  `main_text` — AC-040's exact-JSON comparison was untestable without a
  tokenization rule.
- CFG-035's notes now mention its third role: the [ERR-018] hop-wait
  threshold [R-131].

### Versioning

- KB version 1.7.0 → 1.8.0; all touched documents bumped accordingly.

## 1.7.0 — 2026-08-23 (review pass v7: correctness, completeness, consistency, unambiguity)

### Correctness fixes

- NFR-004 dimensional error: `ceil([CFG-027]/[CFG-025])` divides days by
  seconds — with the defaults (90 d retention, 7 d recrawl) it yields 1
  generation instead of 13, understating the disk bound 13×. Corrected to
  `ceil([CFG-027] × 86400 / [CFG-025])` with the unit conversion stated
  inline.
- Recrawl promotion/wake gap (same defect class as the v1.6.0 R-211 fix):
  `due_at_mono` for recrawl is set at fetch completion [DOC-12 §4] while the
  record sits in ST-140, but no actor performed the ST-140→ST-100 transition
  when it expired, and R-211's earliest-of sleep list omitted recrawl due
  times — with no ST-100/ST-150 work pending, the loop could sleep past every
  recrawl forever. DOC-12 §1 now assigns both due-time promotions
  (ST-150→ST-100 on `next_attempt_mono`, ST-140→ST-100 on `due_at_mono`) to
  the scheduling loop before candidacy (resolving the tension with R-050's
  "only ST-100 visible"), and R-211's sleep list gains "next recrawl
  `due_at_mono` over ST-140".
- R-053/R-054 asymmetry: a send-time robots refresh to DISALLOW kept the
  [T-1] attempts increment while the UNKNOWN path rolled it back — yet
  neither sends a request, and R-053's own justification ("no request was
  sent, so no attempt occurred") applies to both. Both send-time aborts now
  roll back the increment and write no fetch_event; R-053, R-054, and
  AC-016 aligned.
- DOC-08 §2.3 mapped an "unparseable body" to UNKNOWN (conservative
  deferral), contradicting NFR-014: RFC 9309 parsing is error-tolerant —
  unrecognized or malformed lines are ignored — so a 2xx body always yields
  a rule set (possibly empty ⇒ ALLOW everything, exactly like 4xx), and
  "unparseable" never occurs. UNKNOWN is now reserved for 5xx, transport
  failures, and body-decode failures (e.g. Content-Encoding).
- Blocklist bypass via redirects: R-131 checked scope, robots, and SSRF per
  hop but not [CFG-037] — a site could 301 the crawler into blocklisted
  URLs, defeating the operator's "never crawl" policy. New error class
  ERR-019 `REDIRECT_BLOCKLISTED` (permanent; chain terminates at that hop,
  source → ST-180, target never fetched, hop recorded in `redirect_chain`);
  new DOC-09 §4 row; R-062's creation-point exemption narrowed to the trap
  filters ([DOC-06 §5] items 2–4) since scope/robots/SSRF/blocklist are all
  per-hop checks now; new AC-056(c).
- R-130: relative `Location` values (legal and common) had no resolution
  rule, and hop targets with a non-http(s) scheme or userinfo were
  unclassifiable. Each `Location` is now resolved against the current hop's
  URL per RFC 3986 §5; an unacceptable target terminates the chain as
  PERMANENT/ERR-011 (note extended). New AC-056(a)(b).
- Registrable Domain was undefined for IP-literal hosts, leaving CFG-006's
  per-domain key and SEED_DOMAINS scope undefined for IP-literal seeds. It
  is now the canonical literal string itself [DOC-00], and R-003
  canonicalizes IPv6 literals to RFC 5952 compressed lowercase form so
  `[2001:0DB8::1]` and `[2001:db8::1]` share one identity. New AC-057.

### Completeness additions

- Robots matching made implementable (R-100/R-101): group selection — a
  group matches when its `User-agent` value is a case-insensitive substring
  of the product name; most specific (longest) matching group wins; no
  matching group and no `*` group ⇒ empty rule set ⇒ allow all. Rule
  matching — byte-wise, case-sensitive, against the path plus `?query` when
  present (so `Disallow: /*?` works); exactly two special characters (`*`
  any sequence incl. empty, terminal `$` anchor); precedence by longest
  rule value (wildcards count as one); empty `Disallow` ⇒ allow all.
  AC-012 extended to cover these cases.
- FR-006 runtime-injection failure semantics: a runtime seed violating
  [FR-002] or [V-4] is now rejected with an error response (recorded ST-190
  where applicable and [CFG-038]=true) instead of the only defined behavior
  being V-4's startup abort — which would kill a live crawl. The Scope
  seed set is also defined as all `is_seed` URL Records, so runtime-injected
  seeds keep expanding scope after restart. New AC-058.
- R-062: the final-target upsert never modifies the target's `attempts` —
  the attempt is accounted on the source record [T-1].
- R-131: hop slot accounting — an in-flight hop holds one concurrency slot
  on the target Host and one global slot, acquired immediately before the
  hop request is sent and released at hop completion (symmetry with robots
  fetches; no slot is held during an [ERR-018] politeness wait).
- DOC-13 §4: the operator DEAD reset also recomputes priority per
  [DOC-12 §2] (previously unspecified, unlike every other ST-→ST-100 path).
- DOC-11 §5: retention sweep indexes added — (fetch_events.ts),
  (pages.fetch_ts), (pages.payload_sha256), (pages.final_url_identity,
  payload_sha256) — the §6 sweeps and blob/artifact reference checks had no
  index support.
- DOC-16 §5: with [CFG-034]=null the mutating operator actions were
  unreachable (no listener existed); they are now served over an
  implementation-defined local channel that accepts no network connections.
- README: the auxiliary ID families (G-, NG-, INV-, C1–C9, P-, T-, V-,
  EXT-*) are declared in the conventions section — they were used throughout
  but absent from the stated convention list.

### Consistency fixes

- FR-005(1): the enqueue-time [CFG-005] trigger condition was implied, not
  stated (unlike the precisely spelled-out [CFG-006] condition). Now
  explicit: count of non-EXCLUDED records ≥ [CFG-005] before insertion ⇒
  ST-190/`CAP_REACHED`; the total never exceeds [CFG-005].
- R-143: "Non-text types" → "Non-HTML types" (`text/plain` is a text type);
  the "set intersected with [CFG-028]" phrasing (a bool cannot be
  intersected) replaced by: effective allowed list = the fixed set when
  [CFG-028]=true, empty when false.
- CFG-038 renamed `log_exclusions` → `record_out_of_scope`: the parameter
  governs ST-190 audit rows, not logging (the `exclusions_total` metric is
  always emitted regardless).
- DOC-11 §6: "blobs left unreferenced by those deletions" was readable as
  only those orphaned by the sweep's own page deletions, contradicting
  AC-031 (crash orphans [T-3] must be swept). Now: all artifacts/blobs with
  no remaining `pages`-row reference after the pages step, explicitly
  including crash orphans.
- T-2: the `pages` insert applies to SUCCESS/UNCHANGED outcomes only;
  failure completions commit the same set minus it (the transaction was
  described as unconditional).
- FR-042: `discovered_from` for extracted links = the page's final URL
  identity (the extraction base [R-020]) — previously ambiguous between
  source and final identity after redirect chains.
- Glossary: Backoff distinguished from the per-URL retry delay ([R-230]
  both may apply); Effective Delay notes the [CFG-011] gating of the
  backoff term.
- DOC-07 §2: the ST-150→ST-180 ("attempts = CFG-020") edge annotated as
  defensive/unreachable — [DOC-13 §3] evaluates the budget at failure time,
  so ST-150 records always hold attempts < [CFG-020]; prevents confusion
  against AC-052's "only legal transitions" check.

### Versioning

- KB version 1.6.0 → 1.7.0; all touched documents bumped accordingly.

## 1.6.0 — 2026-08-23 (review pass v6: correctness, completeness, consistency, unambiguity)

### Correctness fixes

- DOC-11 `page_artifacts` was keyed by `payload_sha256` alone, but artifacts
  are not a pure function of payload bytes: `outlinks` hold identities
  resolved against the fetch's final URL [R-020], so byte-identical payloads
  fetched under different final URLs (mirrors) would collide on a single
  artifacts row with wrong outlinks for one of them — and R-157's "same
  input bytes ⇒ byte-identical artifact JSON" was false as stated. PK is now
  `(payload_sha256, final_url_identity)`; R-157 qualified ("same input bytes
  and same resolution base"); the §6 retention reference rule split per
  store (artifacts keyed on the pair, blobs on the hash); AC-042 extended
  with the mirror case.
- R-211 wake sources omitted enqueue events: discovery/ingestion [C1] and
  operator actions (seed injection, DEAD reset) run concurrently with fetch
  workers, so with an empty frontier the loop could sleep forever past a new
  candidate. The loop is now woken immediately by any event that creates or
  re-due-s a candidate.
- DOC-12 §3 pseudocode listed gates "[FR-011] a–d" with no evaluation point
  for gate (e) (registrable-domain cap), although FR-005(2)/FR-011(e) require
  a dispatch-time exclusion. The loop now evaluates (e) on the selected
  candidate and re-picks without waiting (exclusion, not deferral); noted
  that the re-pick terminates because ST-190 is terminal.
- Adversarial `Crawl-delay` + redirect hops: R-102 honors Crawl-delay
  unclamped and R-131 made each hop wait in-line for the target Host's
  window while holding a fetch worker and the source Host's concurrency
  slot — a hostile robots.txt (`Crawl-delay` ≈ years) could pin global
  workers/slots indefinitely, violating G-4. New error class ERR-018
  `HOP_RATE_LIMITED`: a hop whose target window delays the send by more than
  [CFG-035] aborts the chain RETRYABLE (source → ST-150), with the target
  window's opening as an unclamped floor on `next_attempt_mono`; the retry
  budget bounds pathological hosts to DEAD. New row in DOC-09 §4; DOC-13 §3
  floor comment generalized; new AC-055.
- DOC-06 §5 filter 3 off-by-one: the `>` comparison with existing records
  (the candidate cannot be counted — filters run before insertion [R-040])
  admitted [CFG-030]+1 URLs per path shape, off by one from the parameter's
  meaning, and the counting basis was unstated. Now: existing records ≥
  [CFG-030] ⇒ exclude, so at most [CFG-030] URLs share a shape; CFG-030
  given range >0.
- R-110: the politeness window is advanced at dispatch [T-1], but nothing
  required the request to be sent promptly afterwards, so on-wire
  start-to-start spacing could undercut EffectiveDelay by the
  dispatch-to-send latency. R-110 now requires the request to be sent
  immediately after [T-1] commits (the only permitted pre-send step is the
  R-054 re-check, which sends nothing when it fires).

### Completeness additions

- New R-104 (stale-while-revalidate): on robots cache TTL expiry with a
  cached entry, the verdict during the in-flight revalidation fetch was
  undefined. Previous rules remain authoritative while revalidating; a
  terminal 2xx/4xx response replaces them atomically; a failure defers the
  Host per §2.3. New AC-019.
- New R-112: which outcomes increment `hosts.consecutive_failures` was
  undefined (only "5xx increments" [DOC-09 §4] and "success resets" [R-231]
  were stated). Now: exactly the retryable page-fetch failures against the
  Host (ERR-001 excl. permanent NXDOMAIN, ERR-002, ERR-003, ERR-005, ERR-006,
  ERR-012, ERR-013); ERR-010 and PERMANENT outcomes never increment; robots
  exchanges never modify it (deferral streak instead). R-231 aligned
  ("successful page fetch").
- robots.txt exchange side effects fully specified [DOC-08 §2.2]: no
  fetch_events rows (visibility via robots_queries_total [DOC-15 §1]); no
  effect on consecutive_failures or pages_crawled.
- New R-406 (security): §5 said the operator API requires "no network
  exposure by default", yet nothing prevented exposing the mutating actions
  (seed injection, DEAD reset, drain) via a public [CFG-034] listener.
  Binding a non-loopback address now MUST disable the mutating actions
  (read-only /healthz and /metrics only) and MUST log a startup WARN. New
  AC-054.
- Run lifecycle defined [DOC-03 §Deployment], [DOC-11 §1]: one Run per
  process start (after config validation), closed at graceful shutdown,
  crashed Runs finalized by the next startup; the glossary's Crawl Session
  term (previously defined but used nowhere) is now the anchor for the
  span of Runs across restarts.
- DOC-11 §5 index set extended with (state, priority DESC, url_identity
  ASC) for dispatch selection [FR-010], [NFR-002] and (registrable_domain,
  state) for the cap gates [FR-005], [FR-011(e)] — the single-column indexes
  could not serve the selection order or the cap-count queries.
- Configuration gaps [DOC-14]: CFG-006 and CFG-030 lacked ranges (now >0
  each, per V-1); CFG-033 gains "0 = keep forever" (matching CFG-027/CFG-043
  zero conventions); CFG-039 entry format defined (absolute http(s) URLs;
  scheme+host+path participate, query/fragment ignored).
- ERR-009's recording location defined: parse failures live on the `pages`
  row (parse_ok=false, reason = ERR-009 + detail) and never appear as a
  FetchResult error_class — parsing is post-fetch [DOC-10 §4].
- R-400.3: "sorted ascending" pinned to a fixed total order (IPv4 before
  IPv6, numeric within family) so replay determinism [NFR-006] has an
  implementable sort key.

### Consistency fixes

- `hosts.scheme_host_port` renamed `host_key`, matching `urls.host_key` and
  the glossary Host triple (same column-name drift class as the `due_at_mono`
  fix in v1.5.0).
- FR-045: "nofollow MUST suppress link extraction" reworded to "suppress
  ingestion of anchor-derived candidates" — outlinks are still recorded
  (flagged) per DOC-10 §3, which the old phrasing contradicted.

### Versioning

- KB version 1.5.0 → 1.6.0; all touched documents bumped accordingly.

## 1.5.0 — 2026-08-23 (review pass v5: correctness, completeness, consistency)

### Correctness fixes

- FR-005 / FR-011: the per-Registrable-Domain cap [CFG-006] was enforced only
  at enqueue time — a TOCTOU race. Discoveries routinely enqueue before any
  fetch completes, so a domain could fetch far beyond [CFG-006] while the
  success counter was still zero (AC-006's "exactly two page successes" could
  be violated by concurrent dispatch). Caps are now enforced at two points:
  enqueue time (early exclusion) and dispatch time as a new authoritative
  gate [FR-011(e)], with `successes(D)` ({ST-130, ST-140}) and `inflight(D)`
  ({ST-110, ST-120}) defined precisely. Redirect final-target upserts [R-062]
  are exempt from [CFG-005] and count toward the target domain from
  completion (bounded overshoot documented).
- FR-010 vs DOC-12 §1 vs §3 vs DOC-03 C2: the Frontier ordering key
  `(due_at_mono ASC, priority DESC, …)` contradicted the dispatch rule
  "pick the highest-priority candidate" whenever two candidates with
  different due times and priorities were simultaneously due. Unified:
  `due_at_mono` determines candidacy only; selection among due candidates is
  `(priority DESC, url_identity ASC)`. Also fixes the `due_at_monotonic` vs
  `due_at_mono` column-name drift in DOC-03 C2.
- DOC-13 §3: `max(delay, Retry-After)` followed by `min(delay, CFG-035)`
  clamped `Retry-After` values larger than [CFG-035], contradicting
  "honor Retry-After" (the host-level rule [R-111] applies it unclamped).
  The clamp now applies to the computed backoff only; Retry-After is applied
  last, as a never-clamped floor.
- DOC-07 §2: the ST-110→ST-190 edge label included "UNKNOWN-timeout", yet
  R-103 states in-flight records are never affected by the deferral
  threshold — and a send-time robots refresh to UNKNOWN (Host deferred) had
  no legal handling at all: sending would violate "no page fetches while
  deferred" [DOC-08 §2.3], permanent exclusion was premature, and no
  ST-110→ST-100 transition existed. New R-054: DISALLOW ⇒ ST-110→ST-190;
  UNKNOWN ⇒ the dispatch is compensated (state → ST-100, attempts rolled
  back because no request was sent, slot released) and the record is
  reconsidered after deferral expiry [R-103]. R-051's release-target list and
  R-053 updated; the politeness advance is never rolled back.
- DOC-13 §4 / DOC-16 §5 vs DOC-07 §2: the operator reset of a DEAD URL to
  ST-100 was required but absent from the state machine, which declares every
  other transition a defect. The ST-180→ST-100 transition is now explicit
  (audited; attempts := 0; last error class cleared).
- FR-003: "create or refresh a URL Record and set state ST-100" forced
  already-terminal records back to ST-100 on rediscovery, contradicting
  FR-051/INV-5. Creation is now scoped to new identities.
- DOC-11 §6: DEAD/EXCLUDED deletion keyed on `updated_at` while FR-051/INV-5
  update `last_seen_at` on rediscovery — the two columns could diverge and
  the intended semantics were ambiguous. Deletion now keys on `last_seen_at`
  when set, else `updated_at`, and the deleted-then-recreated re-evaluation
  loop is documented as bounded and distinct from forbidden automated
  re-activation [DOC-13 §4].

### Completeness additions

- New CFG-043 `url_record_retention_days` (default 180): parameterizes the
  previously hardcoded 180-day DEAD/EXCLUDED retention [DOC-11 §6], per
  DOC-14's every-tunable-has-a-CFG-id rule.
- Redirect hops whose target Host is robots-deferred (verdict UNKNOWN) had no
  outcome: R-131 now aborts the chain as RETRYABLE/ERR-010 (source → ST-150,
  next attempt re-runs the chain), so no concurrency slot is held for a
  [CFG-040] deferral; ERR-010's note updated accordingly. New AC-017.
- Redirect responses for the robots.txt request itself were undefined
  ("same machinery as pages" would recurse robots-for-robots): followed per
  [DOC-09 §3] hop rules with SSRF/politeness/caps; scope and robots gates do
  not apply recursively; verdict from the final response [DOC-08 §2.2].
- R-144: the 304-after-retention refetch is defined mechanically — modeled
  like a redirect hop (politeness/caps respected), completing the same fetch
  attempt (no extra `attempts` increment).
- FR-051: rediscovery of records in non-success states now specified —
  `last_seen_at` only, state unchanged; DEAD/EXCLUDED re-activation stays
  operator-only [DOC-13 §4].
- DOC-06 §5 filter 3: "shape count" made precise — shape = normalized path
  with each maximal digit run replaced by one `N` (query excluded); count =
  URL Records on the Host sharing the shape (any state, per [R-042]).
- DOC-12 §2: multiple matching manual-boost prefixes — the largest single
  boost applies, never summed.
- `hosts.pages_crawled` defined (lifetime successful page fetches,
  SUCCESS|UNCHANGED; priority input) — referenced by the priority formula but
  never defined before.
- New AC-016 (send-time robots re-check), AC-018 (priority-first selection),
  AC-053 (operator DEAD reset — previously untested operator-API behavior);
  AC-006 extended to the dispatch-time cap gate; AC-024 extended to the
  unclamped-Retry-After case.

### Consistency fixes

- DOC-10 §3: `outlinks` includes all extracted anchor candidates — including
  `nofollow` ones, flagged — regardless of ingestion [R-155]; the `nofollow`
  flag in the tuple is now meaningful, and page-level `nofollow` pages
  [FR-045] store a fully flagged list while ingesting none.
- R-102 reworded: Crawl-delay honored exactly as received (the "> 60 s"
  phrasing implied smaller values might not be).
- R-402: "the host is flagged suspicious" → the referring URL's Host (the
  host that served the malicious redirect).
- Glossary "Page": the identity formula `(final URL identity, SHA-256)` —
  clashing with the actual `pages` primary key `(url_identity, run_id)` —
  replaced by a definition aligned to the storage model.
- DOC-07 §2 edge labels aligned with the rules that trigger them:
  creation→ST-190 gains enqueue-time `CAP_REACHED` [FR-005]; ST-100→ST-190
  gains the dispatch-time cap gate [FR-011(e)]; ST-140→ST-100 gains
  rediscovery refresh [FR-051].

### Versioning

- KB version 1.4.0 → 1.5.0; all touched documents bumped accordingly.

## 1.4.0 — 2026-08-23 (review pass v4: correctness, completeness, consistency)

### Correctness fixes

- DOC-00 glossary: the Processing-stages table cited the wrong documents for
  three terms — Scheduling → [DOC-13] (errors), Fetching → [DOC-10] (parsing),
  Parsing → [DOC-11] (storage). Corrected to [DOC-12], [DOC-09], [DOC-10].
- DEC-011 cited [DOC-15] (observability) for the configuration document;
  corrected to [DOC-14]. (Same off-by-one defect class as the glossary fixes.)
- R-030: still used the ad-hoc outcome label `REDIRECT_OUT_OF_SCOPE` — not
  a FetchResult enum member (same defect class the v1.3.0 pass fixed for
  FR-023/AC-020). Now outcome PERMANENT, error_class ERR-015.
- DOC-06 §2 step 1 (normalization): "percent-decode then re-encode using the
  unreserved set" would re-encode literal reserved delimiters (`/`, `=`, `&`)
  and corrupt every URL if implemented as written. Rewritten to RFC 3986
  §6.2.2 semantics: uppercase `%XX` hex; decode only unreserved octets; keep
  encoded reserved octets (e.g. `%2F`) encoded and literal delimiters as-is.
  Unparseable URLs are discarded at discovery / abort as seeds [FR-002].
- DOC-11 §6 retention: the stated deletion order (artifacts → pages → blobs)
  contradicted the rule that artifacts/blobs may be deleted only when no
  pages row references them. Correct order: pages rows → artifacts/blobs left
  unreferenced by those deletions → url records, each step committing before
  the next.
- FR-051 vs R-201: rediscovery refresh said "priority unchanged" while R-201
  requires recomputation on every ST-140→ST-100 transition. FR-051 now
  specifies attempts := 0, due_at_mono := now, priority recomputed.
- R-231 said `attempts` resets "on terminal success (ST-140)", R-052 said on
  the ST-140→ST-100 transition. Unified: the reset happens when the record
  leaves ST-140 for ST-100 (recrawl or rediscovery refresh).
- DOC-12 §4: recrawl jitter was seeded by hash(url_identity, run_id) — run_id
  changes across runs/restarts, breaking replay determinism [NFR-006],
  [AC-033]. Seed is now hash(url_identity, config_hash). Per-URL retry jitter
  (DOC-13 §3) is now explicitly seeded by hash(url_identity, attempt).
- R-103 said "every non-terminal URL Record" but enumerated only ST-100/ST-150
  (ST-130 is also non-terminal yet already fetched). Reworded to "gated states
  (ST-100 or ST-150)" with in-flight/ST-130 behavior stated.
- FR-010 cited [DOC-12 §2] for the ordering key; the key (heap + tie-breaks)
  is defined in [DOC-12 §1].
- FR-030 "first fetch to a Host in the current Run" implied per-run robots
  refetching; reworded to the persisted `robots_state = INITIAL` condition.

### Completeness additions

- Redirect hops blocked by the target Host's robots gate were required by
  FR-021/R-131 but had no outcome classification. New error class ERR-017
  `REDIRECT_ROBOTS_DISALLOWED` (permanent; source → ST-180, target never
  fetched, hop recorded in redirect_chain), mirroring the SSRF [R-402] and
  out-of-scope [ERR-015] treatments; new AC-015.
- 3xx responses with a missing/unparsable `Location` were unclassifiable;
  now PERMANENT/ERR-011 [DOC-09 §4]; new AC-028.
- CFG-028=false semantics were undefined for allowed non-HTML types (no
  stored payload ⇒ no pages row ⇒ no terminal state). ERR-008 now keys on the
  effective allowed list ([R-143] list ∩ [CFG-028]); FR-041/R-143 aligned;
  missing Content-Type defaults to `application/octet-stream`; new AC-029.
- Schema gaps closed in DOC-11 §1: `urls.host_key` and
  `urls.registrable_domain` (FR-005's per-domain cap, R-103 host queries, and
  trap path-shape rebuild had no indexed columns), `urls.consecutive_unchanged`
  (DOC-12 §4's 304-doubling had no persisted state), and
  `hosts.robots_deferred_since_mono` (R-103's "continuously deferred ≥ CFG-040"
  had no anchor, especially across restarts). New R-042 defines the
  deterministic startup rebuild of path-shape counters from `host_key`.
- R-211's sleep condition omitted ST-150 backoff expiries and robots-deferral
  expiries — the scheduler could sleep past both. Wake sources enumerated.
- New ST-100 records are now explicitly due at enqueue time (DOC-12 §1).
- R-111: Retry-After parsing defined (delta-seconds or HTTP-date per RFC
  9110; unparseable values ignored).
- DOC-08 §2.3: robots.txt bodies processed only up to the first 500 KiB
  (RFC 9309 recommended cap).
- DOC-09 §1: TLS 1.2 minimum with mandatory fail-closed verification;
  HTTP/2 server push MUST be disabled when negotiated; no request headers
  beyond those specified (fingerprint surface).
- R-400.3: deterministic address selection (sorted ascending) when several
  validated IPs exist — closes a replay-determinism hole.
- R-145 (DOC-09 §6): timings/payload fields of a FetchResult are defined for
  redirect chains (total spans the chain incl. politeness waits; per-phase
  timers describe the final hop).
- DOC-13 §4: operator reset of a DEAD URL now specifies attempts := 0 and
  clearing the last error class.
- DOC-10 §3: new `meta_refresh` artifact (the §2 table promised
  "discovery + metadata" but no artifact field existed); images row no longer
  implies a nonexistent discovery-type column.
- New AC-006 covers [CFG-006] per-Registrable-Domain capping and the
  ST-190-audit-records-don't-consume-[CFG-005] rule [FR-005].

### Consistency fixes

- DOC-16 §5 operator API and DOC-07 §2: the two robots-exclusion transitions
  (ST-100→ST-190 vs ST-110→ST-190) are now disambiguated — the gate is
  evaluated at dispatch selection (ST-100); the ST-110 path covers verdict
  changes between [T-1] and request send; slots release exactly once either
  way [R-051].
- DOC-03 C5: FetchResult field list aligned to the canonical contract
  [DOC-09 §6] (was an ad-hoc shorthand: status/payload_ref/error).
- NFR-004: disk bound restated correctly for recrawl (per-generation
  retention), not the single-generation formula.
- DOC-15 §1: `robots_queries_total` label `deferred` → `unknown` (matches the
  C4 verdict enum and R-240's closed-enum rule); `content_type_class` enum
  defined; new `suspicious_hosts` gauge (R-402 called for an
  operator-visible signal that no metric exposed).
- R-100: unverifiable citation "RFC 9309 §5.2" reduced to RFC 9309.
- PREFIX_LIST scope match defined precisely (segment-boundary path prefix,
  query ignored) [DOC-06 §4].
- R-031 no longer implies a nonexistent scope-verdict cache column; the
  verdict is state-reflected and evaluated once per identity at ingestion.

### Versioning

- KB version 1.3.0 → 1.4.0; all touched documents bumped accordingly.

## 1.3.0 — 2026-08-23 (review pass v3: consistency, completeness, correctness)

### Correctness fixes

- R-051: host/global concurrency slots are now released on the first
  transition out of {ST-110, ST-120}. Previously they were released only on
  transitions out of ST-120, but ST-110→ST-190 (robots exclusion at gate
  time [FR-031]) is a legal transition — each occurrence leaked one slot and
  could permanently starve a Host's concurrency caps until restart.
- R-103: robots-unknown exclusion now applies to every non-terminal record on
  the deferred Host (ST-100 or ST-150), and the transition ST-150→ST-190 was
  added to the state machine. Previously ST-150 records were never covered:
  with no legal transition to exclude them, they waited in ST-150 forever on
  a permanently deferred Host.
- FR-023 / AC-020: outcome labels aligned to the FetchResult enum [DOC-09 §6];
  oversized payloads and redirect-cap exhaustion are recorded as
  outcome=PERMANENT with error_class ERR-007 / ERR-011 (the ad-hoc labels
  `PAYLOAD_TOO_LARGE` / "REDIRECT cap error" were not enum members).
- DOC-13 §2: `attempts` is now described as the count of started attempts,
  incremented in the dispatch transaction [T-1] — "completed" contradicted
  R-053/FR-012 crash counting.
- DOC-13 §3: jitter term `U(−j, +j)` had an undefined `j`; corrected to
  `U(−1, +1) × CFG-024`.
- DOC-09 §2: DNS-resolution timeouts were not mapped to any error class;
  now ERR-001 (retryable — NXDOMAIN remains permanent).
- DOC-16 §3: the 64 KiB header-size cap had no outcome classification; new
  error class ERR-016 `HEADER_TOO_LARGE` (permanent).
- R-402: "ST-180/`SSRF_BLOCKED`" cited an undefined reason code (reason codes
  exist only for ST-190); now ST-180 with error_class ERR-004.
- Section-reference corrections (all verified against actual section
  numbering): DEC-007 → [DOC-08 §2]; DOC-03 C3 → [DOC-08 §4]; FR-013 →
  [DOC-12 §2]; FR-025 → [DOC-11 §1]; FR-032 → [DOC-08 §4]; R-220 →
  [DOC-08 §3]. DOC-07's sections are now numbered and the stale [DOC-07 §5]
  references in DOC-03 point to §4 (R-060/R-062).
- FR-013 no longer lists "change frequency hints" as a priority input; the
  [DOC-12 §2] formula's actual inputs are depth, seed status, host failure
  history, prior host success, and manual boost.

### Completeness additions

- DOC-11 `hosts` schema: added `robots_rules` (persisted parsed robots rules —
  DOC-08 §2.4 required caching "verdict function inputs" but nothing stored
  them across restarts) and the `INITIAL` robots_state (pre-first-fetch).
- DOC-08 §2.2: the robots.txt fetch is now explicitly specified to advance
  `next_allowed_fetch_at` and hold one concurrency slot (previously only
  "obeys the host politeness window" — consumption was undefined), and
  "exempt from page caps" is qualified to page-success caps and Content Store
  storage only; transport safety caps [DOC-16 §3] still apply per security
  precedence [R-000].
- R-400's "configured deny-list" now has a real parameter: CFG-041
  `egress_deny_ips`. New CFG-042 `egress_allow_private_ranges` (default false,
  startup WARN, production-forbidden via new R-405) gives the acceptance-test
  fixture server a documented escape hatch from fail-closed SSRF blocking —
  previously every fixture-server-based AC (AC-010 et al.) was unsatisfiable
  because R-400 blocks loopback; DOC-17's preamble now states fixture configs
  set it, with AC-025 running with it false.
- FR-002 extended to reject userinfo-bearing seeds [R-002], matching AC-002
  (which tested a rule no FR specified).
- ST-150→ST-100 now specifies `due_at_mono := next_attempt_mono` (the mapping
  between the two scheduler columns was previously undefined).
- FR-005: the CFG-005 total now counts only non-EXCLUDED records — ST-190
  audit records (e.g., OUT_OF_SCOPE under CFG-038=true) no longer consume the
  crawl budget.
- DOC-12 §4: 304-based recrawl-interval doubling now specifies
  consecutive-304 semantics with reset on the next full 200 (was ambiguous
  between one and repeated doublings, with no reset rule).
- DOC-10 §3: `truncated` artifact flag added (DOC-16 §3 required flagging
  truncation but no such field existed in the artifact schema).
- CFG-016 given a lower bound (1) — its range was one-sided.

### Consistency fixes

- DOC-16 §5: `[health_listen_addr]` → [CFG-034] (DOC-14 requires parameters
  be referenced by CFG id only).
- DOC-11 §2: "two-level fanout" → sharded on the first two hex chars (the
  actual layout `blobs/ab/abcdef…` has a single fanout level).
- R-401: redundant "(plus explicit scheme defaults)" removed; ports are
  restricted to the scheme defaults (80/443).
- ERR-008 note aligned with ST-180 semantics ("not fetched again this run"
  implied a per-run scope that does not exist; DEAD is permanent until
  operator reset [DOC-13 §4]).
- DOC-07's final section retitled "Recovery and redirect completion" to
  match its content (R-061/R-062 are not crash-recovery rules).

### Versioning

- KB version 1.2.0 → 1.3.0; all touched documents bumped accordingly.
## 1.2.0 — 2026-08-23 (review pass v2: correctness, completeness, consistency)

### Correctness fixes

- FR-012, T-1, R-053: `urls.attempts` is now incremented inside the dispatch
  transaction. Previously no rule incremented it, yet DOC-13 §5 crash
  classification and the ST-150→ST-180 budget check both depended on "attempt
  already counted at dispatch".
- DOC-11 §6 retention: blobs and page_artifacts rows are now deleted only when
  NO remaining `pages` row references their payload_sha256. The old age-only
  rule would have dangled references whenever dedup shared a hash across URLs.
  Deletion basis for DEAD/EXCLUDED records specified (`updated_at`); fetch_events
  retention now references [CFG-033] instead of a hardcoded 7 days.
- INV-1 vs redirects: INV-1 now exempts redirect-hop targets; new R-062 defines
  URL Record creation for a chain's final target (depth = source depth per
  DEC-009). Previously DEC-004 required such records but nothing created them,
  and INV-1 contradicted hop fetching outright.
- Error taxonomy: added ERR-014 HTTP_4XX_PERMANENT and ERR-015
  REDIRECT_OUT_OF_SCOPE; DOC-09 §4's ad-hoc ST-180 labels (`HTTP_4xx_AUTH`,
  `GONE`, `HTTP_4xx_OTHER`) and the unmapped `REDIRECT_OUT_OF_SCOPE` outcome now
  resolve to real error classes.
- State machine: ST-120→ST-130 now explicitly covers 304/UNCHANGED (it said
  "2xx" only); new R-144 defines the 304-after-retention cache-miss refetch.
- DOC-08 §4 / FR-012: harmonized the dispatch advance to
  `max(next_allowed_fetch_at, now) + EffectiveDelay` (FR-012 previously said
  "advance by", which drifts when now < next_allowed_fetch_at).

### Completeness additions

- R-103 + CFG-040: defined when ST-190/`ROBOTS_UNKNOWN_TIMEOUT` is actually
  applied (continuous robots deferral ≥ CFG-040) — the reason code existed but
  nothing triggered it; robots deferral backoff cap parameterized (was
  hardcoded 24 h).
- Schema gaps filled: `urls.is_seed` (priority formula input),
  `urls.last_seen_at` (FR-051/INV-5), `hosts.suspicious` (R-402) — all
  referenced elsewhere but missing from DOC-11 §1.
- R-131 extended: every redirect hop must respect the target host's politeness
  window and concurrency caps (previously unspecified).
- R-232: "retryable (once)" classes (ERR-003, ERR-013) given a defined
  effective budget of 2 total attempts.
- Priority term "+50 if same host has fresh successful history" replaced with
  the testable `host.pages_crawled > 0`.
- V-4 + AC-005: PREFIX_LIST mode aborts startup if any seed matches no prefix
  (previously such a config crawled nothing silently).
- New acceptance criteria AC-026 (cross-host redirect politeness + final-target
  record) and AC-027 (ROBOTS_UNKNOWN_TIMEOUT); AC-043 extended to the shared-
  blob retention case.

### Consistency fixes

- Rule ID blocks made collision-free and documented: DOC-11 storage rules
  renumbered R-300..R-302 → R-500..R-502 (all references updated); DOC-16 note
  corrected to "owns R-3xx and R-4xx"; README example updated.
- CFG-003b renamed CFG-039 (ID convention is CFG-nnn); reference in DOC-06 §4
  updated.
- DOC-03 C7 table list aligned with the actual DOC-11 schema (dropped phantom
  `kv_config_state`; added `pages`, `page_artifacts`).
- Removed remaining self-dialogue prose (same defect class as DEC-009 in
  v1.1.0): DOC-06 normalization steps 8–9, DOC-09 §2 total-transfer cell.
- R-041 qualified to except OUT_OF_SCOPE drops under [CFG-038]=false (was
  inconsistent with FR-004); garbled recovery parenthetical in DOC-03 fixed;
  NFR-015 editorial aside rewritten normatively; fetch_events "bounded ring
  buffer" wording aligned with its time-based retention.

### Versioning

- KB version 1.1.0 → 1.2.0; all touched documents bumped accordingly.

## 1.1.0 — 2026-08-23 (review pass: consistency, completeness, correctness)

No behavioral semantics changed except where noted; this revision resolves
defects found in a full cross-reference audit of v1.0.0.

### Correctness fixes

- FR-050, R-052: fixed wrong section references ([DOC-12 §6] → §4 and §2
  respectively).
- DOC-08 §4: per-host backoff formula produced a nonzero baseline (CFG-022)
  with zero consecutive failures and used an exponent inconsistent with the
  per-URL retry formula; corrected to `0` when `consecutive_failures = 0`,
  else `CFG-022 × CFG-023^(failures − 1)`.
- DOC-07 state machine: added missing legal transitions `(creation) → ST-190`
  (filter failures [FR-004]) and `ST-100 → ST-190` (robots DISALLOW at gate
  time [FR-031]); previously required by other documents while the machine
  claimed "no other transitions exist".
- R-060: crash recovery now explicitly rebuilds `hosts.inflight` from URL
  record states after the ST-110/ST-120 reset (required for INV-3/NFR-011;
  previously the persisted value could be stale after kill -9).
- DEC-009: removed leftover editorial self-dialogue ("? — NO:") from the
  binding decision text.
- R-231: rewrote garbled sentence about what resets host vs URL counters.

### Completeness additions

- CFG-035 `max_backoff_delay_ms`: parameterizes the hardcoded backoff caps
  (600000 ms in DOC-08, "1 hour" in DOC-13) into one tunable.
- CFG-036 `dns_timeout_ms`: the DNS timeout was a non-conformance constant
  ("10 s fixed"), violating DOC-14's "every parameter referenced by CFG id"
  rule; also now covered by FR-022.
- CFG-037 `url_blocklist`: the `BLOCKLIST` exclusion reason existed in DOC-07
  but no mechanism defined it; added as trap-filter step 1 in DOC-06 §5.
- CFG-038 `log_exclusions`: referenced by FR-004 but never defined; added
  (default true).
- DOC-11: `fetch_events.redirect_chain` column added so the redirect chain is
  actually persisted [R-133]; documented same-run refetch UPSERT semantics for
  the `pages` primary key.
- Seed depth defined as 0 (was implied but never stated).

### Consistency fixes

- DOC-16 §2: renumbered colliding rule IDs R-300b..R-303b → R-400..R-403
  (they clashed with DOC-11's R-300..R-302); README now states that R-nnn IDs
  are globally unique per ID block.
- CFG-006 renamed `max_pages_per_host` → `max_pages_per_registrable_domain`
  to match its actual scope ([FR-005], glossary Host vs Registrable Domain).
- DOC-12 §2 priority formula: sloppy `[CFG: manual per-prefix boosts]`
  replaced with a proper [CFG-031] reference.
- DOC-12 §4: removed confusing stray line ("override = Retry-After never
  applies here") and made CFG-025=0 ⇒ never-recrawl explicit in the formula.
- R-002: clarified userinfo URL handling (discarded at discovery with no
  record, like R-001).

### Versioning

- KB version 1.0.0 → 1.1.0; all touched documents bumped accordingly.

## 1.0.0 — 2026-02-23

Initial approved draft.
