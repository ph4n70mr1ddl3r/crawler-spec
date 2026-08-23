---
id: DOC-12
title: Scheduling, Priority, and Recrawl
version: 1.4.0
---

# Scheduling and Recrawl

## 1. Model

Time in the scheduler is a single monotonic counter [DEC-012]. The Frontier is
conceptually a min-heap on `due_at_mono` with tie-break `(priority DESC, url_identity ASC)`
[FR-010]. Only ST-100 records with `due_at_mono ≤ now` are dispatch candidates.
A newly created ST-100 record is immediately due: `due_at_mono` is set to its
enqueue time (later transitions overwrite it: backoff [DOC-07 §2], recrawl [§4]).

## 2. Priority computation (0–1000, default 500)

```
priority = clamp(500
           − 40 × depth                                  // deeper = lower
           + 100 if URL is a seed
           + boost from [CFG-031] manual per-prefix boosts  // optional, 0–300
           − 100 × min(consecutive_failures(host),3)     // deprioritize flaky hosts
           + 50 if host.pages_crawled > 0                // host has prior success [DOC-11 §1]
         , 0, 1000)
```

- R-200: Priority never affects politeness or caps; it only reorders due work.
- R-201: Priority is recomputed on each ST-140→ST-100 recrawl transition.

## 3. Dispatch loop (normative pseudocode)

```
loop:
  wait until ∃ candidate c ∈ frontier where due(c) and gates pass for c.host
      gates = [FR-011] a–d
  pick c = highest-priority candidate (ties: lexicographically smallest identity)
  execute dispatch transaction [T-1]
  hand to fetcher worker pool
```

- R-210: Starvation freedom [NFR-015]: if a candidate is due and its gates pass,
  it MUST be dispatched within one loop iteration; gates are the ONLY reason to wait.
- R-211: The loop MUST NOT busy-poll; it sleeps until the earliest of
  (next `due_at_mono` over ST-100, next `next_attempt_mono` over ST-150,
  next politeness expiry, next robots-deferral expiry).

## 4. Freshness & recrawl

On success, a page's next recrawl time:

If [CFG-025]=0 the URL is never scheduled for recrawl (one-shot fetch).
Otherwise:

```
base_interval   = CFG-025
interval        = base_interval
                  × (1 ± CFG-026 jitter, seeded by hash(url_identity, config_hash))
                  // deterministic jitter: same config ⇒ same offset.
                  // config_hash (not run_id) is the seed so restarts/replays
                  // reproduce identical schedules [NFR-006], [AC-033]
if validators present and server returned 304 on a refetch:
                  interval doubles per consecutive 304, capped at
                  4 × base_interval
                  // consecutive-304 count persists as urls.consecutive_unchanged
                  // [DOC-11 §1]; any full 200 resets it (and the multiplier) to 0/1
                  // Retry-After never affects recrawl intervals
due_at_mono     = fetch_complete_mono + interval
```

- R-220: Recrawl respects caps [FR-005] and robots at dispatch time [DOC-08 §3].
- R-221: `Cache-Control: no-store` responses are not refetched automatically (treated as one-shot).
- R-222: Recrawl of a page whose URL Record was deleted by retention does not happen (records are the source of truth).

## 5. Backoff interplay

Host backoff [DOC-08 §4] applies to all scheduling for that host, including
recrawls and robots re-fetching. Backoff state is per-host, persisted, and
survives restart [NFR-011].

## 6. Determinism requirement

Given identical config and a recorded network fixture, the sequence of
`(url_identity, attempt, outcome)` triples MUST be reproducible [NFR-006],
because: monotonic scheduling, deterministic jitter, deterministic tie-breaks.
