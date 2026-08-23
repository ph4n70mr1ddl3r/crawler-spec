---
id: DOC-11
title: Storage Model
version: 1.1.0
---

# Storage Model

Two stores only [DEC-003]. All timestamps UTC RFC 3339. All monotonic values
are integers (implementation may map to wall-clock offsets at rest).

## 1. Metadata Store schema (logical)

```sql
urls (
  url_identity      TEXT PRIMARY KEY,     -- normalized URL [DOC-06]
  raw_first_seen    TEXT NOT NULL,        -- first raw form observed
  state             TEXT NOT NULL,        -- ST-nnn code
  exclude_reason    TEXT NULL,            -- for ST-190
  priority          INT  NOT NULL DEFAULT 500,
  depth             INT  NOT NULL,
  source_run_id     INT  NOT NULL,
  discovered_from   TEXT NULL,            -- parent url_identity
  attempts          INT  NOT NULL DEFAULT 0,
  next_attempt_mono INT NULL,             -- for ST-150 backoff
  due_at_mono       INT NULL,             -- scheduler key component
  last_fetch_mono   INT NULL,
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
  payload_sha256    TEXT PRIMARY KEY,
  json              TEXT NOT NULL         -- [DOC-10 §3], deterministic
)

hosts (
  scheme_host_port  TEXT PRIMARY KEY,
  robots_state      TEXT NOT NULL,        -- OK|ALLOW_ALL|DEFERRED
  crawl_delay_s     INT NULL,
  robots_fetched_at TEXT NULL,
  robots_deferred_until_mono INT NULL,
  next_allowed_fetch_at_mono INT NOT NULL DEFAULT 0,
  inflight          INT  NOT NULL DEFAULT 0,
  consecutive_failures INT NOT NULL DEFAULT 0,
  pages_crawled     INT  NOT NULL DEFAULT 0
)

fetch_events (                       -- bounded ring buffer per [DOC-11 §6]
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
```

## 2. Content Store layout

```
blobs/ab/abcdef1234...   (SHA-256 hex; two-level fanout)
tmp/                     staging dir; atomic rename into place [R-300]
```

- R-300: Blob writes: write `tmp/<random>` → fsync → rename to final path. A blob file, once visible under its hash name, is immutable and complete.
- R-301: Before inserting a `pages` row referencing a hash, the blob MUST already exist on disk [INV-2].
- R-302: Orphan blobs (no referencing pages row) are garbage after retention sweep [§6]; deletion allowed only then.

## 3. Transactions & consistency

- T-1: Dispatch transaction = {urls.state→ST-110, hosts.inflight+1, hosts.next_allowed_fetch_at advance} — single commit [FR-012].
- T-2: Completion transaction = {urls.state update, pages insert, fetch_event insert, hosts.inflight−1, hosts counters} — single commit.
- T-3: Blob write happens BEFORE T-2 commits [R-301]; a crash between them leaves an orphan blob, cleaned by §6.

## 4. Interfaces for the Downstream Consumer

Read-only access via SQL over Metadata Store + direct file reads over Content Store. No writes permitted. Documented views:
- `v_latest_pages`: latest successful fetch per URL identity.
- `v_frontier`: urls where state=ST-100.

## 5. Scale assumptions

Design point: 10M URL records, 5M blobs, single host. All queries above must use indexes on (state), (due_at_mono), (url_identity).

## 6. Retention

- Retention job runs hourly:
  - fetch_events older than 7 days deleted (aggregate metrics preserved separately) [CFG-033 optional override];
  - pages + blobs whose `fetch_ts` < now − [CFG-027] deleted oldest-first;
  - DEAD/EXCLUDED url records older than 180 days deleted.
- Deletion order: artifacts → pages rows → blob files → url records, per-commit consistent so Consumer never sees dangling references.
