---
id: DOC-10
title: Parsing and Extraction
version: 1.11.0
---

# Parsing and Extraction

Applies to `text/html` and `application/xhtml+xml` payloads [FR-040].

## 1. Parsing

- R-150: Parse with an HTML5-conformant parser (error-recovering, per WHATWG HTML). Parse errors MUST NOT abort extraction.
- R-151: XHTML served as `application/xhtml+xml` MAY be parsed with an XML parser; on XML failure, fall back to the HTML5 parser.
- R-152: Scripts and styles are not executed or fetched; `<noscript>` content is parsed.

## 2. Link extraction (discovery)

Candidate sources, in order of processing:

| Source | Attribute | Notes |
|---|---|---|
| anchors | `href` of `<a>` | primary discovery |
| area | `href` of `<area>` | image-map links; discovered like anchors |
| images | `src`, `srcset` entries | discovered like anchors (same filters; typically stored-not-parsed under [R-143]) |
| iframes | `src` | treated as pages |
| link elements | `href` where rel ∈ {canonical, alternate} | canonical → metadata; alternate → discovery |
| meta refresh | `content="N; url=..."` | treat as redirect hint (discovery + metadata) |

Rules:

- R-153: Resolve each candidate against base URL ([R-020]/[R-021]), normalize, then apply scope/trap filters identically to seeds.
- R-154: Deduplicate candidates within one page before ingestion.
- R-155: `rel=nofollow` on an anchor ⇒ that candidate is NOT ingested. Page-level `nofollow` ([FR-045]) suppresses all anchor-derived candidates from that page.
- R-156: Extracted links record anchor text (first 256 chars, whitespace-collapsed) as metadata on the discovery edge.

## 3. Content artifacts (per page)

Stored in the Metadata Store as JSON documents keyed by
`(payload_sha256, final_url_identity)` [DOC-11 §1] — artifacts are a function
of the payload bytes AND the link-resolution base ([R-020]/[R-021]):
byte-identical payloads fetched under different final URLs (mirrors) share a
blob [FR-043] but hold distinct artifacts when their resolved outlinks differ:

| Artifact | Definition |
|---|---|
| title | text of first `<title>` in head, trimmed |
| description | `meta[name=description]` content |
| canonical_url | normalized `link[rel=canonical]` href (informational only — does NOT change URL Identity [DEC-004]) |
| meta_refresh | first `http-equiv=refresh` target URL (normalized) and delay, if present |
| lang | `lang` attribute of `<html>`, lowercased |
| meta_robots | union of directives from all `meta[name=robots]` content values — tokens are split on commas and whitespace; `none` ≡ `noindex,nofollow`; unknown tokens are ignored (only `noindex` and `nofollow` are recognized, and they drive [FR-045]) |
| headings | ordered list of `{level, text}` for h1–h3 (capped at 1000 entries; overflow sets `truncated`) |
| main_text | text content of `<body>` after removing script/style/nav/footer/aside/template/noscript, whitespace-normalized, capped at 1 MiB characters |
| word_count | count of whitespace-separated tokens in `main_text` |
| outlinks | list of `{url_identity, anchor_text, nofollow}` (capped 1000/page; overflow flagged) |
| truncated | `true` iff any per-page cap (headings, outlinks, main_text, artifacts JSON size [DOC-16 §3]) was applied |

`outlinks` records all extracted anchor candidates — including `nofollow`
ones, flagged — regardless of ingestion [R-155]; a page-level `nofollow` page
[FR-045] stores its full flagged outlink list while ingesting none.

Serialization cap: when the serialized artifacts JSON would exceed the
2 MiB row cap [DOC-16 §3], fields are tail-truncated in a fixed order —
`headings` first, then `main_text`, then `outlinks` (each dropping from its
end) — until the document fits, and `truncated=true`. The order and the
tail-truncation are normative so that [R-157]'s byte-identical guarantee
extends to capped pages.

- R-157: Extraction is deterministic: same input bytes and same resolution base (final URL) ⇒ byte-identical artifact JSON (stable key ordering).
- R-158: Pages > [CFG-016] never reach parsing [FR-023].

## 4. Failure handling

Parse failures (should be rare given error recovery) mark the page
`parse_ok=false` with reason string (error class ERR-009 plus detail;
recorded on the `pages` row only — never a FetchResult error_class, since
parsing is post-fetch); payload is retained; state still advances
ST-130→ST-140 (extraction "complete" includes failed-parse recording).
