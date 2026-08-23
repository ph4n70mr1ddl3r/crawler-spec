---
id: DOC-01
title: Scope, Goals, Non-Goals
version: 1.0.0
---

# Scope and Goals

## Purpose

A general-purpose, polite, resumable web crawler that:

1. Starts from a set of Seed URLs.
2. Systematically discovers and fetches pages within the configured Crawl Scope.
3. Stores page payloads and extracted metadata for later consumption by
   search/indexing or analysis systems (the **Downstream Consumer**, an
   out-of-scope system reading the Content Store and Metadata Store).

## Stakeholders

| Stakeholder | Interest |
|---|---|
| Operator | Configures, starts/stops, monitors the crawler; reads metrics/logs. |
| Downstream Consumer | Reads stored pages/metadata via documented interfaces [DOC-11]. |
| Target site owners | Must not be disturbed: robots.txt honored, bounded load [DOC-08]. |
| Implementing engineer / AI agent | Needs unambiguous, testable requirements. |

## Goals (v1)

- G-1: Correctness over speed; every rule below is verifiable.
- G-2: Crash-safe and resumable: killing the process at any instant never loses
  discovered URLs nor causes policy violations on restart (see NFR-010..NFR-012).
- G-3: Politeness is guaranteed, not best-effort: hard caps enforced in code paths,
  not advisory ([DOC-08]).
- G-4: Bounded resource use under all inputs, including adversarial ones
  (traps, huge files, infinite redirects) [DOC-16].
- G-5: Deterministic re-runs: identical config + identical web content ⇒ identical
  stored state modulo timestamps.

## In scope (v1)

- HTTP/1.1 and HTTP/2 over TLS; `http` and `https` schemes only.
- HTML/XHTML parsing, link extraction, basic content extraction [DOC-10].
- robots.txt (RFC 9309) fetching, caching, enforcement [DOC-08].
- URL normalization, scope filtering, trap mitigation [DOC-06], [DOC-16].
- Content-addressed storage, exact dedup, retention [DOC-11].
- Recrawl/refresh scheduling [DOC-12].
- Metrics, structured logs, health endpoint [DOC-15].
- Configuration validation [DOC-14].

## Out of scope (v1, explicit non-goals)

- NG-1: JavaScript execution / headless-browser rendering.
- NG-2: Near-duplicate detection, ML-based content understanding.
- NG-3: Distributed multi-node crawling.
- NG-4: Authenticated crawling (cookies, logins); v1 sends no cookies received
  from sites back to any site (see R-160).
- NG-5: Media analysis beyond storing bytes of allowed types.
- NG-6: Building a search index or ranker (Downstream Consumer's job).
- NG-7: Sitemap-driven discovery (reserved extension point EXT-SITEMAP; MAY be
  added in a minor revision without changing any requirement in this KB).

## Success criteria

The crawler v1 is complete when every AC-### criterion in [DOC-17] passes.
