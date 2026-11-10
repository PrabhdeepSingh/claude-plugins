---
name: safe-migrations
description: >-
  Zero-downtime schema and data migration rules: expand, migrate, contract; destructive steps ship alone and late; backfills run as jobs. Use when writing or editing a migration, a backfill, or code that ships with a schema change. Not for new schema design ([[code-standards]]) or a broken migration ([[debugging]]).
---

# Safe migrations — write the safe path, not the final state

The project's own instructions win: where its AGENTS.md, CLAUDE.md or established conventions say otherwise, follow them and skip that part of this.

These rules assume a deployed service whose releases roll out while the database is live. During a rolling deploy, a blue/green switch or a rollback, old code and new schema run against each other, sometimes for hours. A project with no such deployment (a local tool, a single-user store) needs only the parts about reversibility and rehearsal.

Before writing a migration, ask whether the previous release's code still works against the schema this migration produces. If not, split it into stages.

## 1. Compatible one release in each direction

The schema after the migration must work with the previous release's code, because a deploy may be in progress or rolled back. The current code must work with the pre-migration schema only if the platform runs code before migrations, so know the ordering; most run migrations first.

## 2. Expand, migrate, contract

A breaking change (rename, drop, type change, moving data) is split across separate releases.

1. **Expand.** Add the new thing beside the old: a nullable column, a table, an index. Old code does not notice.
2. **Migrate.** Write to both old and new, backfill existing rows (section 4), then switch reads to the new location. Each step can be its own deploy and is reversible.
3. **Contract.** After a full release cycle in which nothing reads or writes the old location, drop it, in a release that changes nothing else about that data.

```sql
-- Avoid: the one-step rename. Old pods fail on every SELECT email,
-- and rolling the code back does not help because the column is gone.
ALTER TABLE users RENAME COLUMN email TO email_address;

-- Prefer: the staged path
-- Release 1 (expand):    ALTER TABLE users ADD COLUMN email_address TEXT NULL;
--                        code dual-writes email + email_address
-- Release 1.x (migrate): backfill job copies email -> email_address (batched, section 4)
--                        code switches reads to email_address, still dual-writes
-- Release 2:             code stops writing email
-- Release 3 (contract):  ALTER TABLE users DROP COLUMN email;
```

## 3. Destructive operations ship alone, one release late

`DROP COLUMN`, `DROP TABLE`, narrowing a type and tightening a constraint are safe only when nothing in any deployable version references the target.

- Do not ship one in the same release as the code change that stops using the target, because rolling that release back would need the dropped thing.
- Search first: ORM models, raw SQL, views, triggers, reports and scheduled jobs.
- The change description says which release removed the last reference, so a reviewer can check the waiting period passed.

## 4. Backfills are jobs, not migrations

An `UPDATE` over millions of rows inside the deploy pipeline holds locks, times out the deploy and leaves a half-migrated table with no record of progress. A backfill is:

- batched, with progress recorded so it can resume;
- throttled, because it shares the database with production traffic;
- idempotent, with guards such as `WHERE email_address IS NULL`, because it will be re-run;
- verified before reads switch: row counts match, a sample is checked, and edge rows (NULLs, empty strings, the oldest records) are checked.

## 5. Every step has a down path or says it has none

Additive steps reverse by dropping what was added. A destructive step cannot be reversed by a down migration, because the data is gone: mark it `-- IRREVERSIBLE: take a snapshot/backup before running` and make the backup a stated pre-step. If you wrote a down migration, run it once against a copy.

## 6. Know which operations lock

The same logical change often has a blocking and a non-blocking form. The difference does not show on a small dev database. These are engine-specific; check your engine's documentation.

| Intent | Blocking form | Safer form |
|---|---|---|
| Add an index | `CREATE INDEX` (locks writes in some engines) | `CREATE INDEX CONCURRENTLY` (Postgres) or `ALGORITHM=INPLACE` (MySQL), outside a transaction |
| Add NOT NULL | `ALTER COLUMN SET NOT NULL` (full scan under lock) | Add `CHECK (col IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT` separately (Postgres) |
| Change a column type | In-place `ALTER` rewriting the table | New column, dual-write, backfill, swap (section 2) |
| Add a column with a volatile default | Table rewrite on older engines | Add nullable, backfill, then set the default |

Keep each migration transaction short, and do not mix a schema change and a data change in one migration. The change description states how long the migration holds which lock at production row counts.

## 7. Enums and constraints

Adding an enum value is safe. Removing or renaming one follows expand and contract like any rename. A new constraint is added as non-validating first, or the data is cleaned first, and then enforced, because a constraint that fails on old rows takes the deploy down. Foreign keys onto large tables use the same two steps where the engine supports them.

## 8. Rehearse against realistic data

Run the migration against a recent production-shaped copy and record how long it took and what it locked. That decides whether it is a deploy step or a scheduled maintenance job.

When the session has no such copy, say so and write the rehearsal into the hand-off as a named pre-deploy step: what to restore, what to run, what to record. Do not claim a rehearsal, a lock duration or a tested down path that did not happen.

## Maintenance

Section 6 was last checked against Postgres and MySQL documentation in 2026-07. Managed variants differ.
