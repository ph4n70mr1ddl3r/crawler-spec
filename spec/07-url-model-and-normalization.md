---
id: DOC-06
title: URL Model, Normalization, Identity, Filtering
version: 1.1.0
---

# URL Model and Normalization

## 1. Acceptance

- R-001: Only absolute URLs with scheme ∈ {`http`, `https`} [CFG-003] are acceptable. All others (relative without base, `ftp:`, `javascript:`, `data:`, `mailto:`) MUST be discarded at discovery time with no record.
- R-002: URLs containing userinfo (`user:pass@host`) MUST be rejected entirely [NFR-016]: at discovery time they are discarded with no record, identically to [R-001]; credentials MUST never be logged or stored [R-320].
- R-003: IDN hostnames MUST be converted to punycode (IDNA2008/UTS-46) before any use; the original unicode form MAY be kept as display metadata only.

## 2. Normalization algorithm (normative)

Given an accepted URL, produce the **Normalized URL** by applying, in order:

1. Parse per RFC 3986; percent-decode then re-encode path and query components using unreserved set = ALPHA / DIGIT / `-._~`, uppercase hex.
2. Lowercase scheme.
3. Lowercase hostname (after punycode conversion).
4. Remove port if it equals the scheme default (80 for http, 443 for https).
5. Remove fragment (`#...`) — fragments never participate in URL Identity.
6. Resolve path dot-segments per RFC 3986 §5.2.4.
7. Empty path ⇒ `/`.
8. Sort query parameters? NO — preserve original parameter order (order can be semantically significant). Deduplicate nothing.
9. Collapse multiple consecutive slashes in the path? NO — preserved verbatim (they may be distinct resources).

The output string is the **URL Identity** (primary key, DEC-004).

- R-010: Normalization MUST be a pure function: same input string ⇒ same identity, always.
- R-011: Two raw URLs normalize equal ⇒ same resource for all crawl purposes.

## 3. Cross-host resolution

- R-020: Links found in a page MUST be resolved against the **final URL** of that page's fetch chain (after redirects), not the originally requested identity.
- R-021: `<base href>` in HTML overrides the final URL as resolution base when present and valid.

## 4. Scope predicate

Evaluated after normalization, before enqueueing. Exactly one of:

| Verdict | Condition |
|---|---|
| IN_SCOPE | scope_mode=SEED_DOMAINS: URL's Registrable Domain ∈ seed Registrable Domains set |
| IN_SCOPE | scope_mode=SEED_HOSTS: URL's Host ∈ seed Hosts set |
| IN_SCOPE | scope_mode=PREFIX_LIST: URL starts with ≥1 entry of [CFG-003b] (scheme+host+path prefix match) |
| OUT_OF_SCOPE | otherwise |

- R-030: Redirect targets are subject to the identical predicate. A redirect leaving scope terminates the chain at that hop with outcome `REDIRECT_OUT_OF_SCOPE`; no fetch of the target occurs.
- R-031: The Scope predicate is evaluated once per new identity; verdict cached on the URL Record.

## 5. Trap mitigation (pre-scheduling filters)

Applied to IN_SCOPE URLs in order; first match wins:

1. **URL blocklist**: URL matches any [CFG-037] glob pattern → ST-190/`BLOCKLIST`.
2. **Param blocklist**: drop URLs whose query contains a parameter name matching [CFG-029] patterns → ST-190/`TRAP_PARAM`.
3. **Path budget**: per Host + normalized-path-shape (digits replaced by `N`), if shape count > [CFG-030] → ST-190/`TRAP_PATH_BUDGET`.
4. **Depth**: depth > [CFG-004] → ST-190/`DEPTH_LIMIT`. Seeds have depth 0.

- R-040: Filters MUST run before Frontier insertion so traps never consume fetch capacity.
- R-041: Excluded URLs keep records (state ST-190) so re-discovery is O(1) and auditable.
