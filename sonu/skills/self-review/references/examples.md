# Worked output examples

## Inline pass (small diff)

A 40-line change to a retry helper and its caller:

```
Risk: permission check on line 42 uses `user.role` before the role is loaded — could silently pass for unauthenticated users [auth/middleware.ts:42]
Risk: `deleteUserData()` is irreversible and has no dry-run mode — one bad caller will drop real data [users/service.ts:118]
Risk: the retry loop has no backoff — under load this will hammer the upstream API [api/client.ts:67]
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Fan-out synthesis (substantial diff)

Six lenses reported nine raw findings on a 600-line billing change. Synthesis rejected four (two style preferences, one finding that misread an unchanged line, one with no articulable failure mechanism), merged two co-flagged duplicates, and kept:

```
Risk: `computeInvoiceTotal` rounds per-line instead of per-invoice — totals drift by cents vs the old behavior and downstream reconciliation compares exact amounts (co-flagged by correctness + silent-change) [billing/invoice.ts:88]
Risk: the new `amount_minor` column is read by the export job, which still expects `amount` — export degrades silently because its reader catches the missing field and emits 0 [export/rows.ts:31]
Risk: migration 014 backfills in one UPDATE with no batching — table-locks a hot table for the whole backfill [migrations/014_amount_minor.sql:9]
Risk: no test trips the new per-customer invoice cap — the limit is configured but both sides of the threshold are unasserted [billing/cap.test.ts]
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

Notes on what made this a good synthesis:
- The co-flagged item was merged into one entry and ranked first, not listed twice.
- The silently-degrading consumer (export job) outranked the loud one — no error will ever surface it.
- Four rejected findings stayed rejected; the list was not padded to five.

## Low-risk case

```
This diff is low-risk: 30 lines, isolated to one internal helper, behavior
covered by the existing suite plus two new boundary tests, no consumed
contract touched.
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Degraded-lens case (fan-out, one lens failed)

When a lens errors or returns garbage, the output names the gap instead of counting it clean:

```
Risk: `parseWindow` accepts a negative duration and schedules the job in the past — the scheduler drops past jobs without logging [scheduler/window.go:54]
Risk: the security lens failed to complete — auth/ and the new token path in session.go got NO independent security read; review those yourself [auth/, session.go]
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*
