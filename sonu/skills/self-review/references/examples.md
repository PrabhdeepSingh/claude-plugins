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

A 600-line billing change. The code lens went out, and of the four domain lenses only data-integrity and blast-radius matched a clause — three lenses, reporting nine raw findings (the code lens tagged its five with the checklist that caught each, `CODE/correctness` and `CODE/silent-change`, and closed with `Withheld: 2 more.`). Synthesis rejected four (two style preferences, one finding that misread an unchanged line, one with no articulable failure mechanism), merged two co-flagged duplicates, and kept:

```
Risk: `computeInvoiceTotal` rounds per-line instead of per-invoice — totals drift by cents vs the old behavior and downstream reconciliation compares exact amounts (co-flagged by correctness + silent-change) [billing/invoice.ts:88]
Risk: the new `amount_minor` column is read by the export job, which still expects `amount` — export degrades silently because its reader catches the missing field and emits 0 [export/rows.ts:31]
Risk: migration 014 backfills in one UPDATE with no batching — table-locks a hot table for the whole backfill [migrations/014_amount_minor.sql:9]
Risk: no test trips the new per-customer invoice cap — the limit is configured but both sides of the threshold are unasserted [billing/cap.test.ts]

Domain lenses: data-integrity (migration 014, backfill) · blast-radius (amount_minor read by the export job) · security, interface — no clause matched.
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

Notes on what made this a good synthesis:
- The co-flagged item was merged into one entry and ranked first, not listed twice.
- The silently-degrading consumer (export job) outranked the loud one — no error will ever surface it.
- Four rejected findings stayed rejected; the list was not padded to five.
- The dispatch line is part of the output, not a footnote: it names the clause that fired for each dispatched domain lens and says plainly that the other two matched nothing. `/sonu:ship` reads that line to set its `security_surface:` verdict, so an output that omits it leaves the ship flow with nothing to read.

## Prose-only diff (docs-product repo)

A 140-line wording change across two skill files in a plugin repo, where the product *is* the documents. No executable code changed, so no lens ran its code prompt; the conditions were judged against what the prose governs:

```
Risk: the new dispatch rule contradicts the size gate two sections above — one counts prose lines, the other reads that same count as "is there code here", so a docs diff dispatches every lens anyway [skills/reviewer/SKILL.md:71]
Risk: the ledger field is described as write-once here but re-evaluated every pass below — an executor reading the contract first treats the re-evaluation as optional [commands/deploy.md:36]

Domain lenses: security (changes when a security check runs) · blast-radius (two commands consume this skill's output contract) · data-integrity, interface — no clause matched.
Code lens: dispatched on the prose frame with the correctness and silent-behavior-change checklists; test-adequacy checklist dropped — this repo has no test suite whose adequacy the prose changes.
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

A typo or wording pass in the same repo implicates none of the seven, and the honest output is the low-risk line below plus the inline pass — not a fan-out.

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
