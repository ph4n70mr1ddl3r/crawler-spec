---
id: DOC-06
title: URL Model, Normalization, Identity, Filtering
version: 1.8.0
---

# URL Model and Normalization

## 1. Acceptance

- R-001: Only absolute URLs with scheme ∈ {`http`, `https`} [CFG-003] are acceptable. All others (relative without base, `ftp:`, `javascript:`, `data:`, `mailto:`) MUST be discarded at discovery time with no record.
- R-002: URLs containing userinfo (`user:pass@host`) MUST be rejected entirely [NFR-016]: at discovery time they are discarded with no record, identically to [R-001]; credentials MUST never be logged or stored [R-320].
- R-003: IDN hostnames MUST be converted to punycode (IDNA2008/UTS-46) before any use; the original unicode form MAY be kept as display metadata only. IP-literal hostnames are lowercased, and IPv6 literals are canonicalized to their RFC 5952 compressed form, so `[2001:0DB8::1]` and `[2001:db8::1]` share one URL Identity. The Registrable Domain of an IP-literal Host is the canonical literal string itself [DOC-00].

## 2. Normalization algorithm (normative)

Given an accepted URL, produce the **Normalized URL** by applying, in order:

1. Parse per RFC 3986; percent-normalize the path and query components per
   RFC 3986 §6.2.2: (a) uppercase the two hex digits of every `%XX`;
   (b) decode `%XX` only when it encodes a character in the unreserved set
   ALPHA / DIGIT / `-._~`; (c) leave percent-encoded reserved octets (e.g.
   `%2F`, `%3F`, `%26`) encoded and leave literal reserved delimiters
   (`/`, `=`, `&`, …) as they appear. URLs that fail to parse (e.g. a stray
   `%`) are discarded at discovery time identically to [R-001]; as seeds they
   abort startup [FR-002].
2. Lowercase scheme.
3. Lowercase hostname (after punycode conversion).
4. Remove port if it equals the scheme default (80 for http, 443 for https).
5. Remove fragment (`#...`) — fragments never participate in URL Identity.
6. Resolve path dot-segments per RFC 3986 §5.2.4.
7. Empty path ⇒ `/`.
8. Query parameters MUST NOT be sorted and MUST NOT be deduplicated: the
   original parameter order is preserved verbatim (order can be semantically
   significant).
9. Consecutive slashes in the path MUST NOT be collapsed (they may denote
   distinct resources); the path is preserved verbatim.

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
| IN_SCOPE | scope_mode=PREFIX_LIST: URL's (scheme, host, path) matches ≥1 entry of [CFG-039]: identical scheme+host — both compared case-insensitively (candidate identities are lowercase after §2; entry scheme/host are lowercased before comparison) — and a path equal to the entry's path or beginning with it followed by `/` (segment-boundary prefix; the path itself compares verbatim, case-sensitively; query string ignored). Entry paths are used verbatim: an entry with an empty path matches every path on its host, while an entry whose path is `/` matches only the root path |
| OUT_OF_SCOPE | otherwise |

- R-030: Redirect targets are subject to the identical predicate. A redirect leaving scope terminates the chain at that hop with outcome PERMANENT and error_class ERR-015; no fetch of the target occurs.
- R-031: The Scope predicate is evaluated exactly once per new identity, at ingestion; the verdict is reflected in the record's state (ST-190/`OUT_OF_SCOPE` vs. queued) and is never re-evaluated for that identity. Redirect targets are evaluated per hop [R-030].

## 5. Trap mitigation (pre-scheduling filters)

Applied to IN_SCOPE URLs in order; first match wins:

1. **URL blocklist**: URL matches any [CFG-037] glob pattern → ST-190/`BLOCKLIST`.
2. **Param blocklist**: exclude URLs whose query contains a parameter name matching [CFG-029] patterns → ST-190/`TRAP_PARAM`.
3. **Path budget**: per Host + path shape (each maximal digit run in the normalized path replaced by a single `N`; query string excluded), if the number of existing URL Records on that Host sharing the shape — each identity counted once, any state [R-042]; the candidate itself is not yet a record, since filters run before insertion [R-040] — is ≥ [CFG-030] → ST-190/`TRAP_PATH_BUDGET` (at most [CFG-030] URLs may share one shape per Host).
4. **Depth**: depth > [CFG-004] → ST-190/`DEPTH_LIMIT`. Seeds have depth 0.

- R-040: Filters MUST run before Frontier insertion so traps never consume fetch capacity.
- R-041: Excluded URLs keep records (state ST-190) so re-discovery is O(1) and
  auditable — except OUT_OF_SCOPE URLs when [CFG-038]=false, which are dropped
  with no record per [FR-004].
- R-042: Path-shape counters (filter 3) are maintained in memory and rebuilt
  at startup from URL Records via the `host_key` index [DOC-11 §1]; every
  record on the Host — including excluded ones — contributes its shape, so
  the rebuild is deterministic and re-discovery stays O(1).
