---
id: DOC-00
title: Glossary and Controlled Vocabulary
version: 1.0.0
---

# Glossary

All specification documents MUST use these terms with exactly these meanings.
If a term needs a different meaning in context, the document MUST define a
qualified term (e.g., "fetch-timeout" vs "total-timeout").

## Core entities

| Term | Definition |
|------|------------|
| **Crawler** | The overall system specified by this KB. Also: the running process instance. |
| **URL** | An absolute URI with scheme `http` or `https` ([RFC 3986]). |
| **Seed URL** | A URL injected at start or runtime as a crawl origin. |
| **URL Record** | The persistent database row representing one unique URL identity and its crawl state. See [DOC-07]. |
| **URL Identity** | The normalized form of a URL used as its primary key. Defined in [DOC-06]. |
| **Frontier** | The ordered collection of discovered URLs pending fetch. |
| **Host** | A `(scheme, hostname, port)` triple. |
| **Registrable Domain** | The effective top-level domain plus one label (eTLD+1), computed with the Public Suffix List. |
| **Page** | A fetched, successfully transferred response body identified by `(final URL identity, SHA-256 of body bytes)`. |
| **Fetch** | One attempt to retrieve the representation of a URL over HTTP(S). |
| **Crawl Session** | A period during which the crawler runs continuously; may span process restarts. |

## Processing stages

| Term | Definition |
|------|------------|
| **Discovery** | Producing new candidate URLs by extracting links from fetched pages or accepting seeds. |
| **Normalization** | Deterministically rewriting a URL string into canonical form per [DOC-06]. |
| **Filtering (exclusion)** | Deciding a URL must never be crawled (scope, robots, blocklist, traps). |
| **Scheduling** | Choosing which queued URL to fetch next and when, per [DOC-13]. |
| **Fetching** | Performing the HTTP exchange per [DOC-10]. |
| **Parsing** | Decoding and structurally analyzing a fetched body per [DOC-11]. |
| **Extraction** | Deriving outbound links and content artifacts from a parsed page. |
| **Deduplication** | Detecting byte-identical bodies already stored, keyed by SHA-256. |

## Policies and limits

| Term | Definition |
|------|------------|
| **Politeness Delay** | Minimum interval between two consecutive fetches to the same Host. |
| **Effective Delay (per host)** | `max(politeness_delay_cfg, robots crawl_delay, dynamic_backoff)` for that Host. |
| **Global Concurrency** | Max simultaneous in-flight fetches across all Hosts. |
| **Per-Host Concurrency** | Max simultaneous in-flight fetches to one Host. MUST be ≥1. |
| **Retry Budget** | Max fetch attempts per URL identity (initial attempt included). |
| **Backoff** | Exponentially increasing delay applied after failed fetches to a Host. |
| **Trap** | URL pattern generating unbounded distinct URLs within one site (e.g., calendar pagination, session IDs in paths). |
| **Crawl Scope** | The predicate deciding whether a URL is in scope. Default: URL's Registrable Domain ∈ set of seed Registrable Domains. |

## Storage

| Term | Definition |
|------|------------|
| **Metadata Store** | Persistent store of URL records, crawl state, host state, run history. |
| **Content Store** | Content-addressed blob store; key = SHA-256(body bytes). Immutable once written. |
| **Payload** | Raw response body bytes exactly as received (post content-encoding decode). |

## Miscellaneous

| Term | Definition |
|------|------------|
| **UA Token** | The crawler's product token sent in `User-Agent`, e.g. `SpecBot/1.0 (+https://example.com/bot)`. |
| **Robots Group** | A `User-agent:` group in robots.txt applicable to UA Token, per [RFC 9309]. |
| **Run** | One execution of the crawl job, identified by monotonic `run_id`. Survives restart as a record. |

[RFC 3986]: https://www.rfc-editor.org/rfc/rfc3986
[RFC 9309]: https://www.rfc-editor.org/rfc/rfc9309
