---
id: DOC-16
title: Security, Safety, and Abuse Prevention
version: 1.0.0
---

# Security and Abuse Prevention

Precedence: this document overrides all others [R-000]. All guards fail closed.

## 1. Threat model (v1)

| Threat | Vector | Mitigation |
|---|---|---|
| SSRF into private networks | malicious page links to internal IPs / DNS rebinding | §2 |
| Resource exhaustion | huge payloads, infinite redirects, link farms, traps | §3 + caps everywhere |
| Crawling against site wishes | ignoring robots/rate limits | fail-closed gates [DOC-08] |
| Legal/privacy exposure | storing credentials in URLs, sensitive paths | §4 |

## 2. Network egress policy

- R-400: Before every TCP connection (initial AND each redirect hop):
  1. Resolve all A/AAAA records.
  2. Every resolved address MUST be global-unicast; reject if any address is:
     loopback, RFC1918, CGNAT 100.64/10, link-local (v4/v6), ULA fc00::/7,
     multicast, 0.0.0.0/8, ::1, ::, documentation ranges, or the IP of any
     configured deny-list entry.
  3. Pin the validated IP for the connection (defeats TOCTOU rebinding between
     check and connect); TLS SNI/cert still checked against hostname.
- R-401: Ports restricted to 80/443 (plus explicit scheme defaults); other ports ⇒ ERR-004.
- R-402: Redirect to a blocked target terminates the chain with ERR-004; the referring URL is marked ST-180/`SSRF_BLOCKED` and the host is flagged `suspicious=true` (operator-visible metric only).
- R-403: Non-resolving hostnames ⇒ ERR-001 permanent if NXDOMAIN.

## 3. Resource caps (all enforced pre-allocation)

| Cap | Value | Rule |
|---|---|---|
| payload size | [CFG-016] | abort mid-stream, discard partial [FR-023] |
| redirects | [CFG-017] | [R-130] |
| header size | 64 KiB fixed | reject beyond |
| URLs per page | 1000 extracted + overflow flag | [DOC-10 §3 outlinks cap] |
| path shapes per host | [CFG-030] | trap guard [DOC-06 §5] |
| total records | [CFG-005] | [FR-005] |
| metadata row size | artifacts JSON ≤ 2 MiB | truncate deterministic fields, flag `truncated` |

- R-310: Decompression bomb defense: decoded size counts toward [CFG-016];
  decoding streams MUST abort as soon as the running byte count exceeds it.

## 4. Content hygiene

- R-320: Never send URL userinfo anywhere [R-002]; strip credentials before logging or storage. (Rule IDs in this document use the R-4xx block; they never collide with other documents' R-nnn IDs.)
- R-330: Do not special-case crawl-sensitive paths (`/admin`, `.git`, etc.) — scope and robots govern access; but robots DISALLOW is always final [P-2].
- R-340: No execution of content: no JS evaluation, no macro/document conversion — parsing is structural only [DOC-10].

## 5. Operator API surface

Runtime API (local only): inject seeds [FR-006], reset DEAD URL [DOC-13 §4],
trigger graceful drain. MUST require no network exposure by default; if
[health_listen_addr] is set it binds read-only endpoints plus these actions,
and actions are logged with operator identity when available.
