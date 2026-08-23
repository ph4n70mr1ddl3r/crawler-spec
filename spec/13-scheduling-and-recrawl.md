---
id: DOC-12
title: Scheduling, Priority, and Recrawl
version: 1.9.0
---

# Scheduling and Recrawl

## 1. Model

Time in the scheduler is a single monotonic counter [DEC-012]. The Frontier is
a persistent index on `due_at_mono`; only ST-100 records with
`due_at_mono ≤ now` are dispatch candidates. Among due candidates, selection
order is `(priority DESC, url_identity ASC)` [FR-010], [§3] — `due_at_mono`
governs when a record becomes due, never the ordering of already-due work.
A newly created ST-100 record is immediately due: `due_at_mono` is set to its
enqueue time (later transitions overwrite it: backoff [DOC-07 §2], recrawl [§4]).

Before candidacy, the scheduling loop performs the due-time promotions
(R-050 makes only ST-100 records dispatch candidates, so the promotions are
part of the loop, not fetch dispatches):

- ST-150 → ST-100 when `now ≥ next_attempt_mono` (and `attempts <
  [CFG-020]`, which always holds — [DOC-13 §3] evaluates the budget at
  failure time), setting `due_at_mono := next_attempt_mono` [DOC-07 §2];
- ST-140 → ST-100 when `now ≥ due_at_mono` (the recrawl due time computed
  at fetch completion [§4]) [FR-050].

Both promotions are wake sources for the loop [R-211]; a promoted record is
dispatchable in the same iteration.

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

Manual boosts [CFG-031]: a prefix matches when it is a case-sensitive string
prefix of the URL identity; if several prefixes match, the largest single
boost applies (matches are never summed); the boost term remains within 0–300.

- R-200: Priority never affects politeness or caps; it only reorders due work.
- R-201: Priority is recomputed on each ST-140→ST-100 recrawl transition.

## 3. Dispatch loop (normative pseudocode)

```
loop:
  wait until ∃ candidate c ∈ frontier where due(c) and gates [FR-011] a–d pass
      // a–c are per-Host gates; (d) is per-URL; (e) is not a wait condition — see below
  pick c = highest-priority gate-passing candidate (ties: lexicographically smallest identity)
  if gate [FR-011](e) fails for c:              // registrable-domain cap [FR-005]
      move c → ST-190/CAP_REACHED               // exclusion, not deferral
      continue                                  // re-select without waiting;
                                                // terminates — ST-190 is terminal, so
                                                // each exclusion shrinks the candidate set
  execute dispatch transaction [T-1]
  hand to fetcher worker pool
```

- R-210: Starvation freedom [NFR-015]: if a candidate is due and its gates pass,
  it MUST be dispatched within one loop iteration; gates are the ONLY reason to wait.
- R-211: The loop MUST NOT busy-poll; it sleeps until the earliest of
  (next `due_at_mono` over ST-100, next `next_attempt_mono` over ST-150,
  next recrawl `due_at_mono` over ST-140 [FR-050], next politeness expiry,
  next robots-deferral expiry, next robots-unknown threshold expiry
  (`robots_deferred_since_mono + [CFG-040]` [R-103])), and is woken
  immediately by any event that creates or re-due-s a candidate or changes
  a sleep-list key or a gate verdict — discovery/ingestion [C1], operator
  actions (seed injection [FR-006], DEAD reset [DOC-13 §4]), extraction
  completion (ST-130→ST-140 installs a recrawl due time that may precede
  the current sleep target), and robots revalidation completion ([R-104] —
  a verdict change can gate or un-gate candidates) — so an empty frontier
  never causes an unbounded sleep and no scheduler-relevant time is slept
  past.

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
- R-221: `Cache-Control: no-store` responses are not refetched automatically
  (treated as one-shot). In v1 this affects recrawl eligibility only: the
  payload is fetched, stored [FR-043], and extracted normally.
- R-222: Recrawl of a page whose URL Record was deleted by retention does not happen (records are the source of truth).

## 5. Backoff interplay

Host backoff [DOC-08 §4] applies to all scheduling for that host, including
recrawls and robots re-fetching. Backoff state is per-host, persisted, and
survives restart [NFR-011].

## 6. Determinism requirement

Given identical config and a recorded network fixture, the sequence of
`(url_identity, attempt, outcome)` triples MUST be reproducible [NFR-006],
because: monotonic scheduling, deterministic jitter, deterministic tie-breaks.
