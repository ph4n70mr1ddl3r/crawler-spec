---
id: DOC-07
title: URL Lifecycle State Machine
version: 1.2.0
---

# URL Lifecycle State Machine

## States

| ID | Name | Terminal? | Meaning |
|----|------|-----------|---------|
| ST-100 | QUEUED | no | Passed filters; awaiting scheduling. |
| ST-110 | SCHEDULED | no | Dispatched; holding politeness slot. |
| ST-120 | FETCHING | no | HTTP request in flight. |
| ST-130 | FETCHED | no | Payload stored; extraction pending. |
| ST-140 | DONE | yes | Parsed, extracted, artifacts stored. |
| ST-150 | RETRY_WAIT | no | Retryable failure; backoff timer running. |
| ST-180 | DEAD | yes | Retries exhausted or permanent error. |
| ST-190 | EXCLUDED | yes | Never crawled; reason code required. |

Reason codes for ST-190: `OUT_OF_SCOPE`, `ROBOTS_DISALLOW`, `ROBOTS_UNKNOWN_TIMEOUT`,
`TRAP_PARAM`, `TRAP_PATH_BUDGET`, `DEPTH_LIMIT`, `CAP_REACHED`, `BLOCKLIST`.

## Transitions

```
(creation)          ─► ST-100   filters passed [FR-003]
(creation)          ─► ST-190   scope/trap/blocklist filter failed [FR-004], [DOC-06 §5]
ST-100 ─► ST-110                scheduler dispatch [FR-012], atomic
ST-100 ─► ST-190                robots DISALLOW at gate time [FR-031]
ST-110 ─► ST-120                robots ALLOW confirmed, request sent
ST-110 ─► ST-190                robots DISALLOW/UNKNOWN-timeout [FR-031]
ST-120 ─► ST-130                success — 2xx with payload stored [FR-043],
                                or 304 UNCHANGED reusing the existing payload
                                [R-121], [R-144]
ST-120 ─► ST-150                retryable failure [DOC-13 §3]
ST-120 ─► ST-180                permanent failure or retry budget exhausted
ST-150 ─► ST-100                backoff elapsed, attempts < [CFG-020]
ST-150 ─► ST-180                attempts = [CFG-020]
ST-130 ─► ST-140                extraction complete
ST-140 ─► ST-100                recrawl due [FR-050] (attempts reset to 0)
```

No other transitions exist. Any observed other transition is a defect.

## State-dependent rules

- R-050: Only ST-100 records are visible to the Scheduler.
- R-051: ST-110/ST-120 records own one inflight unit each against host/global caps; caps are released exactly once, on transition out of ST-120 (to ST-130, ST-150, or ST-180).
- R-052: ST-140→ST-100 recrawl resets `attempts=0`; priority is recomputed per [DOC-12 §2] (R-201).
- R-053: `attempts` is incremented exactly once per fetch attempt, inside the
  dispatch transaction [FR-012], [T-1]; it is therefore already counted when a
  crash mid-fetch is classified as retryable [DOC-13 §5].

## Crash recovery

- R-060: On startup, every record in {ST-110, ST-120} MUST be reset to ST-100,
  keeping its attempt count. Because dispatch advanced `next_allowed_fetch_at`
  transactionally [FR-012], politeness cannot be violated by this reset.
  Host rows are rebuilt conservatively in the same recovery pass: `inflight` is
  recomputed as the count of records still in {ST-110, ST-120} after the reset
  (i.e., 0 for a clean recovery), restoring INV-3 [NFR-011].
- R-061: ST-130 records at startup resume extraction (idempotent: re-running
  extraction overwrites artifacts deterministically).
- R-062: On completion of a redirect chain, the system MUST upsert a URL Record
  for the chain's final target identity, with depth equal to the source record's
  depth (redirects do not count toward depth [DEC-009]),
  `discovered_from` = source identity, and state per the fetch outcome
  (ST-130 on success, then normal progression; ST-150/ST-180 on failure).
  Intermediate hops are persisted on the source's `redirect_chain` [R-133] and
  receive no URL Records. The final target is exempt from trap filters at this
  creation point (it was already fetched); scope was verified per hop [R-030].
