---
name: code-standards
description: >-
  Prabhdeep (Sonu) Singh's house rules for code: schema and API naming, comments, function size, error handling, logging, config defaults, dependencies. Use when writing, refactoring or reviewing code. Test-first rules: [[tdd]]; contract changes: [[blast-radius]]; trust boundaries: [[security]].
---

# Code standards — the house rules

Explicit project instructions (its AGENTS.md, CLAUDE.md or other written rules) override these rules; follow them and skip that part of this. An established convention in the code overrides the style and structure rules here, as the next paragraph but one says. Existing code or habit alone never waives a trust boundary, a required human authorization or a data-safety rule (sections 7, 9, 10 and 14): where the code conflicts with one of those, say so instead of following it.

These are the owner's preferences and the lessons that cost something to learn. General good practice is assumed and not repeated here. The reader of the code is a senior engineer with no context six months from now.

**A consistent codebase beats a "correct" one.** Read the surrounding file before editing. Where the existing code has a clear convention, match it even where it differs from this guide, and mention the conflict instead of working against it. Apply this guide in full when starting fresh or where no convention exists.

## How to work

**Surface ambiguity.** When a request has more than one reasonable reading, or assumes something the codebase contradicts, say so before building. Ask when a wrong guess would be expensive to undo; otherwise state the assumption and proceed.

**Smallest change that works.** Before writing, find where the behaviour already happens and make the change there. Read the code the request touches and trace the real path first, because a small change in the wrong place is a second bug. Then make the smallest change that fully does what was asked. For a request to add a trailing space to pasted text, edit the existing paste operation directly, unless tracing the code shows why that cannot satisfy the request; do not add a setting, helper or layer. Smallest means the least implementation that meets the request and every rule that applies to it: the test written first, verification, validation at a trust boundary, security, accessibility and failure handling are part of the change, not extras. If the implementation is growing beyond that, stop and explain the concrete need before continuing. Leave optional restructuring until something shows it is needed, such as a review finding or a measured problem. A second use alone does not justify an abstraction (section 4's third-occurrence rule stands), and when this change does need restructuring, do it as section 4's separate, green steps. Say in the hand-off what you left simple on purpose.

**Reuse before writing.** Before adding a helper, branch, option or component, check in this order and stop at the first that answers: does it need to exist at all; is it already in this codebase; is it in the standard library; is it a platform primitive (a native element over a component library, CSS over JavaScript, a database constraint over application code); is it in an installed dependency. Only then write the minimum code that does what was asked. When you skip something as unnecessary (a case that cannot occur, configurability nobody asked for, an abstraction with one use), name the skip in one line of the hand-off, not in a comment. This never removes validation at a trust boundary, error handling that prevents data loss, a security or accessibility requirement, or anything explicitly requested. Before relying on a platform feature, check it against the browsers and runtime versions the project supports.

**Verify APIs against the installed version**, not memory, before calling anything beyond the language's core. A new package is an architectural decision, so do not add one for what a few lines or an existing dependency can do.

**Change only what the task requires.** Do not improve or reformat adjacent lines. Clean up what your change orphaned, and flag out-of-scope dead code to the reviewer instead of deleting it. When an approach fails, revert it fully before trying the next. Re-read a file before editing when your view of it may be stale: after a subagent touched it, after a rebase, after a reviewer's fix landed.

**Report only what you observed.** "Tests pass" means you ran them this session and saw them pass. Every count you report (call sites, rows, tests) is the output of a command you ran. In a hand-off, ticket or subagent prompt, give a path and the relevant lines, not a pasted log or large file.

## 1. Names

Do not use `data`, `info`, `item`, `value`, `temp`, `result`, `obj`, `arr`, `list` or `thing` as identifiers, or `Manager`, `Helper`, `Util` or `Service` as a catch-all suffix. They describe the type, which the reader can already see. Name for the domain, in the vocabulary the business uses. Booleans read as a yes/no statement (`isExpired`, `hasUnpaidInvoices`).

## 2. Schema and API naming

A column or API field is far more expensive to rename than a variable, so get these right at the start and keep them consistent.

- API, JSON and database field names are `snake_case` (`first_name`, `is_active`). In camelCase ecosystems, map at the boundary.
- Identifiers are UUIDs, not auto-increment integers, which leak record counts, collide across merged systems and can be guessed in URLs. Name them `id` on their own table and `<entity>_id` as a foreign key.
- Date and time fields are `<thing>_date` or `<thing>_at`, named for the event. Pick one per project. Store UTC.
- Store names as separate fields (`first_name`, `last_name`), not one `full_name` to split later.
- Booleans read as a question in data too (`is_active`, `has_subscription`).
- Money is not a float. Store integer minor units (`total_amount_cents`) or a decimal type, and name the currency.

`references/data-and-api.md` has an example record and the idempotency contract; read it when the change touches a schema, an endpoint or a retried operation.

## 3. Comments and docstrings

Prefer clear code to clever code, and guard clauses to nested conditionals.

The default is no comment. A comment exists only for what the code cannot say, and before writing one, check four things:

1. It states a hidden constraint, an invariant, a unit or format, a workaround with its issue link, a keep-in-sync pointer, a non-obvious algorithm choice, or a `TODO` that says what would make it removable. It does not describe what the next line does, and it does not describe the edit or the task ("added", "fixed", "as requested"); that belongs in the commit message.
2. Budget: at most one inline comment per function. Needing a second means the function wants splitting or a better name, so fix that instead. The Arrange/Act/Assert markers in a test and the justification a suppression must carry (section 12) do not count against the budget and are exempt from check 3. Inline comment lines never outnumber code lines in any function.
3. It sits above the function or block, not interleaved line by line.
4. It is one line. A longer explanation is a docstring or a doc file.

No dividers, end-of-block markers or commented-out code.

Public or exported API gets a docstring in the language's own convention. Private helpers get none unless the logic is non-obvious. The summary line says what the caller gets and does not repeat the signature or its types; add only what the signature cannot say (a unit, an invariant, a failure mode).

A `TODO` for a deferred shortcut names the known limit and the trigger that upgrades it: `// TODO(owner): global lock; per-account locks when throughput matters`. For a reviewer finding judged a false positive, it says why the code is safe. `references/comments.md` has one example.

## 4. Small, single-purpose pieces

- A function is at most 40 lines. Past that, split it, or spend one line in the hand-off saying why it stays whole. A function that needs blank-line sections to stay readable has already crossed the limit. Split while writing, because a later splitting pass rarely happens.
- A new file has one responsibility, and its primary export's name states it.
- A helper with one caller exists only if its name carries a concept the reader needs. A wrapper that only forwards its arguments is inlined.
- New behaviour goes in the module that owns the data it touches. The caller gains a call, not a block.
- Abstract on the third occurrence, once you know what varies.

**A refactor must reduce the number of concepts a reader holds**, not move them. Prefer the change that makes a branch, mode or layer disappear. Before simplifying unfamiliar code, know its responsibility, its callers, its edge cases and why it might be written this way (check `git blame`). A simplification that needs test changes to pass probably changed behaviour ([[tdd]] section 9).

**Structural and behavioural changes do not travel in the same step.** Mixed together, a red suite cannot say whether the new behaviour is wrong or the restructuring broke something. Do the structural move first, on green; then the behaviour. If the suite goes red after a pure restructuring, revert it. Inside `/sonu:ship`'s review cycles, where each cycle's fixes land as one commit, answer a finding with the behavioural fix and leave the restructure as a follow-up.

A file near a thousand lines is a signal to decompose before adding to it. A mechanical refactor across hundreds of lines is done with a codemod or script, because hand edits at that scale are error-prone and hard to review.

## 5. Data access

Write every query as if the table is already large: select only the columns used, paginate by default, filter and aggregate in the database, avoid one query per row, and index the columns filtered and sorted on. A change made in order to be faster follows [[performance]], which measures first.

## 6. Presentation

No inline styles; style belongs in stylesheets, tokens or the project's styling system. The exception is a truly dynamic runtime value, and a CSS custom property is preferred even then. Name thresholds and status values instead of using magic numbers and strings.

## 7. Errors

Do not swallow errors. Catch only what you can handle, where there is enough context to decide.

**Write the failure path with the happy path**, because it is the path nobody exercises.

- Every acquire (a reservation, a lock, a marker, a slot) is released in a `finally`, or is not taken until after the last `await` that can reject.
- Every detached promise has a `catch` that logs. Without one, a downstream failure crashes the process.
- A remote write's transport timeout is ambiguous, never failed. The write may have landed, so the caller does not retry as if nothing happened, and the message to the user says so.

Before calling a change done, ask what fires two levels out when it runs (callbacks, middleware, observers), whether the tests exercise the real chain or only mocks, and whether a failure can leave orphaned state.

**A silent fallback is worse than an empty catch.** `catch { return [] }` or `?? defaultValue` on a failure path turns a crash that would be fixed today into wrong data that is trusted for months. A fallback is legitimate only as a deliberate product decision, and even then it logs the failure and is visibly a fallback.

**At a data boundary, a parse failure produces a signal.** When parsing external or cross-component data fails, log at error level with the raw input (truncated to a few hundred characters and redacted), increment an error metric, or rethrow. A default return is acceptable only alongside that signal. When you are the one changing what flows into someone else's parser, find the consumers first ([[blast-radius]]).

## 8. Logging

Log through one shared logger, not scattered `console.log` or `print` calls, so levels, format, redaction and routing have one home. Each line has a stable message plus structured key/value context, with values kept out of the message string. Attach a correlation or request id. Do not log secrets or personal data: log the `user_id`, not the user. The exception is a dedicated, access-controlled, short-retention audit log for security events.

## 9. Input validation

Every value crossing a trust boundary is untrusted; [[security]] covers which boundaries exist. Validate type, shape, length, range and allowed values on the server when input enters, with a schema validator and allow-lists. Do not re-validate between internal functions that share a type contract or on data read back from your own database, because validation everywhere buries the boundaries that matter. Parameterize queries, pass argument arrays to shells, and encode on output.

## 10. Do not leak information

Return generic errors and log the detail internally under a correlation id. A response does not carry a stack trace, SQL, an internal id or another user's data. Do not confirm which half of a credential pair was wrong; treat password reset and signup the same way, and keep `403` and `404` indistinguishable where existence itself is sensitive.

## 11. State

Prefer immutable values and the tightest scope.

## 12. When tooling objects, fix the cause

A type error, lint violation or failing hook is information. Suppression (`as any`, `@ts-ignore`, `eslint-disable`, `--no-verify`) is a last resort and does not ship bare: use the narrowest scope, one line, with a comment saying why and what would make it removable. The same holds for failing tests ([[tdd]] section 9) and runtime errors (section 7).

## 13. API design

- Build responses from an explicit allowlist of fields, not the ORM entity, which ships whatever the row contains now and later.
- Status codes are truthful: no `200` with an error body.
- One error envelope for every endpoint.
- Removing or renaming a response field is a staged migration, as in [[safe-migrations]]: add alongside, migrate consumers, retire deliberately.
- Retried operations need idempotency designed. Derive the key from the intent, not the attempt; claim it atomically with a unique constraint; and treat every outbound call as having three outcomes: success, failure and unknown. Record intent before calling out. `references/data-and-api.md` has the full contract.

## 14. Configuration and feature flags: absence must be safe

When a config value, env var or feature flag that gates behaviour is missing or unparseable, it resolves to the safe state, never silently to enabled. Absent config means the operator did not choose.

```js
// Avoid: a missing or misspelled env var turns the feature on
const newDashboardEnabled = process.env.NEW_DASHBOARD_ENABLED !== 'false';

// Prefer: only the explicit value enables
const newDashboardEnabled = process.env.NEW_DASHBOARD_ENABLED === 'true';
```

The test for which rule applies is whether running with the thing off would be unsafe. A feature gate (new UI, experiment, optimization) defaults off. A protective control (auth enforcement, rate limiting, TLS verification) is required config and fails fast at startup in production when unset, because off is not a safe state for a protection. A default of `true` is an explicit, commented decision at the definition site. Log the resolved values of behaviour-gating flags once at startup.

## 15. Dependencies

- Read the changelog, not the version number. A patch release can change behaviour.
- Upgrade one dependency per change, so a breakage has one suspect and a clean revert.
- Tests pass before and after. Thin coverage around the dependency's behaviour is the finding: add the test first.
- Review the lockfile diff, commit the lockfile, and do not hand-edit it.
- When you cannot verify a framework API against current documentation for the installed version, mark the code `UNVERIFIED: not checked against current docs for the installed version`.

## Reference files

| File | What it answers |
|------|-----------------|
| `references/data-and-api.md` | An example record (section 2) and the idempotency contract (section 13) |
| `references/comments.md` | One example of a comment and a docstring that pass the checks (section 3) |
