---
id: DOC-15
title: Observability
version: 1.0.0
---

# Observability

## 1. Metrics (expose via health endpoint and/or push; exact wire format free)

Counters:

| Metric | Labels | Meaning |
|---|---|---|
| urls_discovered_total | outcome{ingested,duplicate,excluded} | [FR-003..FR-004] |
| state_transitions_total | from,to | every legal transition pair [ST-*] |
| fetch_attempts_total | outcome,error_class | FetchResult outcomes |
| bytes_downloaded_total | content_type_class | post-decode payload sizes |
| robots_queries_total | verdict{allow,disallow,deferred} | [DOC-08] |
| exclusions_total | reason | ST-190 reasons |

Gauges:

| Metric | Meaning |
|---|---|
| frontier_size | count ST-100 |
| inflight_global / inflight_per_host{host} | live concurrency |
| host_next_allowed_ms{host} | ms until host eligible |
| dead_count, excluded_count | lifecycle totals |

Histograms: `fetch_duration_ms` (by outcome), `payload_size_bytes`.

- R-240: Counters are monotonic; gauges reflect current truth; all labeled values come from closed enums in this KB.

## 2. Structured logs

One JSON line per event on stdout. Required fields: `ts`, `level`, `component`
(C1–C9), `event`. Fetch logs add: `url_identity`, `attempt`, `outcome`,
`error_class`, `http_status`, `timings_ms`.

- R-241: No payload bodies in logs [NFR-016]; userinfo redacted [R-002].
- R-242: Log level from [CFG-032]; `debug` MAY add scheduler decision traces.

## 3. Health endpoint (optional, [CFG-034])

`GET /healthz` → 200 with JSON: `{state:"running|draining", uptime_s,
frontier_size, inflight_global, version, config_hash}`.
`GET /metrics` → metrics above. Bound to [CFG-034].

## 4. Run summary

On graceful shutdown or every N=1000 completions, emit a `run_summary` event:
counts by final state, by error class, elapsed time, bytes total. This event is
the operator's primary progress signal.
