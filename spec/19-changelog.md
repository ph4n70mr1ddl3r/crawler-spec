---
id: DOC-18
title: Changelog
version: 1.2.0
---

# Changelog

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
