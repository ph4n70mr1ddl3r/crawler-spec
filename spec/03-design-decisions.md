---
id: DOC-02
title: Binding Design Decisions
version: 1.0.0
---

# Design Decisions

Each decision is binding unless explicitly revised. Revision requires updating
this document and bumping the KB version.

| ID | Decision | Rationale / Consequences |
|----|----------|--------------------------|
| DEC-001 | The crawler is specified behaviorally; implementation language and frameworks are free choices, provided all FR/NFR/R rules are met. | Keeps spec durable. Implementations must map each requirement to code/tests. |
| DEC-002 | Single-node process, modular components communicating through well-defined internal interfaces (queue + store contracts). No cross-network RPC between components in v1. | Simplicity now; interfaces are shaped so Scheduler/Fetcher/Extractor can be split into services later without changing persisted formats. |
| DEC-003 | All persistent state lives in exactly two stores: Metadata Store (relational, SQLite-class acceptable) and Content Store (content-addressed blobs on disk). No other durable state. | Guarantees restart safety [G-2]; makes backup trivial (two directories). |
| DEC-004 | URL Identity = normalized URL string per [DOC-06]. It is the sole primary key for scheduling. Redirect targets create their own URL Records linked to the source. | Deterministic dedup of URLs independent of fetch outcomes. |
| DEC-005 | Content dedup is exact-byte (SHA-256), never fuzzy. Near-duplicate detection is out of scope. | Unambiguous, cheap, testable. |
| DEC-006 | v1 fetches server-rendered HTML only; non-HTML types are stored (if under caps) but parsed only for metadata headers, except `text/html` and `application/xhtml+xml` which get full parsing. | Bounded complexity; clear extension point for JS rendering later. |
| DEC-007 | robots.txt is authoritative per Host. Unreachable/broken robots.txt ⇒ temporarily treat entire Host as disallowed and retry robots.txt with backoff (see [DOC-08 §4]). | Conservative, defensible etiquette. |
| DEC-008 | Scheduling uses per-host virtual clocks and a global due-queue; no wall-clock timers per URL. | Enables deterministic simulation and testing. |
| DEC-009 | Depth limit counts redirect hops toward the same budget as navigation depth? — NO: redirect hops do NOT count toward depth; only extraction depth (link distance from seeds) counts. | Prevents surprising interactions between redirect chains and scope rules. |
| DEC-010 | The crawler identifies itself honestly: fixed UA Token, no rotation of fingerprints, honors `From` if configured. | Ethical baseline; simplifies webmaster contact. |
| DEC-011 | Configuration is a versioned, validated document (JSON/YAML) conforming to [DOC-15]; invalid config aborts startup before any network activity. | Fail-fast; no partial-crawl surprises. |
| DEC-012 | Time inside the scheduler is monotonic; wall-clock time appears only in records/logs. | Restart-safe delays; immune to clock jumps. |
