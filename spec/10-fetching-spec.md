---
id: DOC-09
title: Fetching Specification (HTTP Behavior)
version: 1.1.0
---

# Fetching

## 1. Request construction

| Aspect | Rule |
|---|---|
| Method | GET only. |
| HTTP versions | HTTP/1.1 mandatory; HTTP/2 optional via ALPN; no h2c upgrade. |
| Headers | `User-Agent` = [CFG-018] exact; `Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`; `Accept-Encoding: gzip, deflate, br`; `From` iff [CFG-019]; conditional validators on refetches [R-121]. |
| Forbidden | Cookies, Authorization, custom fingerprint headers, referer spoofing. |
| Body | None. |

- R-160: The crawler MUST be stateless with respect to site state: it MUST NOT store cookies, MUST NOT transmit any cookie received from any site, and MUST ignore `Set-Cookie` headers entirely.

## 2. Timeouts (all independent, all abortive)

| Timer | Config | Starts | Notes |
|---|---|---|---|
| DNS | [CFG-036] | resolution start | includes SSRF check time |
| Connect | [CFG-012] | connect start | TCP established |
| TLS | [CFG-013] | handshake start | certificate verified, fail closed |
| Response headers | [CFG-014] | request bytes written | until full header block received |
| Total transfer | [CFG-015] | request start | covers redirects? NO — resets per hop |

Timeout violations classify ERR-002 (connect), ERR-003 (TLS), ERR-012 (headers/total).

## 3. Redirects

- R-130: Follow 301, 302, 303, 307, 308 up to [CFG-017] hops; method stays GET throughout.
- R-131: Each hop: resolve DNS + SSRF check [DOC-16 §2], scope check [R-030], robots check [FR-021].
- R-132: Redirect loop detection: if any hop URL identity repeats within the chain ⇒ stop, ERR-011.
- R-133: The final hop's URL is recorded as final_url_identity; the original identity keeps its record, linked via `redirect_chain` (ordered list of identities), persisted as JSON on the attempt's `fetch_events` row [DOC-11 §1].

## 4. Response classification

| Status | Class | Action |
|---|---|---|
| 200–299 | success | store payload [FR-043] → ST-130 |
| 304 | success-unchanged | refresh freshness metadata, keep old payload hash |
| 401, 403 | permanent | ST-180/`HTTP_4xx_AUTH`; do NOT retry |
| 404, 410 | permanent | ST-180/`GONE` |
| 418, 451 | permanent | ST-180 |
| 429 | retryable | honor Retry-After [R-111] → ST-150 |
| other 4xx | permanent | ST-180/`HTTP_4xx_OTHER` |
| 5xx | retryable | → ST-150, increments host consecutive_failures |
| 3xx non-followable (loop/cap/out-of-scope) | per cause | ERR-011 or REDIRECT_OUT_OF_SCOPE |

## 5. Payload handling

- R-140: Decode Content-Encoding before hashing [FR-024]; unknown encodings ⇒ ERR-013 (retryable once, then permanent).
- R-141: Content-Length vs actual bytes mismatch ⇒ trust actual bytes, log anomaly metric.
- R-142: Charset detection order: HTTP `charset` param → BOM → HTML meta charset in first 1024 bytes → default UTF-8 (strict errors ⇒ replacement char, never fatal).
- R-143: Non-text types: sniff Content-Type only; payload stored iff [CFG-028] and type allowed list = {`image/*`, `application/pdf`, `text/plain`, `application/xml`, `application/rss+xml`, `application/atom+xml`}.

## 6. FetchResult contract

Every attempt returns exactly:

```json
{
  "url_identity": "...",
  "outcome": "SUCCESS|UNCHANGED|RETRYABLE|PERMANENT",
  "error_class": "ERR-nnn|null",
  "http_status": 200,
  "final_url_identity": "...",
  "redirect_chain": ["..."],
  "payload_sha256": "...|null",
  "payload_size": 12345,
  "content_type": "text/html; charset=utf-8",
  "validators": {"etag": "...", "last_modified": "..."},
  "timings_ms": {"dns":12,"connect":8,"tls":15,"ttfb":40,"total":180},
  "attempt": 2
}
```

Scheduler consumes outcomes per [DOC-13].
