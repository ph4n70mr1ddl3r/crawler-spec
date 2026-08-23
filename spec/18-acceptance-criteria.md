---
id: DOC-17
title: Acceptance Criteria
version: 1.7.0
---

# Acceptance Criteria

Each criterion is verifiable against a test fixture suite. v1 is complete when
all pass [DOC-01 §Success]. Fixtures use a local HTTP test server + recorded
responses; fixture configs set [CFG-042]=true so the local server is reachable
under the fail-closed egress policy — SSRF criteria (AC-025) run with it
false. Politeness tests use virtual time where possible [DEC-012].

## Ingestion & normalization

- AC-001: For each case in a 50-case normalization vector set (percent-encoding incl. reserved-octet preservation and unreserved decoding, uppercase hex, default ports, dot-segments, fragments, IDN, uppercase hosts, empty paths), output matches the expected identity exactly; function is pure (repeat calls identical).
- AC-002: Seeds that are unparseable, have a disallowed scheme, or contain userinfo abort startup with exit code ≠ 0 before any network I/O.
- AC-003: A page containing 100 duplicate links yields exactly one new URL Record.
- AC-004: With scope_mode=SEED_DOMAINS, a link from sub.example.org to example.org is IN_SCOPE; to other.org is OUT_OF_SCOPE and never fetched.
- AC-005: Startup with scope_mode=PREFIX_LIST where a seed matches no entry of [CFG-039] aborts with exit code ≠ 0 before any network I/O [V-4].
- AC-006: With [CFG-006]=2, exactly two page successes per Registrable Domain are fetched; further in-scope discoveries on that domain are recorded ST-190/`CAP_REACHED` and never fetched, other domains are unaffected, and ST-190 audit records do not consume the [CFG-005] budget [FR-005]. The bound holds even when all discoveries enqueue before the first fetch on the domain completes (dispatch-time gate [FR-011(e)]).

## Politeness, robots & scheduling

- AC-010: With two workers and CFG-007=5000ms, request starts to one host are ≥ 5000 ms apart across 20 fetches (start-to-start).
- AC-011: Per-host inflight never exceeds CFG-009 under any fixture; global inflight never exceeds CFG-010.
- AC-012: robots.txt Disallow for UA group blocks matching URLs → ST-190/ROBOTS_DISALLOW; Allow longer-match wins on tie per [R-101]; `*` ignored when token group exists. Rule matching per [R-101]: values match byte-wise, case-sensitively, against path plus `?query` when present (`Disallow: /*?` blocks a URL with any query); a terminal `$` anchors the end; a `*` matches any sequence incl. empty; no group matches and no `*` group exists ⇒ all URLs allowed; an empty `Disallow` value ⇒ allow all.
- AC-013: robots.txt returning 503 defers ALL host fetches; retry after backoff succeeds once robots returns 200; no page was fetched during deferral.
- AC-014: Crawl-delay=12s honored over CFG-007=5s; Retry-After=120s overrides backoff when larger.
- AC-015: A redirect whose hop target is disallowed by the target Host's robots.txt terminates the chain with outcome=PERMANENT and error_class=ERR-017; the source URL → ST-180; the target is never fetched; the hop appears in the source's `redirect_chain` [R-131].
- AC-016: Between dispatch [T-1] and request send, a robots cache refresh to DISALLOW moves the record ST-110→ST-190/`ROBOTS_DISALLOW` and releases its slot; a refresh to UNKNOWN (Host deferred) compensates the record back to ST-100. In both cases no request is sent, no fetch_event is written, and the [T-1] attempts increment is rolled back [R-053], [R-054]; no request is sent while the Host is deferred.
- AC-017: A redirect hop whose target Host is robots-deferred (verdict UNKNOWN) aborts the chain with outcome=RETRYABLE and error_class=ERR-010; the source → ST-150; no request is sent to the target Host during deferral [R-131].
- AC-018: With two due candidates on different Hosts — priorities 900 and 100, the lower-priority one due earlier — the higher-priority candidate is dispatched first: `due_at_mono` never orders already-due work [FR-010].
- AC-019: On robots.txt cache TTL expiry [CFG-008] with a cached entry, gate queries keep using the previous rules while the revalidation fetch is in flight; a 2xx revalidation swaps in the new rules atomically; a failing revalidation defers the Host [R-104].

## Fetching & errors

- AC-020: 6-hop redirect chain with CFG-017=5 stops at hop 5; outcome=PERMANENT with error_class=ERR-011 recorded [DOC-09 §6].
- AC-021: Redirect loop (A→B→A) detected at first repetition.
- AC-022: Payload of CFG-016+1 bytes aborted mid-stream; nothing persisted for it; ERR-007 recorded.
- AC-023: gzip body decoded before hashing; stored hash equals SHA-256 of decoded bytes.
- AC-024: 429 with Retry-After honored; attempts stop at CFG-020; then DEAD. A 429 `Retry-After` larger than [CFG-035] is honored unclamped [DOC-13 §3].
- AC-025: Connection to a host resolving to 127.0.0.1 (from a page link) is blocked with ERR-004 and never connects.
- AC-026: A redirect chain crossing into a second host waits for that host's politeness window before each hop request [R-131]; on success the final target has its own URL Record with depth equal to the source's [R-062], and the chain is persisted on the source's fetch_events row [R-133].
- AC-027: With robots.txt persistently returning 5xx for ≥ [CFG-040], gated URLs transition to ST-190/`ROBOTS_UNKNOWN_TIMEOUT` and are never fetched [R-103].
- AC-028: A 3xx response with a missing or unparsable `Location` header yields outcome=PERMANENT with error_class=ERR-011; no follow occurs [DOC-09 §4].
- AC-029: With [CFG-028]=false, fetching an allowed non-HTML type (e.g. `application/pdf`) stores no blob, records error_class=ERR-008, and the URL → ST-180; with [CFG-028]=true the payload is stored and no parse artifacts are produced [R-143].

## State machine & durability

- AC-030: kill -9 during ST-120 leaves record resumable; after restart it re-enters ST-100 with attempts preserved and no politeness violation (next_allowed_fetch_at respected).
- AC-031: kill -9 after blob write but before T-2 commit leaves an orphan blob that the retention sweep removes; no pages row references a missing blob ever (invariant scan passes).
- AC-032: Full restart with unchanged config produces zero duplicate ingestions [NFR-012].
- AC-033: Replay of a recorded fixture produces identical fetch decision sequences across three runs [NFR-006].

## Extraction & storage

- AC-040: Known HTML fixture yields exact expected artifact JSON (title, canonical, headings, main_text, outlinks) byte-identically across runs.
- AC-041: Page with meta robots=nofollow yields zero ingested anchor candidates but stores the page.
- AC-042: Byte-identical payload from two different URLs shares one blob file; both pages rows reference its hash. Two URLs on different Hosts returning byte-identical HTML containing relative links additionally hold distinct page_artifacts rows, keyed `(payload_sha256, final_url_identity)` — mirrors share bytes, never each other's resolved outlinks [DOC-11 §1].
- AC-043: Retention job deletes pages older than CFG-027 including blobs, never leaving dangling references mid-sweep; a blob/artifact row still referenced by any surviving pages row is retained even when individual referencing pages are deleted.

## Operations

- AC-050: SIGTERM triggers drain: no new dispatches, in-flight complete ≤ total_transfer_timeout, exit code 0, run_summary emitted.
- AC-051: Startup with 1M-record store reaches first fetch within 60 s [NFR-005].
- AC-052: All metrics counters appear after exercise; state_transitions_total contains only legal transition pairs.
- AC-053: Operator reset of a DEAD URL via the runtime API returns the record to ST-100 with attempts=0 and last error class cleared, emits the ST-180→ST-100 transition metric [DOC-15 §1], and writes an audit log entry [DOC-13 §4].
- AC-054: With [CFG-034] bound to a non-loopback address, the mutating operator actions (seed injection, DEAD reset, drain trigger) are disabled, a startup WARN is logged, and /healthz and /metrics remain available [R-406].
- AC-055: A redirect hop whose target Host's Crawl-delay implies a politeness wait > [CFG-035] aborts the chain with outcome=RETRYABLE and error_class=ERR-018; the source → ST-150; no worker slot or host concurrency slot is held during the wait; `next_attempt_mono` is floored at the target Host's window opening; if the window never opens within the retry budget, the URL reaches ST-180 [R-131], [DOC-13 §3].
- AC-056: Hop-target acceptance [R-130], [R-131], one fixture per case: (a) a relative `Location` value resolves against the hop URL to the correct absolute target (chain follows it); (b) a `Location` with a non-http(s) scheme yields outcome=PERMANENT, error_class=ERR-011, no follow; (c) a hop target matching a [CFG-037] pattern yields outcome=PERMANENT, error_class=ERR-019, source → ST-180, target never fetched, hop recorded in `redirect_chain`.
- AC-057: With scope_mode=SEED_DOMAINS and an IP-literal seed (e.g. `http://93.184.216.34/`), other URLs on the same IP literal are IN_SCOPE and every named host is OUT_OF_SCOPE [DOC-00]; `[2001:0DB8::1]` and `[2001:db8::1]` normalize to one URL Identity [R-003].
- AC-058: Runtime seed injection of a URL violating [FR-002] (bad scheme, unparseable, userinfo) or, with scope_mode=PREFIX_LIST, matching no [CFG-039] entry [V-4] returns an error, does not abort the process, and records ST-190 (`OUT_OF_SCOPE` where applicable and [CFG-038]=true); a valid injection behaves identically to a config seed [FR-006].
