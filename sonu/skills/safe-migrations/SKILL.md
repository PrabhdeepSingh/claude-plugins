---
name: safe-migrations
description: >-
  Zero-downtime schema and data migration discipline — expand → migrate → contract, never destructive in the release that ships the code, backfills as jobs, every step reversible. INVOKE PROACTIVELY when creating or editing a migration in any framework, writing ALTER TABLE / index / enum / constraint changes, planning a backfill, or shipping code and schema together. Not for greenfield schema design ([[code-standards]]) or diagnosing a broken migration ([[debugging]]).
---

# Safe migrations — the schema change and the safe path to it are different artifacts

The single most expensive junior mistake in backend work: writing the migration for the **final state** instead of the **safe path** to it. `ALTER TABLE users RENAME COLUMN email TO email_address` looks correct — and it is correct, for a world where the deploy is instantaneous. Real deploys are rolling: for minutes (or hours, or days if the deploy fails), **old code and new schema run against each other**. The rename ships, old pods still SELECT `email`, and production is down before the deploy finishes. Every rule below exists because the database outlives and overlaps every version of the code that touches it.

## How to apply this

Before writing any migration, answer one question: **"will the *previous* release's code still work against the schema this migration produces?"** If no, the migration must be split into stages. Then walk the change through the expand → migrate → contract sequence, and run the self-check before shipping.

---

## 1. The core law: every migration is compatible one release in each direction

During a rolling deploy, blue/green switch, or a failed-deploy rollback, the schema meets both the old and the new code. So:

- The schema after the migration must work with the **previous** release's code (deploy in progress, or rolled back).
- The current code must work with the **pre-migration** schema only if your platform runs code before migrations — know your ordering; most run migrations first.

This is the invariant everything else derives from. A migration that requires the new code to already be everywhere is a migration that requires downtime — and nobody scheduled any.

## 2. Expand → migrate → contract

Any breaking change (rename, drop, type change, moving data) decomposes into three phases across **separate releases**:

1. **Expand** — add the new thing alongside the old: a new nullable column, a new table, a new index. Purely additive; old code doesn't notice.
2. **Migrate** — teach the code to write to both old and new (dual-write), backfill existing rows (section 4), then switch reads to the new location. Each of these can be its own deploy; each is individually reversible.
3. **Contract** — only after a full release cycle where nothing reads or writes the old location: drop it. The drop ships **alone**, in a release that changes nothing else about that data.

**Example — renaming `email` to `email_address`:**

```sql
-- Avoid: the "correct" one-step rename — downtime by design
ALTER TABLE users RENAME COLUMN email TO email_address;
-- Old pods now crash on every SELECT email. Rollback of the code doesn't
-- help — the column is already gone.

-- Prefer: the staged path
-- Release 1 (expand):    ALTER TABLE users ADD COLUMN email_address TEXT NULL;
--                        code dual-writes email + email_address
-- Release 1.x (migrate): backfill job copies email -> email_address (batched, section 4)
--                        code switches reads to email_address, still dual-writes
-- Release 2:             code stops writing email
-- Release 3 (contract):  ALTER TABLE users DROP COLUMN email;
```

Yes, it's four releases for a rename. That's the honest cost of zero downtime — and steps 1–3 are each trivially safe, boring, and reversible, which is the point.

## 3. Destructive operations ship alone, one release late

`DROP COLUMN`, `DROP TABLE`, type-narrowing, tightening a constraint: these are only safe when **nothing in any deployable version references the target**. That means:

- Never in the same release as the code change that stops using it — the rollback of that release would need the dropped thing back.
- Grep first: ORM models, raw SQL, views, triggers, reports, and that one cron job nobody remembers.
- The drop migration's PR description says what release removed the last reference, so the reviewer can verify the waiting period actually passed.

## 4. Backfills are jobs, not migrations

A migration that runs `UPDATE users SET email_address = email` on fifty million rows inside the deploy pipeline will hold locks for minutes, time out the deploy, and leave you half-migrated with no record of progress. Data movement is an **operational job**, not a schema migration:

- **Batched** (thousands of rows per transaction, not one transaction for everything), with progress recorded so it's **resumable** after interruption.
- **Throttled** — sleep between batches; the backfill shares the database with production traffic.
- **Idempotent** — running a batch twice produces the same result (`WHERE email_address IS NULL` style guards), because it *will* get re-run.
- **Verified** — before flipping reads: row counts match, spot-check a sample, and check the edge rows (NULLs, empty strings, the oldest records).

## 5. Every step has a down path — or an explicit confession

- Additive steps (expand phase) are trivially reversible: the down migration drops what the up added.
- Destructive steps are **not** reversible by a down migration — a `DROP COLUMN` down can restore the column but not the data. Mark these explicitly: `-- IRREVERSIBLE: take a snapshot/backup before running`, and make the backup a stated pre-step, not a hope.
- If you wrote a down migration, **run it once** against a copy — an untested down migration is a rollback plan made of wishes.

## 6. Know which operations lock — and use the non-blocking forms

The same logical change often has a blocking and a non-blocking form; the difference is invisible in a dev database with 200 rows and an outage at production scale. The patterns (engine-specific — verify against your engine's docs, see Provenance):

| Intent | Blocking form | Safer form |
|---|---|---|
| Add an index | `CREATE INDEX` (locks writes on the table in some engines) | `CREATE INDEX CONCURRENTLY` (Postgres) / `ALGORITHM=INPLACE` (MySQL) — outside a transaction |
| Add NOT NULL | `ALTER COLUMN SET NOT NULL` (full table scan under lock) | Add a `CHECK (col IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT` separately (Postgres) |
| Change a column type | In-place `ALTER` rewriting the table | New column + dual-write + backfill + swap (section 2) |
| Add a column with a volatile default | Table rewrite on older engines | Add nullable, backfill, then set the default |

General rules: keep every migration transaction **short**; never mix a schema change and a data change in one migration; and treat "how long will this hold what lock at production row counts" as a question the PR must answer, not discover.

## 7. Enums and constraints evolve additively too

- Adding an enum value is safe; **removing or renaming** one follows expand→contract like any rename (new value in, code migrated, old value retired after a release, then removed).
- New constraints validate **existing** data before they're enforced — add as non-validating first (or clean the data first), then enforce. A constraint that fails on legacy rows takes the deploy down with it.
- Foreign keys onto big tables: same non-blocking two-step where the engine supports it.

## 8. Rehearse against realistic data

A migration tested only against the dev database has been tested against nothing. Before production: run it against a recent production-shaped copy (a snapshot restore, a staging clone) and record **how long it took and what it locked**. That number is what tells you whether it's a deploy step or a scheduled maintenance job. This is the migration version of [[tdd]]'s "watch the test fail" — evidence over assumption.

---

## Self-check before you ship a migration

- Will the **previous** release's code run correctly against the post-migration schema? (If no: split into expand → migrate → contract.)
- Is anything destructive (drop/rename/narrow) shipping in the same release as the code that stops using it? (Move it one release later, shipping alone.)
- Is any data movement inside a schema migration? (Extract it into a batched, resumable, idempotent, throttled job.)
- Does every step have a tested down path — or an explicit `IRREVERSIBLE` marker with a backup as a stated pre-step?
- Do you know what locks this takes and for how long **at production row counts** — and did you use the non-blocking form where one exists?
- New constraints/enum changes: validated against existing data before enforcement? Enum removals staged like renames?
- Was the migration rehearsed against a production-shaped copy, with its duration recorded?

## Provenance and maintenance

The staging discipline (sections 1–5, 7–8) is durable. Section 6's lock behaviors are **engine- and version-specific** — last sanity-checked 2026-07 against Postgres and MySQL documentation. Before relying on a specific non-blocking form, verify it against your engine's current docs (`CREATE INDEX CONCURRENTLY` caveats, MySQL online DDL support per operation, and managed-service variants like Aurora/Azure SQL differ). The *principle* — know the lock, prefer the non-blocking form, rehearse at scale — survives any engine change.
