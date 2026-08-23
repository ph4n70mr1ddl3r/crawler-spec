---
id: DOC-10
title: Parsing and Extraction
version: 1.0.0
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
| images | `src`, `srcset` entries | stored as discovered URLs, type=image |
| iframes | `src` | treated as pages |
| link elements | `href` where rel ∈ {canonical, alternate} | canonical → metadata; alternate → discovery |
| meta refresh | `content="N; url=..."` | treat as redirect hint (discovery + metadata) |

Rules:

- R-153: Resolve each candidate against base URL ([R-020]/[R-021]), normalize, then apply scope/trap filters identically to seeds.
- R-154: Deduplicate candidates within one page before ingestion.
- R-155: `rel=nofollow` on an anchor ⇒ that candidate is NOT ingested. Page-level `nofollow` ([FR-045]) suppresses all anchor-derived candidates from that page.
- R-156: Extracted links record anchor text (first 256 chars, whitespace-collapsed) as metadata on the discovery edge.

## 3. Content artifacts (per page)

Stored in Metadata Store as JSON document keyed by payload hash:

| Artifact | Definition |
|---|---|
| title | text of first `<title>` in head, trimmed |
| description | `meta[name=description]` content |
| canonical_url | normalized `link[rel=canonical]` href (informational only — does NOT change URL Identity [DEC-004]) |
| lang | `lang` attribute of `<html>`, lowercased |
| meta_robots | union value(s) of `meta[name=robots]` |
| headings | ordered list of `{level, text}` for h1–h3 |
| main_text | text content of `<body>` after removing script/style/nav/footer/aside/template/noscript, whitespace-normalized, capped at 1 MiB characters |
| word_count | token count of main_text |
| outlinks | list of `{url_identity, anchor_text, nofollow}` (capped 1000/page; overflow flagged) |

- R-157: Extraction is deterministic: same input bytes ⇒ byte-identical artifact JSON (stable key ordering).
- R-158: Pages > [CFG-016] never reach parsing [FR-023].

## 4. Failure handling

Parse failures (should be rare given error recovery) mark the page
`parse_ok=false` with reason string; payload is retained; state still advances
ST-130→ST-140 (extraction "complete" includes failed-parse recording).
