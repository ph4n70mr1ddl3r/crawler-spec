---
id: DOC-04
title: Functional Requirements
version: 1.33.0
---

# Functional Requirements

Numbering is grouped by component area. "The system" = the Crawler.

## Ingestion & scope (C1)

- FR-001: The system MUST accept a non-empty list of Seed URLs from configuration [CFG-001] and normalize each per [DOC-06 §2].
- FR-002: The system MUST reject, at startup, any seed that is not an absolute, parseable http(s) URL (there is no base to resolve against), whose scheme ∉ allowed schemes [CFG-003], or whose URL contains userinfo [R-002]; rejection of ≥1 seed aborts startup [DEC-011].
- FR-003: For every new normalized URL that passes the Scope predicate ([DOC-06 §4]) and all applicable pre-enqueue trap filters ([DOC-06 §5]) and enqueue-time caps ([FR-005]), the system MUST create a URL Record in state ST-100 — new identities only; rediscovery of an existing record never re-creates it and instead follows [FR-051]/[INV-5]. Passing the Scope predicate alone is not sufficient: a new identity that is IN_SCOPE but fails a trap filter (URL blocklist, param blocklist, path budget, depth limit [DOC-06 §5]) or an enqueue-time cap [FR-005] is recorded as ST-190 with the corresponding reason code ([DOC-06 §5], [FR-005]) and is never enqueued, exactly as [DOC-07 §2]'s two distinct creation edges — "filters passed [FR-003]" to ST-100 and "scope/trap/blocklist filter failed [FR-004], [DOC-06 §5], or enqueue-time cap reached [FR-005]" to ST-190 — require [AC-111]. Seed URLs have depth 0; discovered URLs have depth = parent depth + 1 [FR-042].
- FR-004: URLs failing the Scope predicate MUST be recorded as ST-190 with reason `OUT_OF_SCOPE` only if [CFG-038]=true; otherwise silently dropped. They MUST NOT be stored in the Frontier.
- FR-005: The system MUST enforce global caps at two points. (1) Enqueue time (early exclusion): before inserting a new URL Record, the system performs two checks. (A) Global cap: if the total count of URL Records in non-EXCLUDED states (i.e. all states except ST-190) across the whole crawl — every Registrable Domain, not just the candidate's — is already >= [CFG-005], the discovery is recorded ST-190/`CAP_REACHED` and never enqueued — the system-wide total of non-EXCLUDED records never exceeds [CFG-005], whose count excludes redirect final-target upserts [R-062] which are exempt from this global count as they are bounded by in-flight chains. [CFG-005] (`max_urls_total`, "total records" in [DOC-16 §3]) is a system-wide resource bound, unlike the per-Registrable-Domain budget [FR-005(1)(B)] keyed on [CFG-006]; it has no dispatch-time gate, so its enqueue-time check is the sole enforcement point. Note: ST-150 records are counted in this global cap since they are non-EXCLUDED states (unlike the per-domain check [FR-005(1)(B)] and dispatch-time gate [FR-005(2)] which exclude ST-150 records from their budget evaluation). (B) Per-domain budget: with D the candidate's Registrable Domain, if successes(D) + inflight(D) >= [CFG-006], where successes(D) = URL Records for D in {ST-130, ST-140} and inflight(D) = URL Records for D in {ST-110, ST-120}, the discovery is recorded ST-190/`CAP_REACHED` and never fetched. Note: at enqueue time the new URL Record would reside in state ST-100, which is not counted toward successes(D) or inflight(D), and ST-150 records are likewise not counted; therefore this per-domain check only triggers when the domain already holds URL Records in {ST-130, ST-140, ST-110, ST-120}. ST-190 audit records do not consume the [CFG-005] budget. Redirect final-target upserts [R-062] are exempt from the global cap [CFG-005] as they are bounded by in-flight chains; they count toward the target domain's per-domain cap [CFG-006] upon successful completion of the chain, and a domain can exceed [CFG-006] only by chains already in flight when the cap was first reached. (2) Dispatch time (authoritative gate [FR-011(e)]): a due candidate satisfying successes(D) + inflight(D) >= [CFG-006] moves to ST-190/`CAP_REACHED` and is never fetched. The dispatch-time gate is authoritative across all states: it re-evaluates the per-domain bound at scheduling time, and per AC-007, AC-008, and AC-075, ST-100‑queued and ST-150‑gated URLs bypass the enqueue-time per-domain check but remain subject to dispatch-time enforcement — the gate triggers when the domain already holds sufficient successes+inflight in {ST-130, ST-140, ST-110, ST-120}, regardless of how many records are queued as ST-100 or waiting as ST-150, using the same per-domain budget evaluation as FR-005(1)(B) (successes(D) = URL Records for D in {ST-130, ST-140}; inflight(D) = URL Records for D in {ST-110, ST-120}; successes(D)+inflight(D) >= [CFG-006]) to ensure consistent authoritative enforcement across both exclusion points, where the counting set {ST-130,ST-140,ST-110,ST-120} explicitly excludes ST-100 and ST-150 records from the per-domain budget evaluation at both enqueue and dispatch time.
- FR-006: Runtime seed injection via the operator API MUST behave identically to config seeds (same normalization, filtering, caps). A runtime seed that violates [FR-002] MUST be rejected with an error
response and MUST NOT be recorded — identically to discovery-time discards
[R-001], [R-002] (an invalid URL has no URL Identity to store, and no ST-190
reason code applies). A runtime seed violating [V-4] MUST be rejected with an error response and, when [CFG-038]=true, recorded ST-190/`OUT_OF_SCOPE`; the [V-4] PREFIX_LIST validation applies at runtime identically to startup: a seed that does not match any [CFG-039] entry is out of scope. The difference is that at startup the process aborts [DEC-011], while at runtime the seed is rejected and may be recorded as ST-190/`OUT_OF_SCOPE` per [CFG-038], allowing the crawl to continue. In both cases the running process MUST NOT abort. The seed set used by the Scope predicate ([DOC-06 §4]) is derived from all URL Records with `is_seed=true` — config seeds are re-ingested idempotently on every start [NFR-012], and injected seeds persist as seed records — so Crawl Scope is stable across restarts; a seed's own Registrable Domain or Host is part of the set when its own scope is evaluated, so a runtime seed may introduce a new domain/Host exactly as a config seed does [R-032]. Ingesting a seed whose identity already has a URL Record (config re-ingestion or runtime injection) sets `is_seed=true` on that record idempotently (with priority recomputed per [DOC-12 §2]) and otherwise follows rediscovery semantics [FR-051]: injection never re-activates DEAD or EXCLUDED records [DOC-13 §4] — the flag change alone expands the scope set and the priority seed bonus.

## Frontier & scheduling (C2/C3)

- FR-010: The Frontier MUST make dispatch candidates (ST-100 records with `due_at_mono ≤ now`) selectable in `(priority DESC, url_identity ASC)` order; `due_at_mono` determines candidacy, never the ordering of already-due work [DOC-12 §1], [DOC-12 §3].
- FR-011: The Scheduler MUST NOT dispatch a fetch to a Host unless all hold:
  (a) host inflight < [CFG-009]; (b) global inflight < [CFG-010];
  (c) monotonic now ≥ host.next_allowed_fetch_at; (d) robots gate = ALLOW for that URL [DOC-08 §3];
  (e) registrable-domain budget [FR-005]: successes(D) + inflight(D) < [CFG-006] — failure ⇒ ST-190/`CAP_REACHED` (exclusion, not deferral).
- FR-012: On dispatch, the system MUST atomically set the URL Record to ST-110,
  increment `urls.attempts` by one (so the attempt is counted even if the process
  dies mid-fetch [DOC-13 §5]), increment host inflight, and set
  host.next_allowed_fetch_at = max(next_allowed_fetch_at, now) + Effective Delay —
  atomic with respect to process death (single transaction) [DOC-08 §4], [T-1].
- FR-013: Priority MUST be computed per [DOC-12 §2] from depth, seed status, host failure history, prior host success, and manual boost [CFG-031]; range 0–1000, default 500.

## Fetching (C5)

- FR-020: The system MUST send requests with header `User-Agent` equal exactly to [CFG-018], plus `Accept`, `Accept-Encoding: gzip, deflate, br`, and `From` when [CFG-019] set. No cookies, no referer spoofing, no fingerprint rotation [DEC-010].
- FR-021: The system MUST follow HTTP 3xx redirects up to [CFG-017], applying SSRF checks and robots checks to every hop target [DOC-16 §2], [DOC-08 §3]. Redirect chains crossing hosts re-check each Host's gate.
- FR-022: The system MUST enforce timeouts [CFG-012..CFG-015] plus the DNS timeout [CFG-036] independently and abort on violation, classifying per [DOC-13].
- FR-023: The system MUST abort and discard a body exceeding [CFG-016], classifying the outcome PERMANENT with error_class ERR-007 [DOC-13 §1]; partial data MUST NOT be persisted.
- FR-024: The system MUST decode Content-Encoding (`gzip`, `deflate`, `br`) before hashing/storing the Payload.
- FR-025: The system MUST record every attempt as a fetch_event with status, timings, error class, final URL, payload hash (on success) [DOC-11 §1].

## Robots & politeness (C4)

- FR-030: While a Host's `robots_state` = INITIAL (i.e., before any authoritative verdict exists for that Host [R-106]), the system MUST obtain an authoritative robots verdict via [DOC-08 §2] (acquisition is single-flight per Host and initiated lazily [R-105]); UNKNOWN ⇒ defer, never fetch.
- FR-031: DISALLOW verdicts MUST move the URL Record to ST-190/`ROBOTS_DISALLOW` without fetching.
- FR-032: Effective Delay per Host = max([CFG-007], group crawl_delay if present, dynamic backoff if enabled) — all terms unit-normalized to milliseconds per [DOC-08 §4].

## Extraction & storage (C6/C7/C8)

- FR-040: Successful responses with content-type `text/html` or `application/xhtml+xml` MUST be parsed and links + content extracted per [DOC-10].
- FR-041: Other content types: store payload iff [CFG-028]=true and type ∈ allowed list [R-143]; skip parsing except recording `Content-Type` and length metadata [DEC-006]. Types outside the effective allowed list are classified PERMANENT/ERR-008 [DOC-13 §1] and the payload is discarded.
- FR-042: Every discovered link MUST be resolved against the page's extraction base ([R-020]: the final response URL, post-redirect, overridden by a valid `<base href>` per [R-021]), then normalized and filtered like a seed [FR-001..FR-005], with depth = parent depth + 1 [DEC-009] and `discovered_from` = the page's final URL identity (provenance of the discovery — not the resolution base).
- FR-043: Payloads MUST be stored in the Content Store keyed by SHA-256(payload bytes) [R-500]; byte-identical payloads MUST reuse the existing blob (no duplicate bytes) while page records reference the shared hash.
- FR-044: Page Records MUST capture: url identity, final URL identity, payload hash, content type, charset, length, fetch timestamp, http status, etag/last-modified (if present), title, canonical link rel=canonical if present, meta robots directives. Scalar columns live in `pages`; canonical URL and meta robots directives are captured in the page artifacts JSON [DOC-10 §3], which is part of the page record set [DOC-11 §1].
- FR-045: If `meta robots` contains `noindex`, the page MUST still be stored but flagged `noindex=true` for the Downstream Consumer; page-level `nofollow` MUST suppress ingestion of anchor-derived candidates from that page (outlinks are still recorded, flagged, per [DOC-10 §3]).

## Recrawl & lifecycle

- FR-050: Pages successfully fetched become eligible for recrawl after `recrawl_interval_s × (1 ± jitter)` [CFG-025], [CFG-026], unless the response's freshness headers dictate otherwise per [DOC-12 §4] (`Cache-Control: no-store` suppresses the recrawl due time entirely [R-221]; the 304-unchanged multiplier lengthens the interval, delaying the next recrawl [DOC-12 §4]).
- FR-051: Rediscovery of an existing terminal-success URL sets `last_seen_at`; if [CFG-021]=true it also moves the record back to ST-100 (due_at_mono := now, attempts := 0, priority recomputed per [R-052], [R-201]). Rediscovery of a record in any other state updates `last_seen_at` only and MUST NOT change its state: DEAD and EXCLUDED records are never re-activated automatically [DOC-13 §4], and scope/trap verdicts are final [R-031], [R-041]. The redirect
  final-target completion upsert [R-062] is not a rediscovery: its effects
  on a pre-existing target record are governed by [R-062].
- FR-052: Retryable failures follow [DOC-13 §3]: attempts up to [CFG-020], backoff per [CFG-022..CFG-024]; budget exhausted ⇒ ST-180 (DEAD).
