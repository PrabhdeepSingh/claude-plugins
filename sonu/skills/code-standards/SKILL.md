---
name: code-standards
description: Prabhdeep (Sonu) Singh's personal coding standards — the house rules and baseline quality bar for writing code the way he does. INVOKE whenever writing, generating, refactoring, or reviewing ANY code in ANY language — including database schema design, API endpoints, SQL/queries, logging, validation, and error handling — even when the user doesn't say "standards" or "style." If you're about to write or edit code, consult this first. (Pairs with [[tdd]], which covers red-green-refactor discipline; that skill assumes this one's bar.)
---

# Code Standards — write it like Sonu would

This is the house style. It exists because code is read far more often than it's written, and the reader is usually a tired human (or model) six months from now with no context. Every rule below optimizes for that reader. Follow the spirit, not just the letter — these are the instincts of someone who's maintained code for 20 years, not a linter config.

## How to apply this

Before writing new code, hold these standards in mind as the target. Before editing existing code, read the surrounding file first: **a consistent codebase beats a "correct" one.** If the existing code already has strong conventions, match them even where they differ from this guide, and mention the conflict rather than silently fighting it. Apply this guide fully when starting fresh, filling gaps, or when the existing code has no clear convention.

When you finish a change, run the self-check at the bottom against your own diff before considering it done.

## How you work — discipline before output

The numbered sections below describe what finished code should look like. These five habits describe how to get there, and they head off the most common ways an AI coding session goes off the rails: confidently building the wrong thing, over-engineering, re-implementing what already exists, and leaving collateral damage in the diff. They bias toward care over raw speed — on a genuinely trivial change, use judgment.

**Think before you code.** Don't assume your way past ambiguity — surface the options when a request has more than one reasonable reading, state the assumptions you're working from, and say so if a simpler approach exists. When something is genuinely unclear and a wrong guess would be expensive to unwind, stop and ask rather than building on the guess. That includes the requester's premises: if the request assumes something the codebase contradicts, surface the mismatch before implementing.

**Build the minimum that solves the problem.** Write the least code that fully does what was asked — no abstractions for single-use code, no configurability nobody requested, no error handling for cases that can't occur. (The rule of three from section 4 applies here too.) If a senior engineer would call it overcomplicated, simplify.

**Look before you write.** The most common way ten lines becomes two hundred is re-implementing something that already exists. Before writing any helper, utility, or algorithm, search in order: the codebase (grep the concept; read `utils`/`lib`/shared folders), the language's standard library, then the project's existing dependencies. Write it fresh only when that search comes up empty — generating new code is *easier* than reading existing code for a model, so the pull toward a fresh implementation is strong; resist it.

```js
// Avoid: hand-rolling what the platform (or a nearby helper) already does
function groupUsersByRole(users) {
  const grouped = {};
  for (const user of users) {
    if (!grouped[user.role]) grouped[user.role] = [];
    grouped[user.role].push(user);
  }
  return grouped;
}

// Prefer: found by checking the stdlib first
const usersByRole = Object.groupBy(users, user => user.role);
```

Two corollaries: **don't guess APIs** — verify a method/signature against the *installed* version, not memory, before calling anything beyond the language's core; and **the inverse of reuse is dependency discipline** — a new package is an architectural decision (maintenance, security surface, upgrade treadmill), so don't install one for what the stdlib or an existing dependency already does.

**Make surgical changes.** Touch only what the task requires. Resist "improving" adjacent lines or reformatting; match the surrounding style even where you'd do it differently. Clean up what *your* change orphaned, but flag out-of-scope dead code to the reviewer instead of deleting it. When an approach fails, revert it fully before trying the next one — a diff must never carry the fossils of abandoned attempts layered under the fix that finally worked.

**Turn the task into a verifiable goal, then loop.** Restate vague asks as something checkable: "fix the bug" becomes "write a test that reproduces it, then make it pass." For anything multi-step, sketch a plan with a verification check per part. And **claim only what you observed**: "tests pass" means you ran them and saw green this session — never assert an outcome you didn't watch happen. The full test-first discipline lives in [[tdd]].

---

## 1. Names carry the meaning

A name should tell the reader what something *is* or *does* without them having to go read its definition. Never ship these as real identifiers: `data`, `info`, `item`, `value`, `val`, `obj`, `temp`, `tmp`, `result`, `res`, `arr`, `list`, `thing`, `stuff`, `foo`, or a `Manager`/`Helper`/`Util`/`Service` suffix used as a junk drawer — they describe the variable's *type*, not its *purpose*, which the reader can already see.

Name for the domain, not the mechanism. Functions are verbs (`calculateInvoiceTotal`), variables/properties are nouns (`outstandingBalance`), booleans read as a yes/no assertion (`isExpired`, `hasUnpaidInvoices`). Avoid abbreviations unless genuinely universal (`id`, `url` are fine; `usrAcctBal` is not). Match the vocabulary the business already uses.

```js
// Avoid: the reader has to decode every name
const arr = users.filter(u => u.a);
let temp = 0;
for (const item of arr) temp += item.amt;

// Prefer: the names ARE the documentation
const activeUsers = users.filter(user => user.isActive);
let totalMonthlyRevenue = 0;
for (const user of activeUsers) totalMonthlyRevenue += user.subscriptionAmount;
```

## 2. Schema and API naming conventions

Data outlives code — a column or API field gets baked into the database and into other teams' integrations, so it's far more expensive to rename later than a local variable. Get these right up front and keep them consistent across the whole surface.

- **API/JSON and database field names are `snake_case`** (`first_name`, `is_active`) — consistent casing means consumers never guess `userId` vs `user_id` vs `UserID`. Map at the boundary in camelCase ecosystems (JS/TS) rather than leaking mixed casing into payloads.
- **Identifiers are UUIDs/GUIDs, not auto-increment integers** — sequential ids leak record counts, collide across merged systems, and are guessable in URLs. Name them `id` on their own table, `<entity>_id` as a foreign key.
- **Date/time fields read as `<thing>_date` or `<thing>_at`**, named for the event (`created_date`/`created_at`). Pick one convention per project, never mix. Store in UTC; let presentation localize.
- **Store names as separate fields** (`first_name`/`last_name`) — never a single `full_name` you have to split later; splitting is locale-hostile and lossy, composing from parts is always correct.
- **Booleans read as a question in data too** (`is_active`, `has_subscription`) so a row is self-describing.
- **Money is never a float** — binary floating point can't represent most decimal amounts exactly, and rounding compounds. Store integer minor units (`total_amount_cents`) or a decimal type, and name the currency.

→ `references/data-and-api.md` — a full example record and the API/status-code depth for §13, read when this change touches a database schema, a query, or an endpoint.

## 3. Write for the next human to read

Favor clarity over cleverness — a dense one-liner that takes five minutes to parse is worse than three obvious lines. Use guard clauses to keep the happy path flat instead of nesting conditionals, which bury the main logic inside a pyramid of braces.

```js
// Avoid: the real work is buried three levels deep
function getDiscount(user) {
  if (user) {
    if (user.isActive) {
      if (user.plan === 'pro') return 0.2;
    }
  }
  return 0;
}

// Prefer: handle the exits first, then the happy path reads straight down
function getDiscount(user) {
  if (!user || !user.isActive) return 0;
  if (user.plan !== 'pro') return 0;
  return 0.2;
}
```

Comments explain **why**, not **what** — the code already says what, and a comment restating the line below it is noise that drifts out of date. Keep `TODO`s actionable and attributed, with enough context that someone could actually act on them. Delete dead code instead of commenting it out; that's what version control is for.

## 4. Small, single-purpose, modular pieces

A function should do one thing at one level of abstraction — if you need "and" to describe it, it's three functions. **Concrete tripwire: a function pushing past ~30–40 lines, or one that needs blank-line "sections" to stay readable, is asking to be split.** It's a heuristic, not a law, but once you cross it the burden of proof flips: justify keeping it whole rather than assuming it's fine.

Separate concerns by layer — business logic doesn't belong in UI components; data access doesn't belong in controllers or views. Prefer pure functions (same input, same output, no hidden side effects) wherever the work is a calculation, and push side effects to the edges. Don't repeat yourself, but don't abstract prematurely either — reach for a shared abstraction on the third occurrence (the rule of three), once you know what actually varies.

## 5. Data access: ask for exactly what you need

Queries are where code meets scale — something instant against 50 rows in development can take down production at 50,000. Write every query as if the table is already huge.

- **Select only the columns you actually use** — `SELECT *` drags every column over the wire, silently changes behavior when a column is added, and hides what the code truly depends on.
- **Paginate by default; never load an unbounded result set into memory.** Bulk pulls (exports, batch jobs) are fine only when deliberate, bounded, and streamed/chunked.
- **Filter and aggregate in the database, not application code** — `COUNT`/`SUM`/`WHERE` is orders of magnitude faster than pulling rows back to loop over.
- **Avoid N+1 queries** — fetching a list then firing one more query per row turns one round trip into a thousand; use a join or a batched `IN (...)`.
- **Index the columns you filter and sort on.**

→ `references/data-and-api.md` — worked SQL and ORM examples, read when this change touches a query.

## 6. Keep presentation, logic, and content separate

No inline styles — style belongs in stylesheets, tokens, or the project's styling system. Inline styles can't be reused, themed, or respond to state cleanly. The one exception is a genuinely dynamic runtime value (prefer a CSS custom property even then).

```jsx
// Avoid
<div style={{ color: '#3a3a3a', padding: '16px' }}>...</div>
// Prefer: the look lives in the stylesheet / token system
<div className="card-body">...</div>
```

The same instinct applies to magic numbers and strings buried in logic — `if (status === 3)` tells the reader nothing; `if (status === OrderStatus.Shipped)` tells them everything. Name thresholds and config values instead of sprinkling them inline.

## 7. Fail loudly and handle errors honestly

Don't swallow errors — an empty `catch {}` turns a bug into a silent mystery. Catch only what you can actually handle, handle it where there's enough context to decide, and otherwise let it propagate. Validate inputs at the system's boundaries so the core can trust what it's working with.

**The silent fallback is the polished version of the empty catch — and it's worse.** `catch { return [] }` or `?? defaultValue` on a failure path *looks* like robustness, but it converts a crash (loud, findable, fixed today) into wrong data (silent, trusted, discovered months later). A fallback is legitimate only when the degraded behavior is a deliberate *product decision*, and even then it logs the failure and is visibly a fallback.

**At a data boundary, a parse failure must produce a signal.** A parser, deserializer, or extractor that catches its own failure and returns `null`/empty converts breakage into silently missing data — everything downstream keeps running, the dashboards it feeds go quietly wrong, and nothing pages. When parsing external or cross-component data fails, emit a signal: log at error level with the raw input (truncated to a few hundred characters — enough to diagnose, not a payload dump — and redacted per §8), increment an error metric, or rethrow. A default return is acceptable only *alongside* that signal, never instead of it. And the mirror duty when you're the one changing what flows into someone else's parser: enumerate the consumers first — that's [[blast-radius]].

→ `references/security.md` — the empty-dashboard example, read when this change adds an error-handling or fallback path.

## 8. Logging: through one helper, never raw `console.log`

Logs are how you understand a system you can't step through — in production there's no debugger, only the trail you left. Route everything through a single shared logger (or a thin wrapper) — never scattered `console.log`/`print` calls — for one place to set levels, format, redact secrets, and route output. Every line carries a **stable, scannable message** plus structured key/value context — never values mashed into the message string, which makes logs ungreppable.

- **Use the right level so noise stays filterable** — `debug` for local detail, `info` for milestones, `warn` for recoverable oddities, `error` for failures needing a human.
- **Attach a correlation/request id** to every line so one request's dozens of log lines can be traced across a concurrent stream.
- **Never log secrets or PII** — log the `user_id`, not the user. The narrow exception is a dedicated, access-controlled, short-retention audit log for security events (failed logins, permission changes) — everywhere else, the rule stands.

## 9. Trust nothing from the outside: validate inputs, parameterize queries

Every value crossing a trust boundary — request body, query string, headers, third-party responses — is hostile until proven otherwise. Two defenses, always together:

**Validate at the boundary, before the value touches any logic.** Check type, shape, length, range, and allowed values the moment input enters; prefer allow-lists over deny-lists; validate on the *server* even when the client already did (client checks are UX, not security). Use a schema validator (zod, Joi, pydantic) rather than a pile of hand-rolled `if`s.

**Never build a query or command by concatenating input — parameterize.** A parameterized query sends SQL and values over separate channels, so a malicious value is treated as data, never executed. The same instinct extends to shells (pass an argument array), `eval`, file paths (traversal), and HTML output (XSS — encode on output).

→ `references/security.md` — the SQL-injection and schema-validation examples, read when this change handles external input or a database query.

## 10. Never leak sensitive information

Assume every response and error message will be read by someone trying to break in. **Return generic errors; log the detail internally** — a response should never carry a stack trace, SQL, an internal id, or another user's data; log the full detail keyed by a correlation id instead.

**Don't let messages become a recon tool.** Never confirm which half of a credential pair was wrong ("no account with that email" vs "incorrect password") — one message for the pair lets an attacker learn nothing from the response. Apply the same thinking to password reset and signup, and keep `403`/`404` indistinguishable wherever existence itself is sensitive.

→ `references/security.md` — the login-enumeration example, read when this change touches auth or an existence check.

## 11. State and side effects

Prefer immutability by default — return new values rather than mutating shared state in place; shared mutable state is the source of most "how did it get into THAT state" bugs. Keep variable scope as tight as possible.

## 12. When the tooling objects, fix the cause — don't silence the messenger

A type error, lint violation, or failing pre-commit hook is the toolchain telling you something is wrong. Suppressing it (`as any`, `@ts-ignore`, `eslint-disable`, `--no-verify`) makes the message go away and leaves the problem in place, now invisible — which is exactly why it's the most tempting shortcut under pressure.

**Suppression is a last resort, never a first response, and it never ships bare.** Work the actual cause first. If you judge the diagnostic genuinely wrong, suppress at the *narrowest possible scope* (one line, never a file or rule globally) with a comment stating why and what would make it removable. The same principle governs failing tests ([[tdd]]'s "the test is innocent" rule) and runtime errors (§7): don't make the signal go away without addressing what it signals.

## 13. API design: the response is a contract, not a data dump

An endpoint's response gets baked into clients you don't control, so what you return you maintain forever, and what you leak you can't unleak.

- **Serialize an allowlist, never the entity.** `res.json(user)` ships whatever the ORM row contains — password hashes, internal flags, columns added next year by someone who never saw this endpoint. Build responses from an explicit allowlist (DTO, serializer, projection) naming exactly what the client needs — including *related* data, which can drag another entity's private fields along.
- **Status codes tell the truth** — never `200` with an error body; a lying status breaks retries, caching, and monitoring.
- **One error shape, everywhere** — every error response uses the same envelope, or every client writes its own parser.
- **Breaking a response is a migration** — remove or rename a field the same staged way as [[safe-migrations]]: add alongside, migrate consumers, retire deliberately, never in one step.

→ `references/data-and-api.md` — the allowlist example, full status-code list, and migration detail, read when this change touches an endpoint's request or response shape.

## 14. Configuration and feature flags: absence must be safe

When a config value, env var, or feature flag that gates behavior is missing or unparseable, it resolves to the **safe state — never silently to enabled.** For a feature gate the safe state is off; for a protective control no accidentally-reached state is safe, so it fails fast instead (criterion below). Absent config means the operator never chose; silently activating a feature (or silently running *without* a protection) makes the system behave differently from what its operator believes, with no signal — which is exactly how critical systems break with nobody noticing. A default of `true` must be an explicit, written decision at the definition site, with a comment saying it's deliberate — never the accident of a fallback expression.

```js
// Avoid: a missing or misspelled env var silently turns the feature ON
const newDashboardEnabled = process.env.NEW_DASHBOARD_ENABLED !== 'false';

// Prefer: absent or malformed resolves to off; only the explicit value enables
const newDashboardEnabled = process.env.NEW_DASHBOARD_ENABLED === 'true';
```

Two corollaries: config that is *required* for correct operation fails fast at startup rather than defaulting at all; and the resolved values of behavior-gating flags are logged once at startup (through the §8 logger) so what's actually running is observable, not assumed.

The criterion for which corollary applies: **would running with this thing off be unsafe?** A *feature* gate (a new UI, an experiment, an optimization) defaults off — absence is safe. A *protective control* (auth enforcement, rate limiting, TLS verification) is required config: in production it fails fast at startup when unset, because "off" isn't a safe state for a protection — it's the outage waiting to be discovered. The one thing absence must never do, in either case, is silently choose "on."

---

## Self-check before you call it done

Run this against your own diff. If any answer is "no," fix it before finishing:

- Assumptions surfaced up front (not silently guessed), and is this the minimum that solves the problem?
- Every changed line traces to the request — no unrequested refactors, no fossils of abandoned attempts, no debugging debris (`console.log`/`print`, `debugger`, temp scripts, commented-out experiments)?
- Zero new bare suppressions (narrowest scope + justifying comment on any that remain), and every claim in your report something you actually observed this session?
- Searched the codebase, stdlib, and existing dependencies before writing any new helper or algorithm?
- Could a new teammate guess every name's meaning — no generic `data`/`temp`/`Manager` survivors — and do schema/API names follow convention (`snake_case`, UUID ids, UTC `created_date`/`*_at`, first/last name split)?
- Every function does one thing at one abstraction level (past the ~30–40-line tripwire → split or justified), with the happy path flat via guard clauses, not nested `if`s?
- Every query selects only needed columns and bounds its result set — filtering/counting in the DB, no N+1?
- Every external input validated against a schema at the boundary — server-side, allow-lists over deny-lists, client checks treated as UX only; all SQL parameterized (never concatenated); shell/`eval`/path/HTML sanitized or encoded?
- Errors handled with context, never silently swallowed or defaulted — every parse failure at a data boundary produces a signal (error log, metric, or rethrow), never a bare default; API failures return a generic message (detail logged internally); auth/lookup responses avoid revealing existence?
- Every behavior-gating config/env/flag resolves to the safe state when missing or malformed — feature gates off, protective controls failing fast, any `true` default an explicit commented decision — and resolved flags are logged once at startup?
- Every API response built from an explicit allowlist, with honest status codes, one error shape, and pagination on lists?
- Logging through the shared logger — right level, stable scannable message, structured context, correlation id, zero secrets/PII?
- Presentation, logic, and data access separated (zero inline styles, zero magic numbers/strings); comments explain *why* only (no commented-out code, no restating the line below); and the change matches the file's existing conventions?

These aren't bureaucracy — each one is a thing that bites the next person to open the file. Leave the code so the next reader thanks you.

## Reference files

| File | What it answers |
|------|-----------------|
| `references/data-and-api.md` | A full example record (§2), worked SQL/ORM query examples (§5), and the API allowlist example, status-code list, and migration detail (§13) |
| `references/security.md` | The silent-fallback example (§7), SQL-injection and boundary-validation examples (§9), and the login-enumeration example (§10) |
