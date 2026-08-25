---
id: DOC-15
title: Observability
version: 1.21.0
---

# Observability

## 1. Metrics (expose via health endpoint and/or push; exact wire format free)

Counters:

| Metric | Labels | Meaning |
|---|---|---|
| urls_discovered_total | outcome{ingested,duplicate,excluded,dropped} | [FR-003..FR-004]; `ingested` also counts URL Records created by redirect final-target upserts [R-062] (they are not [FR-003]/[FR-004] ingestions); `duplicate` = ingestion events finding a pre-existing record — rediscoveries ([FR-051], [INV-5], including seed re-injection [FR-006]) and redirect final-target upserts onto pre-existing records [R-062] — so the four buckets partition every ingestion event exactly; `dropped` = no-record discards ([R-001]/[R-002] unacceptable URLs — runtime-seed rejects included [FR-006], which have no URL Identity to store but are counted here identically to discovery-time discards; OUT_OF_SCOPE with [CFG-038]=false; per-page candidate-cap overflow [R-159]) |
| state_transitions_total | from,to | every legal transition pair [ST-*]; `from=creation` labels the record-creating transitions — filter outcomes and the redirect final-target upsert [R-062] ([DOC-07 §2]) |
| fetch_attempts_total | outcome,error_class | FetchResult outcomes |
| bytes_downloaded_total | content_type_class | post-decode bytes of every response body received, whatever its later disposition — robots.txt bodies included ([DOC-08 §2.2]: the same fetch machinery, classed by its Content-Type like any body); bodies discarded by type ([ERR-008]) or size ([ERR-007]: the bytes received up to the abort) still measure ingress; a `304` body contributes nothing; class ∈ {html, xml, image, pdf, text, other} (html = text/html + xhtml; xml = application/xml + text/xml + rss + atom) |
| content_length_mismatch_total | — | counted when received body size ≠ the `Content-Length` header value [R-141] |
| robots_queries_total | verdict{allow,disallow,unknown} | [DOC-08]; matches the C4 verdict enum |
| exclusions_total | reason | cumulative entries into ST-190, by reason code — creations ([DOC-07 §2]) and later transitions in alike; monotonic, not the live count (that is the `excluded_count` gauge); the series is exposed regardless of [CFG-038], but OUT_OF_SCOPE drops under [CFG-038]=false create no ST-190 record and are counted only by `urls_discovered_total{dropped}` [FR-004] |

Gauges:

| Metric | Meaning |
|---|---|
| frontier_size | count ST-100 |
| inflight_global / inflight_per_host{host} | live concurrency |
| host_next_allowed_ms{host} | ms until host eligible |
| dead_count, excluded_count | lifecycle totals |
| suspicious_hosts | count of hosts with `suspicious=true` [R-402] |

Histograms: `fetch_duration_ms` (by outcome), `payload_size_bytes`.

- R-240: Counters are monotonic; gauges reflect current truth; all labeled
  values come from closed enums in this KB, except the per-host series'
  `host` label — a host key, unbounded in value but bounded in cardinality
  by the set of Hosts the crawl has contacted.

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

On graceful shutdown or every [CFG-045] completions, emit a `run_summary` event:
counts by final state, by error class, elapsed time, bytes total. This event is
the operator's primary progress signal. Process-level counters are
monotonic within a Run and reset on restart; `run_summary` events and the
`runs` table [DOC-11 §1] are the durable aggregate record referenced by
retention [DOC-11 §6].
