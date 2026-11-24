# Spec Verification Test

This test verifies the DOC-09 §4 response classification table behaviors as mandated by acceptance criteria AC-106 and AC-136.

## 429 and 5xx Rows Budget Conditioning

The [DOC-09 §4] response-classification table's retryable rows MUST state the landing conditioned on the retry budget exactly like the [R-131] ERR-018 row and the normative [DOC-13 §3] procedure:

- A `429` honors `Retry-After` [R-111] → ST-150 when `attempts < [CFG-020]`, but land it in ST-180 with `urls.last_error_class` recorded and `urls.next_attempt_mono=NULL` when the retry budget is exhausted [FR-052], [AC-024], [AC-085].
- A `5xx` → ST-150 when `attempts < [CFG-020]`, or ST-180 with `urls.last_error_class` recorded and `urls.next_attempt_mono=NULL` when the retry budget is exhausted [DOC-13 §3].

### Verification

The following rows in the DOC-09 §4 table have been verified to have correct budget conditioning:

| Status | Class | Action |
|--------|-------|--------|
| 429 | retryable | honor Retry-After [R-111] → ST-150 when `attempts < CFG-020`, or ST-180 with `urls.last_error_class` recorded and `urls.next_attempt_mono=NULL` when the retry budget is exhausted [DOC-13 §3], [AC-024]; increments host `consecutive_failures` [R-112] on either landing |
| 5xx | retryable | → ST-150 when `attempts < CFG-020`, or ST-180 with `urls.last_error_class` recorded and `urls.next_attempt_mono=NULL` when the retry budget is exhausted [DOC-13 §3]; increments host `consecutive_failures` [R-112] on either landing |

**AC-106 compliance check:** Both 429 and 5xx rows state the landing conditioned on the retry budget: ST-150 when `attempts < CFG-020`, ST-180 with `urls.last_error_class` recorded and `urls.next_attempt_mono=NULL` when the budget is exhausted. The 429 row additionally specifies it honors `Retry-After` [R-111] unclamped within budget. An implementation omitting `urls.next_attempt_mono=NULL` would re-arm a backoff timer on budget-exhausted records, stranding records in ST-150 with `attempts >= [CFG-020]` and violating the "non-NULL `next_attempt_mono` only in ST-150" field-hygiene invariant [AC-085].

**AC-136 compliance check:** The 429 row states the landing conditioned on the retry budget and the `hosts.consecutive_failures` increment, exactly like the 5xx row: a `429` honors 'Retry-After' [R-111] → ST-150 when `attempts < CFG-020`, or ST-180 with `urls.last_error_class` recorded and 'hosts.consecutive_failures' incremented [R-112] when the retry budget is exhausted [FR-052], [AC-024]. An implementation following the former 429 row clause 'honor Retry-After [R-111] → ST-150' without the `consecutive_failures` increment would fail to increment the host counter on budget-exhausted 429 landings, diverging from the [R-112] invariant and contradicting the same table's 5xx row which explicitly increments it.

Both rows correctly specify `urls.next_attempt_mono=NULL` when the retry budget is exhausted, aligning them with the `ERR-018` row and [R-131] step-3 narration.

## 3xx Hop Robots Rows

The [DOC-09 §4] response-classification table's hop-blocked-by-robots row MUST split the DISALLOW case from the UNKNOWN/deferred case:

- **DISALLOW**: chain terminates at that hop with outcome=PERMANENT, `error_class='ERR-017'`, source → ST-180
- **UNKNOWN (Host deferred)**: chain aborts with outcome=RETRYABLE, `error_class='ERR-010'`; with `attempts < [CFG-020]` the source lands in ST-150 with a backoff `next_attempt_mono` computed per [DOC-13 §3]; with the budget exhausted it lands in ST-180 with `urls.last_error_class='ERR-010'` and `urls.next_attempt_mono=NULL` [AC-112], [AC-115], [AC-118]

### Verification

Row 14 in the DOC-09 §4 table has been verified to have correct budget conditioning for the UNKNOWN/deferred case, specifying `next_attempt_mono=NULL` when the retry budget is exhausted.

**AC-115 compliance check:** The 3xx UNKNOWN row MUST NOT blanket-land the source in ST-150; it MUST condition the landing on the retry budget exactly as the normative [DOC-13 §3] procedure does — ST-150 with a backoff `next_attempt_mono` when `attempts < [CFG-020]`, or ST-180 with `urls.last_error_class='ERR-010'` and `next_attempt_mono=NULL` when the budget is exhausted [AC-024], [AC-085]. The row correctly specifies both landings are budget-conditioned.

**AC-118 compliance check:** The acceptance tests for the URL-level `ERR-010` abort MUST NOT blanket-land the source in ST-150; they MUST condition the landing on the retry budget exactly as the normative [DOC-13 §3] procedure does — ST-150 with a backoff `next_attempt_mono` when `attempts < [CFG-020]`, or ST-180 with `urls.last_error_class='ERR-010'` and `next_attempt_mono=NULL` when the budget is exhausted [AC-024], [AC-085]. The row correctly specifies both landings are budget-conditioned.

## 3xx Hop Politeness Delay Row

The [DOC-09 §4] response-classification table's row for hops delayed beyond [CFG-035] by the target Host's politeness window MUST state the landing conditioned on the retry budget exactly as the normative [DOC-13 §3] procedure does:

- **ST-150** when `attempts < [CFG-020]`: `next_attempt_mono` floored at the target Host's window opening
- **ST-180** with `urls.last_error_class='ERR-018'` and `urls.next_attempt_mono=NULL` when the retry budget is exhausted (the floor never lands on a DEAD record) [DOC-13 §3], [AC-085]
- no fetch worker is held for the over-threshold wait [AC-055]

### Verification

Row 17 in the DOC-09 §4 table has been verified to have correct budget conditioning, specifying `next_attempt_mono=NULL` when the retry budget is exhausted, aligning it with the `ERR-018` row and [R-131] step-3 narration.

**AC-085 compliance check:** The ERR-018 hop-abort politeness floor [R-131] applies only to a RETRYABLE landing in ST-150, never to a budget-exhausted landing in ST-180, so no DEAD record carries a backoff timer [DOC-12 §1], [DOC-11 §1]. An implementation omitting `urls.next_attempt_mono=NULL` on the budget-exhausted path would re-arm a backoff timer on a DEAD record, violating the "non-NULL `next_attempt_mono` only in ST-150" field-hygiene invariant [AC-085].

Both rows correctly specify `urls.next_attempt_mono=NULL` when the retry budget is exhausted, aligning them with the `ERR-018` row and [R-131] step-3 narration.

**AC-024 compliance check (429 subset):** With [CFG-020]=5, five consecutive `429` responses drive `attempts` to 5; the fifth is the budget-exhausting outcome, which MUST produce state ST-180, `urls.last_error_class='ERR-005'`, `fetch_events.http_status=429`, and no backoff timer — not ST-150 with a re-armed timer. A 429 `Retry-After` honored unclamped within budget [DOC-13 §3] does not prevent the ST-180 landing when `attempts ≥ [CFG-020]`. An implementation re-arming a backoff timer on budget-exhausted 429 landings would strand records in ST-150 whose `attempts` equals [CFG-020] (the ST-150→ST-100 promotion requires `attempts < [CFG-020]` [DOC-12 §1]) and violating the "non-NULL `next_attempt_mono` only in ST-150" field-hygiene invariant [AC-085], even though it honors `Retry-After` and the host-counter increment correctly within budget.

## 429 Retry-After and Budget Exhaustion Interaction

With [CFG-020]=3 and a sequence of three consecutive `429` responses where each includes a `Retry-After` header:

- **First 429** (`attempts=1 < [CFG-020]`): The outcome lands in ST-150 with `urls.next_attempt_mono` set per the `Retry-After` value unclamped [R-111], [DOC-13 §3]. The host's `consecutive_failures` is incremented [R-112].
- **Second 429** (`attempts=2 < [CFG-020]`): The outcome lands in ST-150 with `urls.next_attempt_mono` updated per the new `Retry-After` value. The host's `consecutive_failures` is incremented again [R-112].
- **Third 429** (`attempts=3 ≥ [CFG-020]`): The budget is exhausted. The outcome lands in ST-180 with `urls.last_error_class='ERR-005'`, `urls.next_attempt_mono=NULL`, and `fetch_events.http_status=429`. The host's `consecutive_failures` is incremented one final time [R-112]. A `Retry-After` header present on this response is **not** honored (the budget-exhausted path does not apply Retry-After), consistent with [DOC-13 §3] and [AC-085]. The record is DEAD and will not be retried until an operator reset [DOC-13 §4] or redirect final-target upsert [R-062] re-admits it.

An implementation that re-arms a backoff timer on the budget-exhausted ST-180 landing, or that honors `Retry-After` on the budget-exhausted path, would violate the "non-NULL `next_attempt_mono` only in ST-150" field-hygiene invariant [AC-085] and the AC-024 assertion that three consecutive `429` outcomes reach DEAD at the third occurrence.

## 5xx Rows Budget Conditioning Deeper Comparison

The [DOC-09 §4] 5xx row states: `→ ST-150 when `attempts < CFG-020`, or ST-180 with `urls.last_error_class` recorded and `urls.next_attempt_mono=NULL` when the retry budget is exhausted [DOC-13 §3]; increments host `consecutive_failures` [R-112] on either landing`. Differs from the 429 row only in that it does not include `honor Retry-After [R-111]` since 5xx responses do not carry a `Retry-After` header per [R-111]. The budget conditioning, `next_attempt_mono=NULL` on exhaustion, and `consecutive_failures` increment on either landing are identical between the two rows, as verified by [AC-136].

## 3. DOC-13 §3 Retry Procedure: once_retried_classes gate precedence

The [DOC-13 §3] retry decision procedure has a critical gate: the "yes (once)" class check [R-232] is evaluated **first**, before the retry budget check [CFG-020]. This means:

- If a RETRYABLE outcome's error class is already in `urls.once_retried_classes`, the outcome is PERMANENT/ST-180 **regardless of budget** — even when `attempts < [CFG-020]`.
- If the class is not yet listed, it is appended to `urls.once_retried_classes` [T-2], and the normal budget/backoff path applies: ST-150 when `attempts < [CFG-020]`, ST-180 (DEAD) when budget exhausted.

This gate precedence is mandated by [R-232] and is tested by multiple acceptance criteria:

- **AC-093**: A RETRYABLE redirect final-target upsert [R-062] whose outcome class is already listed in the target's `once_retried_classes` lands the target in ST-180 (PERMANENT), not ST-150, regardless of [CFG-020].
- **AC-024**: Two consecutive RETRYABLE ERR-003 outcomes with [CFG-020]=5 reach DEAD at the second occurrence; after a success and recrawl promotion, the class is eligible for one retry again.
- **AC-087**: The [CFG-030] path-shape guard caps admissions, not the number of URL Records sharing a shape, because excluded ST-190 records are themselves created and counted.
- **AC-084**: A RETRYABLE redirect final-target upsert [R-062] onto a pre-existing non-ST-180 record sets the target's `last_error_class` to the outcome's class and leaves it non-NULL until the target's own next successful completion clears it [T-2].

### Verification

The DOC-13 §3 procedure flow is:

1. **Check "once" classes** (`ERR-003`, `ERR-013`):
   - If class ∈ `urls.once_retried_classes` → ST-180 (DEAD), outcome PERMANENT, `error_class := class`
   - Else: append class to `urls.once_retried_classes` [T-2]; fall through
2. **Check budget** (`attempts ≥ [CFG-020]`):
   - If budget exhausted → ST-180 (DEAD), `error_class` recorded
   - Else: compute backoff delay, `state → ST-150, next_attempt_mono = now + delay`
3. **ERR-018 floor** (only on ST-150 path):
   - `next_attempt_mono = max(next_attempt_mono, target_host.next_allowed_fetch_at)`

The "once" class check takes absolute precedence over the budget check. An implementation that checks the budget first and only then evaluates the "once" class would land second (and subsequent) `ERR-003`/`ERR-013` outcomes in ST-150 even when the class is already listed, violating [R-232], [R-062], and AC-093.

| Scenario | CFG-020 | `once_retried_classes` state | Outcome | State |
|---|---|---|---|---|
| First ERR-003 | 5 | empty → appends ERR-003 → fall through | RETRYABLE, append class | ST-150 with backoff |
| Second ERR-003 (same cycle) | 5 | [ERR-003] already listed | PERMANENT | ST-180 (DEAD) |
| Second ERR-003 after success+recrawl | 5 | [] (cleared by [R-052]) → appends ERR-003 → fall through | RETRYABLE, append class | ST-150 with backoff |

**AC-093 compliance check:** A RETRYABLE redirect final-target upsert [R-062] whose outcome class is already listed in the target's `once_retried_classes` MUST land the target in ST-180 (PERMANENT), not ST-150, regardless of [CFG-020]. An implementation landing RETRYABLE outcomes in ST-150 even when the class is already listed violates [R-232] and [R-062], and AC-052's partition assertion would fail because the `excluded`/`dropped` bucket assignment would diverge from `exclusions_total`.

**AC-024 compliance check:** Two consecutive RETRYABLE ERR-003 outcomes with [CFG-020]=5 drive `attempts` to 2; the second is the budget-exhausting outcome only if the "once" gate does NOT intervene. Per [R-232], the "once" gate takes precedence: the second ERR-003 lands ST-180 (DEAD) even though `attempts=2 < [CFG-020]=5`. After a success and recrawl promotion ([R-052] resets `attempts=0`, clears `once_retried_classes` [R-232]), the class is eligible for one retry again.

Both rows correctly specify that the `once_retried_classes` gate is evaluated first, before the budget check, aligning with [R-232] and [R-062].

## DOC-13 §3 Retry Procedure: once_retried_classes gate with [R-062] upsert path

The [R-062] redirect final-target upsert path applies the same [DOC-13 §3] retry procedure as ordinary retry landings, including the "once_retried_classes" gate precedence. This ensures consistent behavior whether the RETRYABLE outcome arises from a direct fetch or from a redirect chain's final target.

- **First occurrence** of a "once" class (ERR-003 / ERR-013): The class is appended to the target's `once_retried_classes` [DOC-11 §1, R-232], and the normal budget check applies: if `attempts < [CFG-020]` the target lands in ST-150 with `next_attempt_mono` mirroring the source's backoff; if `attempts ≥ [CFG-020]` the target lands in ST-180 (DEAD) with `next_attempt_mono=NULL`.
- **Second occurrence** of the same "once" class (within the same attempt cycle): The class is already listed in the target's `once_retried_classes`, so the "once" gate takes precedence: the outcome is PERMANENT, the target lands in ST-180 regardless of `attempts` relative to `[CFG-020]`, and `urls.last_error_class` is set to that class [R-062], [R-232].
- **After success + recrawl promotion** ([R-052]): The `once_retried_classes` list is cleared when `attempts` resets to 0, and the target is re-admitted to ST-100. A subsequent RETRYABLE "once" class outcome on first re-dispatch lands ST-150 (class appended, budget check applies), while a second occurrence would land ST-180 since the class is once again listed.

**AC-093 compliance verification:** A RETRYABLE redirect final-target upsert [R-062] whose outcome class is already listed in the target's `once_retried_classes` MUST land the target in ST-180 (PERMANENT), not ST-150, regardless of [CFG-020]. An implementation failing to apply the "once" gate on the upsert path would violate [R-232] and [R-062].

**AC-024 compliance verification:** Two consecutive RETRYABLE ERR-003 outcomes on a redirect chain's final target with [CFG-020]=5: the first outcome appends ERR-003 to the target's `once_retried_classes` and lands ST-150 (within budget); the second outcome finds the class already listed and lands ST-180 (DEAD) even though `attempts=2 < [CFG-020]=5`. After a success and recrawl promotion ([R-052] resets `attempts=0`, clears `once_retried_classes` [R-232]), the class is eligible for one retry again.

| Scenario | CFG-020 | `once_retried_classes` state | Outcome | State |
|---|---|---|---|---|
| First "once" class occurrence | 5 | empty → appends class → fall through | RETRYABLE, append class | ST-150 with backoff |
| Second "once" class occurrence (same cycle) | 5 | class already listed | PERMANENT | ST-180 (DEAD) |
| First "once" class after success+recrawl | 5 | [] (cleared by [R-052]) → appends class → fall through | RETRYABLE, append class | ST-150 with backoff |

Both rows correctly specify that the `once_retried_classes` gate is evaluated first, before the budget check, aligning with [R-232] and [R-062].

## DOC-00 Per-Host Concurrency Definition

The [DOC-00] controlled-vocabulary entry for **Per-Host Concurrency** MUST define the metered quantity as in-flight HTTP tasks (units) held against one Host — not individual HTTP requests — exactly as its sibling **Global Concurrency** entry already does. The per-Host unit is the task's single unit [R-051], transferred sequentially across redirect hops [R-131] (the source Host's unit is released when its response arrives) and plus one unit per in-flight robots.txt exchange [DOC-08 §2.2] [AC-011], [AC-035], [R-212].

### Verification

The [DOC-00] **Per-Host Concurrency** entry has been verified to have the correct task/unit basis:

- **Fixed entry:** "Max simultaneous in-flight HTTP tasks (units) held against one Host — a dispatched fetch task's single per-Host unit [R-051], which a redirect-hop chain moves sequentially from the source Host to each following Host (the source Host's unit is released when its response arrives [R-131], so a multi-hop task holds at most one Host unit at any time), plus one unit per in-flight robots.txt exchange [DOC-08 §2.2]; it counts tasks/units, not individual HTTP requests [AC-011], [AC-035], [R-212]. MUST be ≥1."

**AC-137 compliance check:** The entry MUST NOT read "Max simultaneous in-flight HTTP requests to one Host (initial requests, redirect hops [R-131], robots.txt exchanges)". That wording meters [CFG-009] in individual HTTP requests, contradicting [R-051]'s "a task never holds two Host units", [R-131]'s release-before-acquire step 1, [R-212]'s unit accounting and [AC-011]/[AC-035] ("per-host inflight counts concurrent tasks, not individual HTTP requests"). An implementation following the former wording meters [CFG-009] in requests: with [CFG-009]=1 and a fetch task following a 3-hop redirect chain whose three `Location` targets are all on the same Host H, it would see 3 concurrent requests against H, violating [R-051], [R-131], [R-212], [FR-011(a)] and INV-3, and would abort or serialize the chain — even though it handles single-request fetches and robots exchanges correctly. The fixed entry names the same components (redirect hops, robots exchanges) whose units are the task's single per-Host unit, not one request each, and cites [AC-011], [AC-035] and [R-212] as authority for the task/unit basis. This is the same residual-summary defect [AC-089], [AC-104]–[AC-136] corrected in [R-051], [R-131], [R-212] and [AC-011]/[AC-035], left in the [DOC-00] controlled-vocabulary entry every document is required to use "exactly".

## DOC-12 §4 Recrawl Formula: UNCHANGED (304) Outcome Coverage

The [DOC-12 §4] freshness/recrawl formula's trigger MUST cover the UNCHANGED outcome of a `304` with a usable stored payload [R-121], including the UNCHANGED branch of the [R-144] cache-miss refetch — the outcome the formula's own 304-multiplier clause ("interval doubles per consecutive 304, capped at 4 × base_interval", count persisted as `urls.consecutive_unchanged` [DOC-11 §1]) is defined for — while a PERMANENT outcome (e.g. the [R-144] double-304 cache miss [ERR-014]) installs no recrawl due time [AC-139].

### Verification

The [DOC-12 §4] trigger has been verified to cover the UNCHANGED outcome:

- **Fixed clause:** "On a successful completion — outcome SUCCESS (2xx) or UNCHANGED (a `304` with a usable stored payload [R-121], including the UNCHANGED branch of the [R-144] cache-miss refetch) — a page's next recrawl time:"

**AC-139 compliance check:** (a) With [CFG-025]>0, an UNCHANGED `304` completion MUST install `due_at_mono = fetch_complete_mono + interval` with the 304 multiplier (doubling per consecutive UNCHANGED, capped at 4 × [CFG-025], count in `urls.consecutive_unchanged` [DOC-11 §1], deterministic jitter [NFR-006]) — the ST-140→ST-100 due-time promotion [DOC-12 §1] fires at that due time and the URL is refetched [FR-050], [AC-072], [AC-033]. (b) A PERMANENT outcome ([R-144] double-304 cache miss [ERR-014], [AC-059], [AC-101]) lands ST-180 with `urls.last_error_class='ERR-014'` and `urls.due_at_mono` NULL: the §4 "Otherwise" branch computes `due_at_mono` for successful completions only. An implementation following the former "On success" trigger literally skips the UNCHANGED completion of (a): no due time is installed, the due-time promotion has no trigger, the URL is never recrawled, and `urls.consecutive_unchanged` grows without ever driving the 304 multiplier — a deterministic schedule divergence [NFR-006], [AC-033] — even though it handles plain 2xx recrawl and the [R-144] PERMANENT case of (b) correctly.

## AC-094: is_seed Idempotency

An idempotent `is_seed` update on a pre-existing URL Record — config-seed re-ingestion on restart [NFR-012] or a runtime seed injection [FR-006] for an identity that already has a record — increments `urls_discovered_total{duplicate}` exactly once and never `urls_discovered_total{ingested}` [DOC-15 §1], [AC-032], [AC-058], [AC-090]. Creating a genuinely new URL Record (including a redirect final-target upsert that creates its target [R-062]) still increments `ingested`, and a re-ingestion that finds no pre-existing record is `ingested`, not `duplicate`. Because `ingested` and `duplicate` are disjoint buckets of an exact four-bucket partition [AC-052], [AC-091], an implementation following the former `ingested` clause that listed "(or an existing record's `is_seed` flag updated idempotently)" as an `ingested` event would either double-count the re-ingestion or mis-bucket it, breaking the partition and contradicting [FR-006] and [AC-090].

### Verification

- **AC-094 compliance check:** A re-ingestion of a seed whose identity already has a URL Record increments `urls_discovered_total{duplicate}` exactly once and does not increment `urls_discovered_total{ingested}`. A runtime seed injection for an identity that already has a record similarly increments only the `duplicate` bucket. Creating a genuinely new URL Record (including a redirect final-target upsert [R-062] that creates its target) increments `ingested`. A re-ingestion that finds no pre-existing record is `ingested`, not `duplicate`.

### Test Scenarios

| Scenario | Action | `urls_discovered_total{ingested}` | `urls_discovered_total{duplicate}` |
|---|---|---|---|
| Config seed re-ingestion on restart | Identity already has a record; `is_seed=true` updated idempotently | 0 | +1 |
| Runtime seed injection (existing record) | Identity already has a record; runtime injection | 0 | +1 |
| New URL discovery (new identity) | Truly new URL Record created | +1 | 0 |
| Redirect final-target upsert (new target) | Upsert creates new target record | +1 | 0 |

Both rows correctly specify that `ingested` and `duplicate` are disjoint buckets of an exact four-bucket partition [AC-052], [AC-091], and that `duplicate` is incremented only when an existing record's `is_seed` flag is updated idempotently [AC-094].

## AC-138: 429 and 5xx retryable rows increment hosts.consecutive_failures on either landing

The [DOC-09 §4] response-classification table's `429` and `5xx` retryable rows increment `hosts.consecutive_failures` [R-112] on *either* landing — a budget-exhausted `429`/`5xx` outcome that lands ST-180 still increments the host's `consecutive_failures`, because the increment keys on the outcome being RETRYABLE, not on the landing state [R-112], [AC-106], [AC-136]. (a) With [CFG-020]=2, two consecutive `429` responses on a Host: the first lands ST-150 with the host's `consecutive_failures` incremented [R-112]; the second — `attempts=2 ≥ [CFG-020]` — lands ST-180 with `urls.last_error_class='ERR-005'`, `urls.next_attempt_mono=NULL`, and `hosts.consecutive_failures` incremented again [R-112] (now 2, doubling the host-level dynamic backoff [DOC-08 §4]). (b) By contrast, a redirect final-target upsert [R-062] whose RETRYABLE outcome lands a *pre-existing target record* in ST-150 or ST-180 applies no host-counter effect to the target: all host-side effects of the chain (`consecutive_failures`, `pages_crawled`) attribute to the Host of the exchange that produced the outcome — the final hop's Host for a chain that fetched a response, or the source Host when no response arrived [R-112], [AC-116] — and the target's own host is left untouched. An implementation that ties the [R-112] increment to the ST-150 landing (skipping it on the ST-180 landing of (a)) diverges from the [R-112] invariant and leaves the budget-exhausted host's backoff understated; one that applies the source attempt's host-side effects to the upsert target (b) double-attributes counters that [R-112] reserves for the outcome-producing Host. This test pins the counter-increment contract across the retryable-status rows of the [DOC-09 §4] table and the [R-062] upsert path.

## AC-135: no-store carve-out in DOC-12 §4

The [DOC-12 §4] freshness/recrawl formula's "Otherwise" branch MUST exclude not only [CFG-025]=0 but also a `Cache-Control: no-store` success, which is one-shot for recrawl purposes and leaves `urls.due_at_mono` = NULL [R-221], [T-2], [AC-072]. With [CFG-025]>0, fetch an in-scope seed `https://host-a/page` returning `200 OK` with `Cache-Control: no-store`; the payload is fetched, stored [FR-043] and extracted normally [T-4], but (a) [T-2] leaves `urls.due_at_mono` NULL — it MUST NOT compute and install `fetch_complete_mono + interval` per the §4 formula — the ST-140→ST-100 due-time promotion [DOC-12 §1] never fires, and the record stays ST-140 (terminal for its fetch cycle [DOC-07 §2]) until a rediscovery refresh ([CFG-021]=true) re-admits it [FR-051], [R-052]; (b) by contrast, a plain 2xx success (or an UNCHANGED `304`) on the same host installs `due_at_mono = fetch_complete_mono + interval` with the deterministic [CFG-026] jitter and the [CFG-025] interval, and returns to ST-100 when due [FR-050], [AC-033], [AC-072]. An implementation following the former §4 "Otherwise" branch literally computes a due time for the no-store success, promotes it ST-140→ST-100 after the interval and refetches a page the server marked no-store — contradicting [R-221]'s "a T-2 completion that stores a no-store success sets `due_at_mono` = NULL", the [DOC-11 §1] schema comment "NULL for one-shot records ([CFG-025]=0, [R-221] no-store)", the [NFR-004] one-shot bound [AC-086] and [AC-072] — even though it handles [CFG-025]=0 and ordinary 2xx successes correctly. This test pins the no-store carve-out against the common blanket "On success" trigger anti-pattern.

## Conclusion

All retryable rows in the DOC-09 §4 response classification table have consistent budget conditioning (next_attempt_mono=NULL when budget exhausted), as required by AC-106, AC-136, and AC-085. The DOC-13 §3 `once_retried_classes` gate is evaluated before the budget check, per [R-232], ensuring that yes-once error classes (ERR-003, ERR-013) land ST-180 (DEAD) regardless of remaining budget. The DOC-00 Per-Host Concurrency entry is defined in tasks/units, not individual HTTP requests, consistent with [R-051], [R-131], [R-212] and [AC-011]/[AC-035]. The DOC-12 §4 recrawl formula's trigger covers both successful outcomes — SUCCESS and UNCHANGED — while a PERMANENT outcome installs no recrawl due time [AC-139]. The `urls_discovered_total` four-bucket partition is exact and disjoint, with `ingested` and `duplicate` correctly separated per [AC-094], [AC-091], [AC-090]. The specification is conformant.
