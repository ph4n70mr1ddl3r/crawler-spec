---
id: DOC-18
title: Changelog
version: 1.4.0
---

# Changelog

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
