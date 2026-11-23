---
id: DOC-05
title: Non-Functional Requirements
version: 1.33.0
---

# Non-Functional Requirements

Targets are for reference hardware: 4 vCPU, 8 GiB RAM, SSD, 100 Mbps.

| ID | Category | Requirement |
|----|----------|-------------|
| NFR-001 | Throughput | Sustained steady-state rate of ≥ 10 pages/sec and burst capability of ≥ 50 pages/sec, provided remote hosts permit it within politeness constraints. Politeness caps dominate throughput by design. |
| NFR-002 | Latency | Scheduler decision latency (due URL selected → dispatched) ≤ 50 ms at p99 under load ≤ 100k queued URLs. |
| NFR-003 | Memory | RSS ≤ 512 MiB regardless of frontier size; Frontier MUST spill to the Metadata Store, never grow unboundedly in RAM. |
| NFR-004 | Disk | Content Store growth is bounded as follows:
- At most [CFG-005] distinct payloads are stored per recrawl generation (modulo the [R-062] redirect-target exemption [FR-005]).
- When [CFG-025]>0 and [CFG-027]>0 (recrawl and retention both enabled): at most ceil([CFG-027] × 86400 / ([CFG-025] × (1 − [CFG-026]))) generations are retained. Here [CFG-027] is in days, [CFG-025] is in seconds (converted before dividing); the divisor [CFG-025] × (1 − [CFG-026]) is positive because [CFG-026]<1 by range, and jitter ±adjusts intervals by a factor of (1 − [CFG-026]) while the 304 multiplier (≤4×) only grows them; the divisor (1 − [CFG-026]) provides a conservative worst-case bound so the at-most claim holds for every URL.
- When [CFG-025]=0 (no automatic recrawl): no recrawl due time is installed, so a URL is fetched at most once per fetch cycle — a single generation per cycle, not per URL identity. This does not forbid a second fetch of the same identity by a rediscovery refresh ([CFG-021]=true, [FR-051], [R-052], [DOC-12 §1], [AC-072]) — a same-run refetch upserts the prior `pages` row [DOC-11 §1] — nor an operator DEAD reset [DOC-13 §4] or the same-attempt [R-144] cache-miss refetch; the Content Store bound remains one stored payload per byte-identical body [DEC-005].
- When [CFG-027]=0 (keep forever): the operator has disabled retention; pages never expire and growth across generations is unbounded by explicit operator choice, so this NFR claims no bound. The retention job enforces [DOC-11 §6]. |
| NFR-005 | Startup | Cold start with 1M existing URL Records ≤ 60 s before first fetch. |
| NFR-006 | Determinism | Same inputs (config, seeds, captured network fixture) ⇒ same sequence of fetch decisions. Enables replay testing. |
| NFR-007 | Observability | Every state transition increments a labeled metric; every fetch that reaches a classified outcome writes exactly one structured fetch log line with the fields in [DOC-15 §2] (`url_identity`, `attempt`, `outcome`, `error_class`, `http_status`, `timings_ms`) [DOC-09 §6], [AC-133]. An attempt interrupted by a crash [R-060] or a send-time robots abort [R-054], [T-5] writes NO fetch log line: no outcome, error class, status, or timings exist and MUST NOT be synthesized [DOC-09 §6], [AC-131], [AC-132]; a robots.txt exchange yields a gate verdict, not a page fetch [DOC-08 §2.2]. Redirect hops [R-131] and a same-attempt [R-144] refetch are folded into that one line [R-133], never emitted as extra fetch log lines. |
| NFR-008 | Portability | No OS-specific features beyond POSIX + standard sockets; must run on Linux/macOS. |
| NFR-009 | Testability | Every component (C1–C9) testable in isolation via injected fake transports/stores; network fixtures recorded in HAR-like format. |
| NFR-010 | Durability | Acknowledged state changes survive process kill -9 at any instant (write-ahead or atomic commits). No lost discoveries. |
| NFR-011 | Safety on restart | Restart NEVER violates politeness: persisted `next_allowed_fetch_at` honored; `inflight` counts rebuilt to exactly 0 for every Host [R-060] — a precise rebuild, not a conservative over-estimate: every {ST-110, ST-120} record has just been reclassified out of the unit-owning states at restart and all robots exchanges died with the process, so a non-zero "conservative" count would strand Host slots forever (each crashed task is gone and no [T-2] will release its slot) and deadlock dispatch, violating INV-3. |
| NFR-012 | Idempotency | Re-running ingestion of the same seed set produces no duplicate work. |
| NFR-013 | Security posture | All SSRF/trap guards fail closed: unknown ⇒ block [DOC-16]. |
| NFR-014 | Compliance | robots.txt handling conforms to RFC 9309 semantics for the groups matching UA Token. |
| NFR-015 | Resource fairness | Per-host load is inherently capped by [CFG-009]; globally, scheduling MUST be fair across Hosts — no starvation: every due URL whose gates pass is dispatched within one scheduling-loop iteration of becoming due, **provided the global-unit pool is not full** [R-210], [R-212]. A due candidate deferred because the [CFG-010] global unit is held by another fetch stays ST-100 and is re-selected on the next wake when that unit is released [R-212], [R-211]; fairness bounds the wait only while a global unit is available and never exempts a dispatch from [FR-011(b)] [AC-011]. |
| NFR-016 | Log hygiene | Logs MUST NOT contain full payload bodies; URLs MAY be logged; credentials (userinfo in URL) MUST be redacted. |
