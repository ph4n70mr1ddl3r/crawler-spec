---
id: DOC-14
title: Configuration Reference
version: 1.8.0
---

# Configuration

Single validated document [DEC-011]. Unknown keys ⇒ startup error.
Every parameter is referenced elsewhere only by its CFG id.

## General

| ID | Name | Type | Default | Range / notes |
|----|------|------|---------|---------------|
| CFG-001 | seeds | URL list | — required, ≥1 | normalized+filtered at load |
| CFG-002 | scope_mode | enum | `SEED_DOMAINS` | `SEED_DOMAINS \| SEED_HOSTS \| PREFIX_LIST` |
| CFG-003 | allowed_schemes | list | `[https, http]` | subset of {http, https} |
| CFG-039 | scope_prefix_list | list | `[]` | required iff scope_mode=PREFIX_LIST; entries are absolute http(s) URLs used verbatim (not normalized) — only scheme, host, and path participate in matching (query/fragment ignored); scheme/host compare case-insensitively, path verbatim; an empty entry path matches every path on its host, an entry path of `/` matches only the root [DOC-06 §4] |
| CFG-004 | max_depth | int | 5 | 0–50 |
| CFG-005 | max_urls_total | int | 100000 | >0 |
| CFG-006 | max_pages_per_registrable_domain | int | 10000 | >0; per Registrable Domain [FR-005] |

## Politeness & concurrency

| ID | Name | Type | Default | Range |
|----|------|------|---------|-------|
| CFG-007 | politeness_delay_ms | int | 5000 | 0–600000 |
| CFG-008 | robots_cache_ttl_s | int | 3600 | 60–86400 |
| CFG-009 | per_host_concurrency | int | 2 | 1–16 |
| CFG-010 | global_concurrency | int | 64 | 1–1024 |
| CFG-011 | dynamic_backoff_enabled | bool | true | |
| CFG-040 | robots_defer_max_s | int | 86400 (24 h) | 60–604800; cap for robots-deferral exponential backoff [DOC-08 §2], and the continuous-deferral duration after which gated URLs go ST-190/`ROBOTS_UNKNOWN_TIMEOUT` [R-103] |

## Fetching

| ID | Name | Type | Default | Range |
|----|------|------|---------|-------|
| CFG-012 | connect_timeout_ms | int | 10000 | 1000–60000 |
| CFG-036 | dns_timeout_ms | int | 10000 | 1000–60000 [DOC-09 §2] |
| CFG-013 | tls_handshake_timeout_ms | int | 10000 | 1000–60000 |
| CFG-014 | response_header_timeout_ms | int | 30000 | 1000–120000 |
| CFG-015 | total_transfer_timeout_ms | int | 300000 | ≥ header timeout |
| CFG-016 | max_payload_bytes | int | 5242880 (5 MiB) | 1–20971520 (20 MiB hard cap) |
| CFG-017 | max_redirects | int | 5 | 0–10 |
| CFG-018 | user_agent_token | string | — required | pattern: `name/version (+url)` |
| CFG-019 | from_header_email | string | null | valid email if set |

## Retry

| ID | Name | Type | Default | Range |
|----|------|------|---------|-------|
| CFG-020 | retry_budget | int | 3 | 1–10 (total attempts) |
| CFG-022 | retry_backoff_base_ms | int | 2000 | 100–60000 |
| CFG-023 | retry_backoff_factor | float | 2.0 | 1.0–10.0 |
| CFG-024 | retry_backoff_jitter | float | 0.25 | 0–0.5 (fraction) |
| CFG-035 | max_backoff_delay_ms | int | 3600000 (1 h) | 1000–86400000; cap for per-URL retry delay [DOC-13 §3] and per-host dynamic backoff [DOC-08 §4], and the redirect-hop politeness-wait threshold ([ERR-018] [R-131]) |

## Recrawl & refresh

| ID | Name | Type | Default | Range |
|----|------|------|---------|-------|
| CFG-021 | refresh_on_rediscovery | bool | false | [FR-051] |
| CFG-025 | recrawl_interval_s | int | 604800 (7 d) | 0 = never recrawl |
| CFG-026 | recrawl_jitter_fraction | float | 0.2 | 0–0.9 |

## Storage & traps

| ID | Name | Type | Default | Range |
|----|------|------|---------|-------|
| CFG-027 | retention_days | int | 90 | 0 = keep forever |
| CFG-028 | store_non_html | bool | true | allowed types [R-143] |
| CFG-029 | url_param_blocklist | list | `["sid","sessionid","phpsessid","jsessionid"]` | glob patterns on param names |
| CFG-030 | max_similar_paths_per_host | int | 500 | >0; trap guard [DOC-06 §5] — at most [CFG-030] URLs per Host may share a path shape |
| CFG-031 | manual_boosts | list | `[]` | `{prefix, boost}` pairs, boost 0–300; multiple matches ⇒ largest single boost [DOC-12 §2] |
| CFG-043 | url_record_retention_days | int | 180 | 0 = keep forever; DEAD/EXCLUDED record retention basis [DOC-11 §6] |

## Operations

| ID | Name | Type | Default | Notes |
|----|------|------|---------|-------|
| CFG-032 | log_level | enum | `info` | debug/info/warn/error |
| CFG-033 | fetch_event_retention_days | int | 7 | ≥0; 0 = keep forever [DOC-11 §6] |
| CFG-034 | health_listen_addr | string | null | optional HTTP endpoint for metrics/health [DOC-15]; also exposes operator actions [DOC-16 §5] |
| CFG-037 | url_blocklist | list | `[]` | glob patterns matched against normalized URLs → ST-190/`BLOCKLIST` [DOC-06 §5] |
| CFG-038 | record_out_of_scope | bool | true | record ST-190 rows for out-of-scope URLs instead of dropping silently [FR-004] (renamed from `log_exclusions`: it governs audit rows, not logging — the exclusions_total metric [DOC-15 §1] is always emitted) |

## Security

| ID | Name | Type | Default | Range / notes |
|----|------|------|---------|---------------|
| CFG-041 | egress_deny_ips | list | `[]` | IPs/CIDRs appended to the SSRF block set [R-400] |
| CFG-042 | egress_allow_private_ranges | bool | false | test-harness escape hatch from [R-400] [R-405]; MUST NOT be enabled in production; startup logs WARN when true |

## Validation rules

- V-1: All ranges above are inclusive; violation ⇒ abort before any I/O except reading the config file itself.
- V-2: `total_transfer_timeout_ms ≥ response_header_timeout_ms` required.
- V-3: Config hash (SHA-256 of canonical serialization) recorded in every `runs` row.
- V-4: If scope_mode=PREFIX_LIST, every seed MUST match ≥1 entry of [CFG-039]; violation aborts startup [DEC-011] (a seed that is out of scope would otherwise silently crawl nothing).
- V-5: [CFG-018] MUST match the UA Token pattern `name/version (+url)` [R-120], and [CFG-019], when set, MUST be a valid email address; violation ⇒ abort per [V-1].
