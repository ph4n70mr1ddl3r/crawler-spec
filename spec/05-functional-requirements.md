---
id: DOC-04
title: Functional Requirements
version: 1.4.0
---

# Functional Requirements

Numbering is grouped by component area. "The system" = the Crawler.

## Ingestion & scope (C1)

- FR-001: The system MUST accept a non-empty list of Seed URLs from configuration [CFG-001] and normalize each per [DOC-06 §2].
- FR-002: The system MUST reject, at startup, any seed that is not an absolute, parseable http(s) URL (there is no base to resolve against), whose scheme ∉ allowed schemes [CFG-003], or whose URL contains userinfo [R-002]; rejection of ≥1 seed aborts startup [DEC-011].
- FR-003: For every normalized URL passing the Scope predicate ([DOC-06 §4]) the system MUST create or refresh a URL Record and set state ST-100. Seed URLs have depth 0; discovered URLs have depth = parent depth + 1 [FR-042].
- FR-004: URLs failing the Scope predicate MUST be recorded as ST-190 with reason `OUT_OF_SCOPE` only if [CFG-038]=true; otherwise silently dropped. They MUST NOT be stored in the Frontier.
- FR-005: The system MUST enforce global caps before enqueueing: total URL records in non-EXCLUDED states ≤ [CFG-005]; per-Registrable-Domain page successes ≤ [CFG-006]. At cap, new discoveries are recorded as ST-190/`CAP_REACHED` and never fetched (ST-190 audit records do not consume the [CFG-005] budget).
- FR-006: Runtime seed injection via the operator API MUST behave identically to config seeds (same normalization, filtering, caps).

## Frontier & scheduling (C2/C3)

- FR-010: The Frontier MUST order due work by `(due_at_monotonic ASC, priority DESC, url_identity ASC)` [DOC-12 §1].
- FR-011: The Scheduler MUST NOT dispatch a fetch to a Host unless all hold:
  (a) host inflight < [CFG-009]; (b) global inflight < [CFG-010];
  (c) monotonic now ≥ host.next_allowed_fetch_at; (d) robots gate = ALLOW for that URL [DOC-08 §3].
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
- FR-023: The system MUST abort and discard a body exceeding [CFG-016], classifying the outcome PERMANENT with error_class ERR-007 [DOC-09 §6]; partial data MUST NOT be persisted.
- FR-024: The system MUST decode Content-Encoding (`gzip`, `deflate`, `br`) before hashing/storing the Payload.
- FR-025: The system MUST record every attempt as a fetch_event with status, timings, error class, final URL, payload hash (on success) [DOC-11 §1].

## Robots & politeness (C4)

- FR-030: While a Host's `robots_state` = INITIAL (i.e., before any first fetch to that Host), the system MUST obtain an authoritative robots verdict via [DOC-08 §2]; UNKNOWN ⇒ defer, never fetch.
- FR-031: DISALLOW verdicts MUST move the URL Record to ST-190/`ROBOTS_DISALLOW` without fetching.
- FR-032: Effective Delay per Host = max([CFG-007], group crawl_delay if present, dynamic backoff if enabled) [DOC-08 §4].

## Extraction & storage (C6/C7/C8)

- FR-040: Successful responses with content-type `text/html` or `application/xhtml+xml` MUST be parsed and links + content extracted per [DOC-10].
- FR-041: Other content types: store payload iff [CFG-028]=true and type ∈ allowed list [R-143]; skip parsing except recording `Content-Type` and length metadata [DEC-006]. Types outside the effective allowed list are classified PERMANENT/ERR-008 [DOC-13 §1] and the payload is discarded.
- FR-042: Every discovered link MUST be resolved against the final response URL (post-redirect), then normalized and filtered like a seed [FR-001..FR-005], with depth = parent depth + 1 [DEC-009].
- FR-043: Payloads MUST be stored in the Content Store keyed by SHA-256(payload bytes) [R-500]; byte-identical payloads MUST reuse the existing blob (no duplicate bytes) while page records reference the shared hash.
- FR-044: Page Records MUST capture: url identity, final URL identity, payload hash, content type, charset, length, fetch timestamp, http status, etag/last-modified (if present), title, canonical link rel=canonical if present, meta robots directives. Scalar columns live in `pages`; canonical URL and meta robots directives are captured in the page artifacts JSON [DOC-10 §3], which is part of the page record set [DOC-11 §1].
- FR-045: If `meta robots` contains `noindex`, the page MUST still be stored but flagged `noindex=true` for the Downstream Consumer; `nofollow` MUST suppress link extraction from that page.

## Recrawl & lifecycle

- FR-050: Pages successfully fetched become eligible for recrawl after `recrawl_interval_s × (1 ± jitter)` [CFG-025], [CFG-026], unless freshness headers dictate otherwise per [DOC-12 §4].
- FR-051: Rediscovery of an existing terminal-success URL sets `last_seen_at`; if [CFG-021]=true it also moves the record back to ST-100 (due_at_mono := now, attempts := 0, priority recomputed per [R-052], [R-201]).
- FR-052: Retryable failures follow [DOC-13 §3]: attempts up to [CFG-020], backoff per [CFG-022..CFG-024]; budget exhausted ⇒ ST-180 (DEAD).
