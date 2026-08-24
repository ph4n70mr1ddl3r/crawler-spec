---
id: DOC-11
title: Storage Model
version: 1.14.0
---

# Storage Model

Two stores only [DEC-003]. All timestamps UTC RFC 3339. All monotonic values
are integers (implementation may map to wall-clock offsets at rest).

## 1. Metadata Store schema (logical)

```sql
urls (
  url_identity      TEXT PRIMARY KEY,     -- normalized URL [DOC-06]
  host_key          TEXT NOT NULL,        -- (scheme, hostname, port) of the identity [glossary]; host queries, trap-shape rebuild [R-042]
  registrable_domain TEXT NOT NULL,       -- eTLD+1; per-domain success cap key [FR-005]
  raw_first_seen    TEXT NOT NULL,        -- first raw form observed
  state             TEXT NOT NULL,        -- ST-nnn code
  exclude_reason    TEXT NULL,            -- for ST-190
  priority          INT  NOT NULL DEFAULT 500,
  depth             INT  NOT NULL,
  is_seed           BOOL NOT NULL DEFAULT FALSE,  -- priority input [DOC-12 §2]
  source_run_id     INT  NOT NULL,
  discovered_from   TEXT NULL,            -- parent url_identity
  attempts          INT  NOT NULL DEFAULT 0,
  last_error_class  TEXT NULL,            -- error class of the outcome that entered ST-180 [DOC-13 §4]; NULL after operator reset, for crash-exhausted records [R-060], and after a success upsert of a terminal record [R-062]
  consecutive_unchanged INT NOT NULL DEFAULT 0,  -- consecutive 304s; recrawl-interval doubling [DOC-12 §4]
  next_attempt_mono INT NULL,             -- for ST-150 backoff
  once_retried_classes TEXT NOT NULL DEFAULT '[]', -- JSON array: yes-once error classes [R-232] already retried in the current attempt cycle; cleared when attempts resets to 0
  due_at_mono       INT NULL,             -- scheduler key component
  last_fetch_mono   INT NULL,             -- completion time of the most recent fetch attempt (set in [T-2])
  last_seen_at      TEXT NULL,            -- most recent rediscovery [FR-051], [INV-5]
  created_at        TEXT NOT NULL,
  updated_at        TEXT NOT NULL
)

pages (
  url_identity      TEXT NOT NULL,
  run_id            INT  NOT NULL,
  fetch_ts          TEXT NOT NULL,
  final_url_identity TEXT NOT NULL,
  payload_sha256    TEXT NOT NULL,        -- FK → blobs (Content Store)
  http_status       INT  NOT NULL,
  content_type      TEXT NOT NULL,
  charset           TEXT NULL,
  payload_size      INT  NOT NULL,
  etag              TEXT NULL,
  last_modified     TEXT NULL,
  title             TEXT NULL,
  noindex           BOOL NOT NULL DEFAULT FALSE,
  parse_ok          BOOL NOT NULL DEFAULT TRUE,
  PRIMARY KEY (url_identity, run_id)
)

-- Within a single Run a URL is fetched at most once per recrawl cycle; if a
-- same-run refetch ever occurs (e.g., operator-triggered refresh), it UPSERTS
-- (replaces) the prior row for that (url_identity, run_id) key.

page_artifacts (
  payload_sha256    TEXT NOT NULL,
  final_url_identity TEXT NOT NULL,        -- link-resolution base [R-020]; artifacts are a
                                            -- function of bytes AND base [DOC-10 §3]
  json              TEXT NOT NULL,         -- [DOC-10 §3], deterministic
  PRIMARY KEY (payload_sha256, final_url_identity)
)

hosts (
  host_key          TEXT PRIMARY KEY,      -- (scheme, hostname, port) [glossary]; same key as urls.host_key
  robots_state      TEXT NOT NULL,        -- INITIAL|OK|ALLOW_ALL|DEFERRED
  crawl_delay_s     REAL NULL,           -- group Crawl-delay, seconds, exactly as received [R-102] (fractional values preserved)
  robots_rules      TEXT NULL,            -- parsed robots.txt rules JSON [DOC-08 §2.4]
  robots_fetched_at TEXT NULL,        -- wall-clock audit timestamp only
  robots_fetched_at_mono INT NULL,    -- robots TTL basis [CFG-008]; monotonic per [DEC-012] [DOC-08 §2.1]
  robots_deferred_until_mono INT NULL,
  robots_deferred_since_mono INT NULL,    -- first deferral of the current streak [R-103]
  robots_defer_failures INT NOT NULL DEFAULT 0, -- consecutive robots-acquisition failures; deferral-backoff exponent [DOC-08 §2.3]; cleared on an authoritative verdict
  next_allowed_fetch_at_mono INT NOT NULL DEFAULT 0,
  inflight          INT  NOT NULL DEFAULT 0,
  consecutive_failures INT NOT NULL DEFAULT 0,
  pages_crawled     INT  NOT NULL DEFAULT 0,   -- lifetime successful page fetches (SUCCESS|UNCHANGED) on this host; priority input [DOC-12 §2]
  suspicious        BOOL NOT NULL DEFAULT FALSE   -- set by R-402 [DOC-16 §2]
)

fetch_events (                       -- time-bounded per §6
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  url_identity      TEXT NOT NULL,
  run_id            INT  NOT NULL,
  ts                TEXT NOT NULL,
  attempt           INT  NOT NULL,
  outcome           TEXT NOT NULL,
  error_class       TEXT NULL,
  http_status       INT NULL,
  final_url_identity TEXT NULL,
  payload_sha256    TEXT NULL,
  redirect_chain    TEXT NULL,        -- JSON array of hop identities [R-133]
  timings_ms        TEXT NULL         -- JSON
)

runs (
  run_id    INTEGER PRIMARY KEY AUTOINCREMENT,
  started_at TEXT NOT NULL, finished_at TEXT NULL,
  config_hash TEXT NOT NULL, config_json TEXT NOT NULL
)
-- Run lifecycle [DOC-03 Deployment model]: one Run per process start, created after
-- config validation succeeds [DEC-011]; finished_at is set at graceful
-- shutdown; a Run whose process died is finalized (finished_at := crash-
-- detection time) by the next startup. A Crawl Session [DOC-00] spans all
-- Runs of one crawl lineage.
```

## 2. Content Store layout

```
blobs/ab/abcdef1234...   (SHA-256 hex; sharded on first two hex chars)
tmp/                     staging dir; atomic rename into place [R-500]
```

- R-500: Blob writes: write `tmp/<random>` → fsync → rename to final path. A blob file, once visible under its hash name, is immutable and complete.
- R-501: Before inserting a `pages` row referencing a hash, the blob MUST already exist on disk [INV-2].
- R-502: Orphan blobs (no referencing pages row) are garbage after retention sweep [§6]; deletion allowed only then.

## 3. Transactions & consistency

- T-1: Dispatch transaction = {urls.state→ST-110, urls.attempts+1 [FR-012], hosts.inflight+1, hosts.next_allowed_fetch_at advance} — single commit.
- T-2: Completion transaction = {urls.state update — together with the
  completion-time URL fields it owns: `last_fetch_mono` := attempt
  completion time, the recrawl `due_at_mono` and `consecutive_unchanged`
  [DOC-12 §4], and, on failures, `last_error_class` and
  `once_retried_classes` [DOC-13 §3], [R-232] — pages insert(s),
  fetch_event insert, hosts.inflight−1 for the single Host unit the attempt
  still holds, hosts counters} — single commit. Under [R-051]/[R-131] unit accounting an attempt holds at most one Host unit at any time; at completion it is the unit of the Host of the attempt's most recent request — the source Host when the chain never left it, otherwise the final hop's Host (intermediate Hosts' units were released when their responses arrived). A chain aborted before sending its next hop request — hop-gate refusal ([ERR-004]/[ERR-015]/[ERR-017]/[ERR-019]), loop or redirect-cap exhaustion ([ERR-011]), target-Host robots deferral ([ERR-010]), or an over-threshold politeness wait ([ERR-018]) — holds NO Host unit at completion: the responding Host's unit was already released when its response arrived, and the hop gate checks run after that release and before any wait [R-131]; T-2 decrements none (decrementing "the Host whose response carried the unfollowable redirect" would double-release a unit already released). Host counters (`pages_crawled`, `consecutive_failures`) attribute to the Host of the final response [R-112] — the same Host whose unit is decremented. The `pages` insert(s) apply to SUCCESS/UNCHANGED outcomes only; failure completions commit the same set minus the `pages` insert(s). A redirect-chain success commits two `pages` rows in this one transaction — the source's (`final_url_identity` = target identity) and the final target's [R-062].
- T-3: Blob write happens BEFORE T-2 commits [R-501]; a crash between them leaves an orphan blob, cleaned by §6.

## 4. Interfaces for the Downstream Consumer

Read-only access via SQL over Metadata Store + direct file reads over Content Store. No writes permitted. Documented views:
- `v_latest_pages`: latest successful fetch per URL identity.
- `v_frontier`: urls where state=ST-100.

## 5. Scale assumptions

Design point: 10M URL records, 5M blobs, single host. All queries above must use indexes on (state), (due_at_mono), (url_identity), (host_key), (registrable_domain), plus composites (state, priority DESC, url_identity ASC) for dispatch selection [FR-010], [NFR-002] and (registrable_domain, state) for the cap gates [FR-005], [FR-011(e)]. The retention sweeps [§6] additionally require indexes on (fetch_events.ts), (pages.fetch_ts), (pages.payload_sha256), (pages.final_url_identity, payload_sha256) for the blob/artifact reference checks, and on (state, updated_at) plus (state, last_seen_at) for the DEAD/EXCLUDED url-record scan (retention basis: `last_seen_at` when set, else `updated_at` [§6]).

## 6. Retention

- The retention job — an in-process maintenance task of the single crawler
  process [DEC-002] — runs every [CFG-046] seconds:
  - fetch_events older than [CFG-033] days deleted (aggregate metrics preserved separately);
  - pages rows whose `fetch_ts` < now − [CFG-027] deleted oldest-first;
  - a page_artifacts row MAY be deleted only when NO remaining `pages` row
    references its `(payload_sha256, final_url_identity)` pair; a blob file
    MAY be deleted only when NO remaining `pages` row references its
    payload_sha256 (payloads are shared across URL identities via dedup
    [AC-042] — age alone never justifies deletion);
  - DEAD/EXCLUDED url records not re-discovered for [CFG-043] days —
    `last_seen_at` when set, else `updated_at` — deleted. A deleted record
    MAY be re-created by later discovery; that bounded re-evaluation
    (≤ [CFG-020] attempts per [CFG-043]-day cycle) is not automated
    re-activation [DOC-13 §4].
- Deletion order within each sweep: pages rows → page_artifacts rows and blobs
  left with no remaining `pages`-row reference after those deletions —
  including crash-orphaned blobs from [T-3] [AC-031] — → url records. Each step commits before
  the next begins, so the Consumer never sees dangling references. (Deleting
  artifacts or blobs before the pages rows that reference them would contradict
  the unreferenced-only rule above.)
