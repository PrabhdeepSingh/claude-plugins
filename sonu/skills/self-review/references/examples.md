# Worked output examples

## Inline pass (small diff)

A 40-line change to a retry helper and its caller:

```
Risk: permission check on line 42 uses `user.role` before the role is loaded — could silently pass for unauthenticated users [auth/middleware.ts:42]
Risk: `deleteUserData()` is irreversible and has no dry-run mode — one bad caller will drop real data [users/service.ts:118]
Risk: the retry loop has no backoff — under load this will hammer the upstream API [api/client.ts:67]
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Cold-read synthesis (substantial diff)

A 600-line billing change — 420 production code lines once tests are excluded, so under the split: **one reader**, on `sonnet`, carrying the code checklists plus the two domain checklists whose clause matched, data-integrity and blast-radius. It reported nine raw Risk lines: five tagged `CODE/correctness` or `CODE/silent-change`, four tagged `DATA INTEGRITY` or `CONSUMERS`. It also closed with `Withheld (CODE/correctness): 2 more.` — two findings beyond that checklist's cap that it never reported, so they are not among the nine and synthesis never saw them; the line is a cue to read that checklist's domain yourself. Synthesis rejected four (two style preferences, one finding that misread an unchanged line, one with no articulable failure mechanism), merged two co-flagged duplicates, and kept:

```
Risk: `computeInvoiceTotal` rounds per-line instead of per-invoice — totals drift by cents vs the old behavior and downstream reconciliation compares exact amounts (co-flagged by CODE/silent-change and data-integrity) [billing/invoice.ts:88]
Risk: the new `amount_minor` column is read by the export job, which still expects `amount` — export degrades silently because its reader catches the missing field and emits 0 [export/rows.ts:31]
Risk: migration 014 backfills in one UPDATE with no batching — table-locks a hot table for the whole backfill [migrations/014_amount_minor.sql:9]
Risk: no test trips the new per-customer invoice cap — the limit is configured but both sides of the threshold are unasserted [billing/cap.test.ts]

Domain lenses: data-integrity (migration 014, backfill) · blast-radius (amount_minor read by the export job) · security, interface — no clause matched. Readers: 1 (sonnet)
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

Notes on what made this a good synthesis:
- The co-flagged item was merged into one entry and ranked first, not listed twice.
- The silently-degrading consumer (export job) outranked the loud one — no error will ever surface it.
- Four rejected findings stayed rejected; the list was not padded to five.
- The dispatch line is part of the output, not a footnote: it names the clause that fired for each carried domain lens, says plainly that the other two matched nothing, and ends with the reader count and tier. `/sonu:ship` reads the `Domain lenses:` part to set its `security_surface:` verdict, so an output that omits it leaves the ship flow with nothing to read.

## Prose-only diff (docs-product repo)

A 140-line wording change across two skill files in a plugin repo, where the product *is* the documents. No executable code changed, so the reader went out on the prose frame; the conditions were judged against what the prose governs:

```
Risk: the new dispatch rule contradicts the size gate two sections above — one counts prose lines, the other reads that same count as "is there code here", so a docs diff dispatches every lens anyway [skills/reviewer/SKILL.md:71]
Risk: the ledger field is described as write-once here but re-evaluated every pass below — an executor reading the contract first treats the re-evaluation as optional [commands/deploy.md:36]

Domain lenses: security (changes when a security check runs) · blast-radius (two commands consume this skill's output contract) · data-integrity, interface — no clause matched. Readers: 1 (sonnet)
Code checklists: carried on the prose frame — correctness and silent-behavior-change; test-adequacy dropped — this repo has no test suite whose adequacy the prose changes.
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

A typo or wording pass in the same repo implicates none of the checklists, and the honest output is the low-risk line below plus the inline pass — not a cold read.

## Low-risk case

```
This diff is low-risk: 30 lines, isolated to one internal helper, behavior
covered by the existing suite plus two new boundary tests, no consumed
contract touched.
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Degraded-reader case (cold read above the split, one of two readers failed)

When a reader errors or returns garbage, the output names every checklist it carried as unread instead of counting it clean:

```
Risk: `parseWindow` accepts a negative duration and schedules the job in the past — the scheduler drops past jobs without logging [scheduler/window.go:54]
Risk: the domain reader (security, data-integrity) failed to complete — auth/ and the new token path in session.go got NO independent security or data-integrity read; review those yourself [auth/, session.go]

Domain lenses: security (auth middleware) · data-integrity (session store schema) · blast-radius, interface — no clause matched. Readers: 2 (sonnet), domain reader degraded
```
> *This is a pointer for your review, not an approval. Read the diff yourself.*
