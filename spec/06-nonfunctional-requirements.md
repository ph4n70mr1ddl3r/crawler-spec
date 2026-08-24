---
id: DOC-05
title: Non-Functional Requirements
version: 1.12.0
---

# Non-Functional Requirements

Targets are for reference hardware: 4 vCPU, 8 GiB RAM, SSD, 100 Mbps.

| ID | Category | Requirement |
|----|----------|-------------|
| NFR-001 | Throughput | Sustained steady-state rate of ≥ 10 pages/sec and burst capability of ≥ 50 pages/sec, provided remote hosts permit it within politeness constraints. Politeness caps dominate throughput by design. |
| NFR-002 | Latency | Scheduler decision latency (due URL selected → dispatched) ≤ 50 ms at p99 under load ≤ 100k queued URLs. |
| NFR-003 | Memory | RSS ≤ 512 MiB regardless of frontier size; Frontier MUST spill to the Metadata Store, never grow unboundedly in RAM. |
| NFR-004 | Disk | Content Store growth is bounded: at most [CFG-005] distinct payloads per recrawl generation (modulo the bounded [R-062] redirect-target exemption [FR-005]), with at most ceil([CFG-027] × 86400 / ([CFG-025] × (1 − [CFG-026]))) generations retained when recrawl is enabled ([CFG-025]>0; [CFG-027] is days, [CFG-025] seconds — the formula converts to seconds before dividing; the divisor is the *minimum* jittered interval: jitter only shrinks intervals and the 304 multiplier (≤ 4×) only grows them, so the bound holds for every URL; [CFG-026] < 1 by range, so the divisor is positive), else one; the retention job enforces [DOC-11 §6]. |
| NFR-005 | Startup | Cold start with 1M existing URL Records ≤ 60 s before first fetch. |
| NFR-006 | Determinism | Same inputs (config, seeds, captured network fixture) ⇒ same sequence of fetch decisions. Enables replay testing. |
| NFR-007 | Observability | Every state transition increments a labeled metric; every fetch writes a structured log line [DOC-15]. |
| NFR-008 | Portability | No OS-specific features beyond POSIX + standard sockets; must run on Linux/macOS. |
| NFR-009 | Testability | Every component (C1–C9) testable in isolation via injected fake transports/stores; network fixtures recorded in HAR-like format. |
| NFR-010 | Durability | Acknowledged state changes survive process kill -9 at any instant (write-ahead or atomic commits). No lost discoveries. |
| NFR-011 | Safety on restart | Restart NEVER violates politeness: persisted `next_allowed_fetch_at` honored; inflight counts rebuilt conservatively. |
| NFR-012 | Idempotency | Re-running ingestion of the same seed set produces no duplicate work. |
| NFR-013 | Security posture | All SSRF/trap guards fail closed: unknown ⇒ block [DOC-16]. |
| NFR-014 | Compliance | robots.txt handling conforms to RFC 9309 semantics for the groups matching UA Token. |
| NFR-015 | Resource fairness | Per-host load is inherently capped by [CFG-009]; globally, scheduling MUST be fair across Hosts — no starvation: every due URL whose gates pass is dispatched within one scheduling-loop iteration of becoming due [R-210]. |
| NFR-016 | Log hygiene | Logs MUST NOT contain full payload bodies; URLs MAY be logged; credentials (userinfo in URL) MUST be redacted. |
