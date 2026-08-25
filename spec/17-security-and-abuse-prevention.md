---
id: DOC-16
title: Security, Safety, and Abuse Prevention
version: 1.16.0
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
     multicast, 0.0.0.0/8, ::1, ::, documentation ranges, or any entry of
     [CFG-041].
  3. Pin the validated IP for the connection (defeats TOCTOU rebinding between
     check and connect); TLS SNI/cert still checked against hostname. If
     several validated addresses remain, the selection is deterministic and
     uses a fixed total order — sorted ascending, IPv4 before IPv6, numeric
     order within each family — preserving replay determinism [NFR-006].
- R-401: Ports restricted to the scheme defaults (80 for `http`, 443 for `https`); any other port ⇒ ERR-004.
- R-402: Redirect to a blocked target terminates the chain with ERR-004; the referring URL is marked ST-180 with error_class ERR-004 and the referring URL's Host is flagged `suspicious=true` (operator-visible metric only).
- R-403: Non-resolving hostnames ⇒ ERR-001 permanent if NXDOMAIN.
- R-405: [CFG-042]=true relaxes R-400.2 (loopback/private ranges permitted)
  and R-401 (non-default ports permitted — the fixture server need not
  bind the privileged port 80/443);
  it exists solely so the acceptance-test fixture server [DOC-17] is
  reachable, defaults to false, MUST log a WARN at startup when enabled, and
  MUST NOT be enabled in production deployments.
- R-406: [CFG-034] defaults to null (no listener). When it is set to a
  non-loopback address, the mutating operator actions of §5 (seed injection,
  DEAD reset, drain trigger) MUST be disabled — only the read-only `/healthz`
  and `/metrics` endpoints are served — and startup MUST log a WARN. Mutating
  actions are reachable over the network only in violation of this rule
  (fail closed [NFR-013]).

## 3. Resource caps (all enforced pre-allocation)

| Cap | Value | Rule |
|---|---|---|
| payload size | [CFG-016] | abort mid-stream, discard partial [FR-023] |
| redirects | [CFG-017] | [R-130] |
| header size | 64 KiB fixed | reject beyond, ERR-016 |
| URLs per page | 1000 extracted + overflow flag | ingestion cap [DOC-10 §2 R-159]; recorded outlink list likewise capped [DOC-10 §3] |
| path shapes per host | [CFG-030] | trap guard [DOC-06 §5] |
| total records | [CFG-005] | [FR-005] |
| metadata row size | artifacts JSON ≤ 2 MiB | tail-truncate per the fixed order in [DOC-10 §3] (headings → main_text → outlinks), flag `truncated` |

- R-310: Decompression bomb defense: decoded size counts toward [CFG-016];
  decoding streams MUST abort as soon as the running byte count exceeds it.
  The abort is classified identically to [FR-023]: outcome PERMANENT,
  error_class ERR-007, and no partial payload is persisted.

## 4. Content hygiene

- R-320: Never send URL userinfo anywhere [R-002]; strip credentials before logging or storage. (Security rules own the R-3xx and R-4xx ID blocks; DOC-11 storage rules own R-5xx — no collisions with other documents.)
- R-330: Do not special-case crawl-sensitive paths (`/admin`, `.git`, etc.) — scope and robots govern access; but robots DISALLOW is always final [P-2].
- R-340: No execution of content: no JS evaluation, no macro/document conversion — parsing is structural only [DOC-10].

## 5. Operator API surface

Runtime API (local only): inject seeds [FR-006], reset DEAD URL [DOC-13 §4],
trigger graceful drain. When [CFG-034] is null, the actions are exposed over
an implementation-defined local channel (e.g., Unix-domain socket or stdin)
that accepts no network connections; when [CFG-034] is set to a loopback
address, they are additionally served by the HTTP listener (a non-loopback
bind disables the mutating actions entirely per [R-406]). Actions are logged
with operator identity when available.
