---
kb: web-crawler-spec
version: 1.16.0
status: APPROVED-DRAFT
last-updated: 2026-08-25
purpose: Complete, implementation-ready specification of a web crawler. No code. For human review and AI-assisted implementation.
---

# Web Crawler Specification Knowledge Base

This directory is the **single source of truth** for the specification of a web
crawler system. It contains no implementation. Every behavioral statement is
expressed as a numbered, testable requirement or rule.

## Document map

| ID   | File                                          | Contents |
|------|-----------------------------------------------|----------|
| DOC-00 | [01-glossary.md](01-glossary.md)            | Controlled vocabulary. All other docs MUST use these terms exactly. |
| DOC-01 | [02-scope-and-goals.md](02-scope-and-goals.md) | Purpose, in/out of scope, goals, non-goals, stakeholders. |
| DOC-02 | [03-design-decisions.md](03-design-decisions.md) | Binding architectural decisions and their rationale. |
| DOC-03 | [04-architecture.md](04-architecture.md)     | Components, responsibilities, data flow, deployment model. |
| DOC-04 | [05-functional-requirements.md](05-functional-requirements.md) | Functional requirements (FR-###). |
| DOC-05 | [06-nonfunctional-requirements.md](06-nonfunctional-requirements.md) | Quality attributes (NFR-###). |
| DOC-06 | [07-url-model-and-normalization.md](07-url-model-and-normalization.md) | URL parsing, normalization, identity, filtering rules. |
| DOC-07 | [08-url-lifecycle-state-machine.md](08-url-lifecycle-state-machine.md) | States (ST-*), transitions, invariants. |
| DOC-08 | [09-politeness-and-robots.md](09-politeness-and-robots.md) | robots.txt handling, rate limiting, crawl etiquette. |
| DOC-09 | [10-fetching-spec.md](10-fetching-spec.md)    | HTTP client behavior, redirects, timeouts, encoding. |
| DOC-10 | [11-parsing-and-extraction.md](11-parsing-and-extraction.md) | Link extraction, content extraction, metadata. |
| DOC-11 | [12-storage-model.md](12-storage-model.md)    | Data stores, schemas, retention, integrity. |
| DOC-12 | [13-scheduling-and-recrawl.md](13-scheduling-and-recrawl.md) | Frontier priority, politeness windows, refresh policy. |
| DOC-13 | [14-error-taxonomy-and-retry.md](14-error-taxonomy-and-retry.md) | Error classes (ERR-*), retry policy, dead-lettering. |
| DOC-14 | [15-configuration.md](15-configuration.md)    | All tunable parameters (CFG-*), defaults, valid ranges. |
| DOC-15 | [16-observability.md](16-observability.md)    | Metrics, logs, health signals. |
| DOC-16 | [17-security-and-abuse-prevention.md](17-security-and-abuse-prevention.md) | SSRF defense, trap avoidance, resource caps. |
| DOC-17 | [18-acceptance-criteria.md](18-acceptance-criteria.md) | Verifiable acceptance criteria (AC-###). |
| DOC-18 | [19-changelog.md](19-changelog.md) | KB revision history. |

## Reading order

1. DOC-00 (glossary) → DOC-01 (scope) → DOC-02 (decisions) → DOC-03 (architecture)
2. Then any topic document; each is self-contained via cross-references.

## Conventions used throughout

- **Requirement IDs**: `FR-nnn` functional, `NFR-nnn` non-functional,
  `AC-nnn` acceptance criterion, `CFG-nnn` configuration parameter,
  `ERR-nnn` error class, `ST-nnn` state, `DEC-nnn` decision, `R-nnn` normative rule.
- **Auxiliary ID families** (each owned by the document that defines it):
  `G-n` goals and `NG-n` non-goals ([DOC-01]); `INV-n` data-flow invariants
  ([DOC-03]); component aliases `C1–C9` ([DOC-03]); `P-n` politeness
  principles ([DOC-08]); `T-n` storage transactions ([DOC-11]);
  `V-n` configuration validation rules ([DOC-14]); extension points
  `EXT-*` ([DOC-01]).
- **Keywords** (RFC 2119): MUST / MUST NOT / SHOULD / SHOULD NOT / MAY.
- **Normative rules** (`R-nnn`) appear inside topic documents and carry the same
  weight as FR/NFR entries. R-nnn IDs are globally unique across the entire KB;
  topic documents own disjoint ID blocks (DOC-11 storage rules own R-5xx;
  DOC-16 security rules own R-3xx and R-4xx).
- Cross-references are written as `[FR-012]`, `[DOC-08]`, `[CFG-007]`.
- Conflicts: a specific topic document overrides general documents;
  security rules ([DOC-16]) override everything else; this precedence order is
  itself normative (see [R-000]).

[R-000]: When two statements conflict, resolve in this order:
(1) DOC-16 security rules, (2) DOC-08 politeness rules,
(3) numbered FR/NFR requirements, (4) prose explanation.
Within a level, the more specific document wins: a topic document
([DOC-06]–[DOC-17]) overrides a general one ([DOC-01]–[DOC-05]) at the
same level. R-nnn rules rank with FR/NFR at level (3), except the blocks
already claimed above: security rules R-3xx/R-4xx ([DOC-16]) at (1) and
[DOC-08]'s politeness rules at (2).
