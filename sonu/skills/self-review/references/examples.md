# Example outputs

The paths and findings are illustrative.

## Inline pass (small diff)

```
Risk: permission check on line 42 uses `user.role` before the role is loaded — could pass for unauthenticated users [auth/middleware.ts:42]
Risk: `deleteUserData()` is irreversible and has no dry-run mode — one bad caller drops real data [users/service.ts:118]
Risk: the retry loop has no backoff — under load this hammers the upstream API [api/client.ts:67]
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Independent reader (substantial diff)

One reader carried the code checklists plus data-integrity and blast-radius. It returned nine Risk lines and `Withheld (CODE/correctness): 2 more.` Four were rejected (two style preferences, one misread of an unchanged line, one with no stated mechanism) and two duplicates were merged:

```
Risk: `computeInvoiceTotal` rounds per line instead of per invoice — totals drift by cents and reconciliation compares exact amounts (flagged by CODE/silent-change and data-integrity) [billing/invoice.ts:88]
Risk: the new `amount_minor` column is read by the export job, which still expects `amount` — its reader catches the missing field and emits 0 [export/rows.ts:31]
Risk: migration 014 backfills in one UPDATE with no batching — locks a hot table for the whole backfill [migrations/014_amount_minor.sql:9]
Risk: no test trips the new per-customer invoice cap [billing/cap.test.ts]

Domain lenses: data-integrity (migration 014, backfill) · blast-radius (amount_minor read by the export job) · security, interface — no clause matched. Readers: 1
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

The silently degrading consumer ranks above the loud one, the merged item is listed once, and the list was not padded to five.

## Low-risk diff

```
This diff is low-risk: 30 lines, isolated to one internal helper, behavior
covered by the existing suite plus two new boundary tests, no consumed
contract touched.
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## A reader failed

```
Risk: `parseWindow` accepts a negative duration and schedules the job in the past — the scheduler drops past jobs without logging [scheduler/window.go:54]
Risk: the domain reader (security, data-integrity) did not complete — auth/ and the new token path in session.go got no independent read; review those yourself [auth/, session.go]

Domain lenses: security (auth middleware) · data-integrity (session store schema) · blast-radius, interface — no clause matched. Readers: 2
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*
