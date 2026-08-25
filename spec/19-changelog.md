---
id: DOC-18
title: Changelog
version: 1.20.0
---

# Changelog

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
