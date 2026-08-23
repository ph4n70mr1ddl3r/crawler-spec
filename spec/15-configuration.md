---
id: DOC-14
title: Configuration Reference
version: 1.0.0
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
| CFG-003b | scope_prefix_list | list | `[]` | required iff scope_mode=PREFIX_LIST |
| CFG-004 | max_depth | int | 5 | 0–50 |
| CFG-005 | max_urls_total | int | 100000 | >0 |
| CFG-006 | max_pages_per_host | int | 10000 | per Registrable Domain |

## Politeness & concurrency

| ID | Name | Type | Default | Range |
|----|------|------|---------|-------|
| CFG-007 | politeness_delay_ms | int | 5000 | 0–600000 |
| CFG-008 | robots_cache_ttl_s | int | 3600 | 60–86400 |
| CFG-009 | per_host_concurrency | int | 2 | 1–16 |
| CFG-010 | global_concurrency | int | 64 | 1–1024 |
| CFG-011 | dynamic_backoff_enabled | bool | true | |

## Fetching

| ID | Name | Type | Default | Range |
|----|------|------|---------|-------|
| CFG-012 | connect_timeout_ms | int | 10000 | 1000–60000 |
| CFG-013 | tls_handshake_timeout_ms | int | 10000 | 1000–60000 |
| CFG-014 | response_header_timeout_ms | int | 30000 | 1000–120000 |
| CFG-015 | total_transfer_timeout_ms | int | 300000 | ≥ header timeout |
| CFG-016 | max_payload_bytes | int | 5242880 (5 MiB) | ≤ 20971520 hard cap |
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
| CFG-030 | max_similar_paths_per_host | int | 500 | trap guard [DOC-06 §5] |
| CFG-031 | manual_boosts | list | `[]` | `{prefix, boost}` pairs, boost 0–300 |

## Operations

| ID | Name | Type | Default | Notes |
|----|------|------|---------|-------|
| CFG-032 | log_level | enum | `info` | debug/info/warn/error |
| CFG-033 | fetch_event_retention_days | int | 7 | [DOC-11 §6] |
| CFG-034 | health_listen_addr | string | null | optional HTTP endpoint for metrics/health [DOC-15]; also exposes operator actions [DOC-16 §5] |

## Validation rules

- V-1: All ranges above are inclusive; violation ⇒ abort before any I/O except reading the config file itself.
- V-2: `total_transfer_timeout_ms ≥ response_header_timeout_ms` required.
- V-3: Config hash (SHA-256 of canonical serialization) recorded in every `runs` row.
