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

## Conclusion

All retryable rows in the DOC-09 §4 response classification table have consistent budget conditioning (next_attempt_mono=NULL when budget exhausted), as required by AC-106, AC-136, and AC-085. The specification is conformant.
