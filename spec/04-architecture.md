---
id: DOC-03
title: System Architecture
version: 1.13.0
---

# Architecture

## Component overview

```
                 ┌────────────────────────────────────────────────────┐
 Seeds ─────────►│  INGESTOR                                          │
                 │  accepts seeds, normalizes, filters, enqueues      │
                 └───────────────┬────────────────────────────────────┘
                                 ▼
                 ┌───────────────────────────┐
   Metadata ◄───►│  FRONTIER (priority due-  │
   Store         │  queue over URL Records)  │
                 └───────────────┬───────────┘
                                 │ next due URL(s)
                                 ▼
                 ┌───────────────────────────┐     ┌────────────────┐
                 │  SCHEDULER                │◄───►│ HOST REGISTRY  │
                 │  politeness windows,      │     │ per-host state │
                 │  priority, backoff        │     └────────────────┘
                 └───────────────┬───────────┘
                                 ▼
                 ┌───────────────────────────┐     ┌────────────────┐
                 │  FETCHER (worker pool)    │◄────│ ROBOTS MANAGER │
                 │  HTTP client, redirects,  │     │ cache per Host │
                 │  timeouts, SSRF guard     │     └────────────────┘
                 └───────────────┬───────────┘
                                 ▼
                 ┌───────────────────────────┐     ┌────────────────┐
                 │  EXTRACTOR                │────►│ CONTENT STORE  │
                 │  parse, extract links &   │     │ SHA-256 blobs  │
                 │  content, emit discoveries│     └────────────────┘
                 └───────────────┬───────────┘
                                 ▼
                          back to INGESTOR path
                                 
                 ┌───────────────────────────┐
                 │  OBSERVABILITY            │
                 │  metrics, logs, health    │
                 └───────────────────────────┘
```

## Components and contracts

### C1 INGESTOR
- Input: Seed URLs (config + runtime API) and discovered URLs from EXTRACTOR.
- Responsibilities: normalization ([DOC-06 §2]), scope filtering ([DOC-06 §4]),
  URL Record creation/upsert in Metadata Store, enqueue into Frontier.
- MUST be the only component that writes new URL Records with state ST-100.

### C2 FRONTIER
- A persistent, ordered collection of URL Records awaiting scheduling.
- Candidacy: `due_at_mono ≤ now`; selection order among due candidates:
  `(priority DESC, url_identity ASC)` [FR-010], [DOC-12 §1].
- Backed by the Metadata Store; MAY keep in-memory indexes for speed, but the
  store is authoritative at all times [G-2].

### C3 SCHEDULER
- Repeatedly selects due URLs from Frontier respecting Effective Delay per Host
  and concurrency limits [DOC-12 §3], [DOC-08 §4].
- Owns HOST REGISTRY: per-host monotonic `next_allowed_fetch_at`, inflight count,
  consecutive failure count, dynamic backoff multiplier.
- Emits fetch tasks to FETCHER workers; handles their results by updating URL
  Records and host state.

### C4 ROBOTS MANAGER
- Fetches/caches/parses robots.txt per Host [DOC-08 §2].
- Answers exactly one query: `allowed(url, ua_token) -> {ALLOW, DISALLOW, UNKNOWN}`
  plus the group's crawl-delay if present.

### C5 FETCHER
- Worker pool of size `global_concurrency` [CFG-010].
- Performs DNS resolution + SSRF checks [DOC-16 §2], HTTP exchange with all
  timeout/retry rules [DOC-09], redirect following within caps.
- Returns a FetchResult with exactly the fields defined in [DOC-09 §6]. Never
  parses bodies beyond content-type sniffing for caps.

### C6 EXTRACTOR
- For each successful fetch of a parseable type: decode charset, parse HTML,
  extract outbound links and content artifacts [DOC-10].
- Emits discovered URLs into INGESTOR and artifacts into stores.

### C7 METADATA STORE
- Relational store holding tables: `urls`, `pages`, `page_artifacts`, `hosts`,
  `fetch_events` (time-bounded), `runs`. Schemas in [DOC-11].

### C8 CONTENT STORE
- Immutable, content-addressed blob files: `blobs/<sha256[0:2]>/<sha256>`.
- Keyed by SHA-256 hex of payload bytes. Writes are atomic (temp+rename) [R-500].

### C9 OBSERVABILITY
- Metrics counters/gauges, structured JSON logs on stdout, optional HTTP health
  endpoint [DOC-15].

## Data flow invariants

- INV-1: Every URL identity dispatched from the Frontier exists as a URL Record
  before fetching begins. Redirect-hop targets are the sole exception: they are
  authorized in-flight by [R-131] and persisted on the source record's
  attempt row (`fetch_events.redirect_chain`)
  [R-133]; the chain's final target receives its own URL Record
  at completion per [R-062] [DOC-07 §4].
- INV-2: A payload in the Content Store is written before any metadata row that
  references its hash is committed (referential safety after crash).
- INV-3: The number of in-flight fetches to a Host NEVER exceeds
  Per-Host Concurrency, even across restarts (host state persisted).
- INV-4: Two fetches to the same Host are separated by ≥ Effective Delay,
  measured start-to-start, enforced via HOST REGISTRY.
- INV-5: Discovery of a URL already present as a URL Record never duplicates it;
  it updates `last_seen_at` ([FR-051] mandates the update in every rediscovery
  branch) and refreshes scheduling only if configured [CFG-021].

## Deployment model

Single OS process; graceful shutdown on SIGINT/SIGTERM (stop issuing fetches,
finish in-flight fetch tasks within their bounded timers — per-hop
[CFG-015], inter-hop waits ≤ [CFG-035] [DOC-09 §2], [R-131] — commit all
state, exit code 0).
Each process start opens a new Run record (after config validation succeeds
[DEC-011]); graceful shutdown closes it (`finished_at`); a Run whose process
died is finalized by the next startup. A Crawl Session [DOC-00] spans all
Runs of one crawl lineage — restarts open new Runs within the same session.
Crash recovery on startup: rebuild HOST REGISTRY and in-memory indexes from
Metadata Store; any URL Record left in {ST-110 (SCHEDULED), ST-120 (FETCHING)}
is reclassified per the crash rule [DOC-13 §5] — ST-150 (RETRY_WAIT) with
backoff, or ST-180 (DEAD) when the retry budget was exhausted [R-060] —
keeping its attempt count [DOC-07 §4].
