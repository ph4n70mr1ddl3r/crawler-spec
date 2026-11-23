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

Both rows correctly specify `urls.next_attempt_mono=NULL` when the retry budget is exhausted, aligning them with the `ERR-018` row and [R-131] step-3 narration. An implementation omitting `urls.next_attempt_mono=NULL` would re-arm a backoff timer on budget-exhausted records, stranding records in ST-150 with `attempts >= [CFG-020]` and violating the "non-NULL `next_attempt_mono` only in ST-150" field-hygiene invariant [AC-085].

## 3xx Hop Robots Rows

The [DOC-09 §4] response-classification table's hop-blocked-by-robots row MUST split the DISALLOW case from the UNKNOWN/deferred case:

- **DISALLOW**: chain terminates at that hop with outcome=PERMANENT, `error_class='ERR-017'`, source → ST-180
- **UNKNOWN (Host deferred)**: chain aborts with outcome=RETRYABLE, `error_class='ERR-010'`; with `attempts < [CFG-020]` the source lands in ST-150 with a backoff `next_attempt_mono` computed per [DOC-13 §3]; with the budget exhausted it lands in ST-180 with `urls.last_error_class='ERR-010'` and `urls.next_attempt_mono=NULL` [AC-112], [AC-115], [AC-118]

### Verification

Row 14 in the DOC-09 §4 table has been verified to have correct budget conditioning for the UNKNOWN/deferred case, specifying `next_attempt_mono=NULL` when the retry budget is exhausted.

## Conclusion

All retryable rows in the DOC-09 §4 response classification table have consistent budget conditioning (next_attempt_mono=NULL when budget exhausted), as required by AC-106 and AC-136. The specification is conformant.
