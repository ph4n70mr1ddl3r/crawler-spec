---
id: DOC-17
title: Acceptance Criteria
version: 1.3.0
---

# Acceptance Criteria

Each criterion is verifiable against a test fixture suite. v1 is complete when
all pass [DOC-01 §Success]. Fixtures use a local HTTP test server + recorded
responses; fixture configs set [CFG-042]=true so the local server is reachable
under the fail-closed egress policy — SSRF criteria (AC-025) run with it
false. Politeness tests use virtual time where possible [DEC-012].

## Ingestion & normalization

- AC-001: For each case in a 50-case normalization vector set (percent-encoding, default ports, dot-segments, fragments, IDN, uppercase hosts, empty paths), output matches the expected identity exactly; function is pure (repeat calls identical).
- AC-002: Seeds with disallowed scheme or userinfo abort startup with exit code ≠ 0 before any network I/O.
- AC-003: A page containing 100 duplicate links yields exactly one new URL Record.
- AC-004: With scope_mode=SEED_DOMAINS, a link from sub.example.org to example.org is IN_SCOPE; to other.org is OUT_OF_SCOPE and never fetched.
- AC-005: Startup with scope_mode=PREFIX_LIST where a seed matches no entry of [CFG-039] aborts with exit code ≠ 0 before any network I/O [V-4].

## Politeness & robots

- AC-010: With two workers and CFG-007=5000ms, request starts to one host are ≥ 5000 ms apart across 20 fetches (start-to-start).
- AC-011: Per-host inflight never exceeds CFG-009 under any fixture; global inflight never exceeds CFG-010.
- AC-012: robots.txt Disallow for UA group blocks matching URLs → ST-190/ROBOTS_DISALLOW; Allow longer-match wins on tie per [R-101]; `*` ignored when token group exists.
- AC-013: robots.txt returning 503 defers ALL host fetches; retry after backoff succeeds once robots returns 200; no page was fetched during deferral.
- AC-014: Crawl-delay=12s honored over CFG-007=5s; Retry-After=120s overrides backoff when larger.

## Fetching & errors

- AC-020: 6-hop redirect chain with CFG-017=5 stops at hop 5; outcome=PERMANENT with error_class=ERR-011 recorded [DOC-09 §6].
- AC-021: Redirect loop (A→B→A) detected at first repetition.
- AC-022: Payload of CFG-016+1 bytes aborted mid-stream; nothing persisted for it; ERR-007 recorded.
- AC-023: gzip body decoded before hashing; stored hash equals SHA-256 of decoded bytes.
- AC-024: 429 with Retry-After honored; attempts stop at CFG-020; then DEAD.
- AC-025: Connection to a host resolving to 127.0.0.1 (from a page link) is blocked with ERR-004 and never connects.
- AC-026: A redirect chain crossing into a second host waits for that host's politeness window before each hop request [R-131]; on success the final target has its own URL Record with depth equal to the source's [R-062], and the chain is persisted on the source's fetch_events row [R-133].
- AC-027: With robots.txt persistently returning 5xx for ≥ [CFG-040], gated URLs transition to ST-190/`ROBOTS_UNKNOWN_TIMEOUT` and are never fetched [R-103].

## State machine & durability

- AC-030: kill -9 during ST-120 leaves record resumable; after restart it re-enters ST-100 with attempts preserved and no politeness violation (next_allowed_fetch_at respected).
- AC-031: kill -9 after blob write but before T-2 commit leaves an orphan blob that the retention sweep removes; no pages row references a missing blob ever (invariant scan passes).
- AC-032: Full restart with unchanged config produces zero duplicate ingestions [NFR-012].
- AC-033: Replay of a recorded fixture produces identical fetch decision sequences across three runs [NFR-006].

## Extraction & storage

- AC-040: Known HTML fixture yields exact expected artifact JSON (title, canonical, headings, main_text, outlinks) byte-identically across runs.
- AC-041: Page with meta robots=nofollow yields zero ingested anchor candidates but stores the page.
- AC-042: Byte-identical payload from two different URLs shares one blob file; both pages rows reference its hash.
- AC-043: Retention job deletes pages older than CFG-027 including blobs, never leaving dangling references mid-sweep; a blob/artifact row still referenced by any surviving pages row is retained even when individual referencing pages are deleted.

## Operations

- AC-050: SIGTERM triggers drain: no new dispatches, in-flight complete ≤ total_transfer_timeout, exit code 0, run_summary emitted.
- AC-051: Startup with 1M-record store reaches first fetch within 60 s [NFR-005].
- AC-052: All metrics counters appear after exercise; state_transitions_total contains only legal transition pairs.
