---
name: code-standards
description: Prabhdeep (Sonu) Singh's personal coding standards — the house rules for writing code the way he does. Covers working discipline (including reuse-before-reimplementing), intention-revealing naming, schema/API conventions (snake_case fields, UUID ids, UTC timestamps), readability and guard clauses, modularity and function-size limits, separation of concerns, safe data access (pagination, no N+1), presentation/logic separation, error handling, logging, input validation and injection defense, information-leak prevention, state/immutability, and API design (response allowlisting — never serialize a raw entity — honest status codes, one error shape, pagination, staged deprecation). Use this skill whenever writing, generating, refactoring, or reviewing ANY code in ANY language — including database schema design, API endpoint design, SQL/queries, logging, validation, and error handling — even when the user doesn't mention "standards" or "style." It defines the baseline quality bar for all code produced here. If you're about to write or edit code, consult this first. (Pairs with the [[tdd]] skill, which covers the full red-green-refactor discipline.)
---

# Code Standards — write it like Sonu would

This is the house style. It exists because code is read far more often than it's written, and the reader is usually a tired human (or model) six months from now with no context. Every rule below optimizes for that reader. Follow the spirit, not just the letter — these are the instincts of someone who's maintained code for 20 years, not a linter config.

## How to apply this

Before writing new code, hold these standards in mind as the target. Before editing existing code, read the surrounding file first: **a consistent codebase beats a "correct" one.** If the existing code already has strong conventions, match them even where they differ from this guide, and mention the conflict rather than silently fighting it. Apply this guide fully when starting fresh, filling gaps, or when the existing code has no clear convention.

When you finish a change, run the self-check at the bottom against your own diff before considering it done.

## How you work — discipline before output

The numbered sections below describe what finished code should look like. These five habits describe how to get there, and they head off the most common ways an AI coding session goes off the rails: confidently building the wrong thing, over-engineering, re-implementing what already exists, and leaving collateral damage in the diff. They bias toward care over raw speed — on a genuinely trivial change, use judgment.

**Think before you code.** Don't assume your way past ambiguity. If a request has more than one reasonable reading, surface the options instead of silently picking one. State the assumptions you're working from, and if a simpler approach exists than the one asked for, say so. When something is genuinely unclear and a wrong guess would be expensive to unwind, stop and ask rather than building on the guess. A clarifying question up front is cheap; a confident wrong implementation is not. That includes the requester's premises: if the request assumes something the codebase contradicts ("just update the config flag" when no such flag exists), surface the mismatch before implementing — confident phrasing doesn't make it true.

**Build the minimum that solves the problem.** Write the least code that fully does what was asked, and nothing speculative — no abstractions for single-use code, no configurability nobody requested, no error handling for cases that can't occur. (The rule of three from section 4 applies here too: don't generalize until you've actually seen the repetition.) If a senior engineer would call it overcomplicated, it is. Simplify.

**Look before you write.** The most common way ten lines becomes two hundred is re-implementing something that already exists. Before writing any helper, utility, or algorithm, search in this order: the codebase (grep for the concept; read the `utils`/`lib`/shared folders), the language's standard library, then the project's existing framework and dependencies. Write it fresh only when that search comes up empty. This matters doubly for a model: generating new code is *easier* than reading existing code, so the pull is always toward a fresh implementation — resist it, because the codebase then carries two versions and the next reader inherits both.

Two corollaries. **Don't guess APIs**: when calling anything beyond the language's core — especially a less-common library — verify the method, signature, and options against the *installed* version (the types in `node_modules`, the pinned version's docs, `pip show`), not against memory; a plausible-looking call to an API that doesn't exist is a classic AI-authored bug. And **the inverse of reuse is dependency discipline**: don't install a new package for what the stdlib or an existing dependency already does. A new dependency is an architectural decision — a maintenance obligation, a security surface, an upgrade treadmill — and needs stating why the existing options don't suffice, not a reflex `npm install`.

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

**Make surgical changes.** Touch only what the task requires. When editing existing code, resist the urge to "improve" adjacent lines, reformat, or refactor things that aren't broken — match the surrounding style even where you'd personally do it differently. Every changed line should trace directly back to the request. Clean up the imports and variables *your* change orphaned, but don't go out of scope to delete pre-existing dead code unrelated to your task — flag it to the reviewer instead. (When dead code is in scope, delete it rather than commenting it out — section 3.) This keeps diffs reviewable and keeps your change from quietly breaking something unrelated. The same discipline applies to your own dead ends: **when an approach fails, revert it fully before trying the next one.** A diff must not carry the fossils of abandoned attempts — a stray import from approach A, a half-wired helper from approach B — layered under the fix that finally worked.

**Turn the task into a verifiable goal, then loop.** Restate vague asks as something you can actually check: "add validation" becomes "write tests for the invalid inputs, then make them pass"; "fix the bug" becomes "write a test that reproduces it, then make it pass." For anything multi-step, sketch a short plan with a verification check for each part. Strong success criteria are what let you work independently to the finish line instead of stopping every few minutes to ask whether you're on track. And **claim only what you observed**: "tests pass" means you ran them in this session and saw green — never assert an outcome you didn't watch happen, and when reporting a result, show the command and its actual output. An unverified "this should work now" costs the reader more than an honest "I couldn't run this because X." The full discipline — red-green-refactor, what to test, how to name tests, when to mock — lives in [[tdd]].

---

## 1. Names carry the meaning

A name should tell the reader what something *is* or *does* without them having to go read its definition. Generic names force that lookup every single time, so they're banned.

Never ship these as real identifiers: `data`, `info`, `item`, `value`, `val`, `obj`, `temp`, `tmp`, `result`, `res`, `arr`, `list`, `thing`, `stuff`, `foo`, or a `Manager`/`Helper`/`Util`/`Service` suffix used as a junk drawer for unrelated functions. They describe the variable's *type*, not its *purpose*, which the reader can already see.

Name for the domain, not the mechanism. Functions are verbs (`calculateInvoiceTotal`), variables and properties are nouns (`outstandingBalance`), booleans read as a yes/no assertion (`isExpired`, `hasUnpaidInvoices`, `canRetry`). Avoid abbreviations unless they're genuinely universal in the domain (`id`, `url`, `http` are fine; `usrAcctBal` is not). Match the vocabulary the business already uses — if the team says "shipment," don't invent "parcel."

**Example — loop and accumulator:**
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

**Example — function and its argument:**
```js
// Avoid
function process(data) { /* ... */ }

// Prefer: name says what it does and what it expects
function archiveCompletedOrders(orders) { /* ... */ }
```

## 2. Schema and API naming conventions

Data outlives code. A column name or API field gets baked into the database, into other teams' integrations, into client apps you don't control — so it's far more expensive to rename later than a local variable. Get these right up front and keep them consistent across the whole surface.

- **API/JSON and database field names are `snake_case`** (`first_name`, `total_amount`, `is_active`). Consistent casing across the wire means consumers never have to guess whether it's `userId`, `user_id`, or `UserID`. In languages whose ecosystem expects camelCase (JS/TS), map at the boundary rather than leaking mixed casing into your payloads.
- **Identifiers are UUIDs/GUIDs, not auto-increment integers.** Sequential integer ids leak how many records you have, collide when you merge data across systems or environments, and are guessable in URLs. UUIDs are safe to expose, safe to generate client-side, and safe to merge. Name them `id` on their own table and `<entity>_id` as a foreign key (`customer_id`, `order_id`).
- **Date and time fields read as `<thing>_date` or `<thing>_at`** and are named for the event, not the type: `created_date`, `last_modified_date` (or `created_at`, `updated_at`). Pick one of those two conventions per project and never mix them. Store timestamps in UTC and let the presentation layer localize; a naive local-time timestamp is a bug waiting for a daylight-saving boundary.
- **Store names as separate fields** — `first_name` and `last_name` (or `given_name`/`family_name`), never a single `full_name` you have to split. Splitting a full name is locale-hostile and lossy (middle names, two-word surnames, ordering all break). Composing a display name from parts is trivial and always correct; decomposing one never is.
- **Booleans read as a question in data too** — `is_active`, `has_subscription`, `can_refund` — so a row is self-describing.
- **Money is never a float.** Binary floating point can't represent most decimal amounts exactly (`0.1 + 0.2 !== 0.3`), and rounding errors compound. Store integer minor units (`total_amount_cents`) or a decimal type, and say the currency (a field or a documented convention).

**Example — a well-named record:**
```json
{
  "id": "9f1c2e7a-3b44-4e8d-9a2f-6c5d4e3b2a10",
  "customer_id": "1d2f6b8c-0a11-4c33-8e55-77a9b0c1d2e3",
  "first_name": "Ada",
  "last_name": "Lovelace",
  "is_active": true,
  "total_amount_cents": 14900,
  "currency": "USD",
  "created_date": "2026-05-01T14:32:00Z",
  "last_modified_date": "2026-05-03T09:10:00Z"
}
```

## 3. Write for the next human to read

Favor clarity over cleverness. A dense one-liner that takes five minutes to parse is worse than three obvious lines. The compiler doesn't care; the reader does.

Use guard clauses to keep the happy path flat instead of nesting conditionals. Deep nesting hides the main logic inside a pyramid of braces.

```js
// Avoid: the real work is buried three levels deep
function getDiscount(user) {
  if (user) {
    if (user.isActive) {
      if (user.plan === 'pro') {
        return 0.2;
      }
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

Comments explain **why**, not **what** — the code already says what. A comment that restates the line below it is noise that drifts out of date. Save comments for the non-obvious: a workaround, a business reason, a deliberate edge case. Delete dead code instead of commenting it out; that's what version control is for. Keep `TODO`s actionable and attributed with enough context that someone could actually act on them.

## 4. Small, single-purpose, modular pieces

A function should do one thing at one level of abstraction. If you need "and" to describe it (`validateAndSaveAndNotify`), it's three functions. Small focused pieces are easier to name, test, reuse, and reason about — and they make diffs reviewable.

**Concrete tripwire: a function pushing past ~30–40 lines, or one that needs blank-line "sections" or `// --- step 2 ---` comments to stay readable, is asking to be split.** Those section markers are the function telling you its parts already have names — extract them. This is a heuristic, not a law (a flat mapping table or exhaustive `switch` can legitimately run long), but once you cross it, the burden of proof flips: justify keeping it whole rather than assuming it's fine. "It all fits on my screen" is not a justification.

Separate concerns by layer. Business logic doesn't belong in UI components; data access (SQL, fetch calls) doesn't belong in controllers or views. Keep a clean seam between "what the app decides" and "how it's shown" and "where it's stored," so each can change without dragging the others along.

Prefer pure functions — same input, same output, no hidden side effects — wherever the work is a calculation. They're trivial to test and impossible to surprise you. Push side effects (I/O, mutation, network) to the edges.

Don't repeat yourself, but don't abstract prematurely either. Two similar blocks are fine; reach for a shared abstraction on the third occurrence (the rule of three), once you actually know what varies. A wrong abstraction is more expensive than a little duplication.

## 5. Data access: ask for exactly what you need

Queries are where code meets scale. Something that's instant against 50 rows in development can take down production at 50,000. Write every query as if the table is already huge, because one day it will be.

- **Select only the columns you actually use.** `SELECT *` drags every column over the wire (including the big `description` blob you don't need), silently changes behavior when someone adds a column, and hides what the code truly depends on. Name the columns so the query documents its own footprint.
- **Paginate by default; never load an unbounded result set into memory.** Use `LIMIT`/`OFFSET` or, better at scale, keyset/cursor pagination. Pulling thousands of records at once is acceptable *only* when it's a deliberate, bounded part of the feature — an export, a scheduled batch job — and even then, stream or chunk it rather than holding it all in memory.
- **Filter and aggregate in the database, not in application code.** Don't pull 10,000 rows back to count or sum them in a loop — `COUNT`/`SUM`/`WHERE` is what the database is for, and it's orders of magnitude faster.
- **Avoid N+1 queries.** Fetching a list and then firing one more query per row turns one round trip into a thousand. Use a join or a batched `IN (...)` fetch.
- **Index the columns you filter and sort on**, and be aware of which queries rely on those indexes.

**Example — SQL:**
```sql
-- Avoid: every column, every row, straight into memory
SELECT * FROM orders;

-- Prefer: only the fields used, filtered and bounded
SELECT id, customer_id, total_amount, created_date
FROM orders
WHERE status = 'open'
ORDER BY created_date DESC
LIMIT 50 OFFSET :page_offset;
```

**Example — ORM:**
```js
// Avoid: loads every column of every user
const users = await User.findAll();

// Prefer: just the fields you need, one page at a time
const users = await User.findAll({
  attributes: ['id', 'email', 'last_login_date'],
  where: { is_active: true },
  order: [['created_date', 'DESC']],
  limit: 50,
  offset: pageNumber * 50,
});
```

## 6. Keep presentation, logic, and content separate

No inline styles. Style belongs in stylesheets, design tokens, or the project's styling system (CSS modules, styled-components, Tailwind classes, a theme file) — never hardcoded onto the element. Inline styles can't be reused, can't be themed, can't respond to state cleanly, and quietly override everything else. The one exception is a genuinely dynamic value that must be computed at runtime (e.g. a progress-bar width from a variable), and even then prefer a CSS custom property over a literal.

```jsx
// Avoid
<div style={{ color: '#3a3a3a', padding: '16px', fontSize: '14px' }}>...</div>

// Prefer: the look lives in the stylesheet / token system
<div className="card-body">...</div>
```

The same instinct applies everywhere: no magic numbers or magic strings buried in logic. Give them names. `if (status === 3)` tells the reader nothing; `if (status === OrderStatus.Shipped)` tells them everything. User-facing strings, config values, and thresholds belong in named constants or config, not sprinkled inline.

## 7. Fail loudly and handle errors honestly

Don't swallow errors. An empty `catch {}` turns a bug into a silent mystery that surfaces somewhere unrelated. Catch only what you can actually handle, handle it at the level that has enough context to decide, and otherwise let it propagate. Validate inputs at the system's boundaries (API edges, user input, external data) so the core can trust what it's working with. Error messages should give the reader enough to act — what failed and ideally why.

**The silent fallback is the polished version of the empty catch — and it's worse.** `catch { return [] }` or `?? defaultValue` on a failure path *looks* like robustness, but it converts a crash (loud, findable, fixed today) into wrong data (silent, trusted, discovered months later in a report someone acted on). A fallback is legitimate only when the degraded behavior is a *product decision* — "show cached prices if the pricing service is down" — and even then it logs the failure and is visibly a fallback. When you find yourself adding one just to make an error go away, you're not handling the error; you're hiding it from the person who needs to know.

```js
// Avoid: a failure becomes an empty dashboard that everyone believes
async function getMonthlyRevenue(customerId) {
  try {
    return await revenueApi.fetch(customerId);
  } catch {
    return 0;   // "robust" — and now a down API reads as zero revenue
  }
}

// Prefer: handle what's decided, propagate what isn't
async function getMonthlyRevenue(customerId) {
  return revenueApi.fetch(customerId); // caller decides; the error stays loud
}
```

## 8. Logging: through one helper, never raw `console.log`

Logs are how you understand a system you can't step through — in production there's no debugger, only the trail you left. So logging goes through a single shared logger (the project's logging library, or a thin wrapper around it), never scattered `console.log` / `print` / `System.out` calls. Every language has the equivalent: `logging`/`structlog` in Python, `ILogger`/Serilog in .NET, `slog` in Go, SLF4J in Java. The reason for funneling everything through one helper is leverage — one place to set levels, one consistent format, one place to attach shared context, one place to route output (stdout, file, aggregator), and one place to redact secrets. Scattered prints give you none of that and can't be turned off.

Make logs human-readable and filterable. A log line should be scannable by a tired person at 2am and greppable by a machine. That means a **level**, a **stable message** that reads like a sentence, and **structured context** as key/value fields rather than values mashed into the string:

- **Use the right level so noise stays filterable.** `debug` for local detail, `info` for normal milestones, `warn` for recoverable oddities, `error` for failures that need a human. Logging everything at `error` makes the real errors impossible to find.
- **Attach a correlation/request id** to every line so you can trace one request across the dozens of lines it produces. Without it, concurrent requests interleave into noise.
- **Never log secrets or sensitive data** — passwords, tokens, API keys, full card numbers, and avoid dumping whole objects that might carry them. Log the `user_id`, not the user. The one principled exception is **security/audit events** (failed logins, permission denials, account changes): these legitimately need identifying detail like the attempted email to investigate abuse and support real users — but they belong in a **dedicated auth/audit log that is access-controlled and short-retention**, not the general application log that fans out to every dashboard. Keep that carve-out narrow; everywhere else, the rule stands.

**Example:**
```js
// Avoid: unleveled, unfilterable, and it dumps the whole user object (PII, maybe a password hash)
console.log('got user', user);

// Prefer: shared logger, a level, a scannable message, structured context, no secrets
logger.info('User signed in', { user_id: user.id, request_id: req.id });
```

## 9. Trust nothing from the outside: validate inputs, parameterize queries

Every value that crosses a trust boundary — request body, query string, route params, headers, uploaded files, third-party API responses — is hostile until proven otherwise. Most catastrophic bugs (data deleted, tables dropped, servers run as someone else's code) come from code that treated input as safe. Two defenses, always used together.

**Validate at the boundary, before the value touches any logic.** Check type, shape, length, range, and allowed values the moment input enters, and reject what doesn't conform immediately. Prefer allow-lists (accept known-good) over deny-lists (block known-bad) — you can enumerate what's valid, you can never enumerate every attack. Validate on the *server* even when the client already validated; client-side checks are a UX nicety anyone can bypass, not a security control. Use a schema validator (zod, Joi, pydantic, FluentValidation, bean validation) so the rules are declarative and complete rather than a pile of hand-rolled `if`s that miss a case.

**Never build a query or command by concatenating input. Parameterize.** A parameterized query / prepared statement sends the SQL and the values over separate channels, so a value like `'; DROP TABLE users;--` is treated as data and never executed — that's the whole SQL-injection defense, and it's non-negotiable. The same instinct applies past SQL: never hand unsanitized input to a shell (command injection — pass an argument array, not a shell string), to `eval`, to a file path (path traversal), or into an HTML page (XSS — encode on output). Reach for the framework's safe primitive every time.

**Example — SQL:**
```js
// Avoid: input concatenated into SQL — `email` could be '; DROP TABLE users;--
const account = await db.query(
  `SELECT id, email FROM users WHERE email = '${email}'`
);

// Prefer: parameterized — the value is data, never executable SQL
const account = await db.query(
  'SELECT id, email FROM users WHERE email = $1',
  [email]
);
```

**Example — validate at the boundary:**
```js
// Avoid: trusting the body's shape and types, straight into the table
function createUser(req) {
  return db.insert('users', req.body);
}

// Prefer: validate against a schema, reject what doesn't fit, then use only known fields
const NewUser = z.object({
  email: z.string().email(),
  first_name: z.string().min(1).max(100),
  last_name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(130).optional(),
});

function createUser(req) {
  const parsed = NewUser.safeParse(req.body);
  if (!parsed.success) return badRequest('Invalid input');
  const { email, first_name, last_name, age } = parsed.data;
  return db.insert('users', { email, first_name, last_name, age });
}
```

This pairs with the section below: when validation rejects something, return a generic "invalid input" to the caller and log the specifics internally — a detailed validation error is itself a hint an attacker can probe with.

## 10. Never leak sensitive information

Assume every API response and error message will be read by someone trying to break in. Two things leak: what you *return*, and what your behavior *reveals*.

**Return generic errors; log the detail internally.** A response should never carry a stack trace, SQL, an internal id, a raw exception message, or another user's data. On failure, return a stable, generic message — ideally with a correlation id — and log the full detail internally (section 8) keyed by that id, so you can troubleshoot without handing an attacker a map of your internals.

**Don't let messages become a recon tool.** The textbook case is login: never say "incorrect password" or "no account with that email." Each tells an attacker which half they got right and lets them enumerate which emails have accounts. Return one message for the pair — "Email or password is incorrect." Apply the same thinking to password reset and signup: respond identically whether or not the account exists ("If that email is registered, we've sent a reset link"), so the response can't be used to discover who's a user. Anywhere there's an existence or authorization check, keep the `not found` and `forbidden` shapes (and ideally timing) indistinguishable, so the caller can't probe for what exists or what they're not allowed to see.

**Example:**
```js
// Avoid: confirms the email is valid, so an attacker now only has to brute-force the password
if (!user)            return res.status(404).json({ error: 'No account found for that email' });
if (!passwordValid)   return res.status(401).json({ error: 'Incorrect password' });

// Prefer: one generic message for the pair; the real reason is logged, not returned
if (!user || !passwordValid) {
  // Full email is OK *here* because authLog is access-controlled with short
  // retention — the audit-event exception in section 8, not the general app log.
  authLog.warn('Failed login attempt', {
    request_id: req.id,
    reason: user ? 'bad_password' : 'no_account',
    email_attempted: email,
  });
  return res.status(401).json({ error: 'Email or password is incorrect' });
}
```

## 11. State and side effects

Prefer immutability by default — return new values rather than mutating shared state in place. Shared mutable state is the source of most "how did it get into THAT state" bugs. Keep variable scope as tight as possible; declare things close to where they're used, not at the top of a giant function.

## 12. When the tooling objects, fix the cause — don't silence the messenger

A type error, a lint violation, a compiler warning, a failing pre-commit hook: each one is the toolchain telling you something is wrong. Suppressing it — `as any`, `@ts-ignore`, `eslint-disable`, `# type: ignore`, `@SuppressWarnings`, `--force`, `--no-verify` — makes the message go away and leaves the problem in place, now invisible. Under pressure this is the single most tempting shortcut, which is exactly why it's the one to refuse: the suppression outlives the deadline, and the next reader inherits code the tooling can no longer protect.

The rule: **suppression is a last resort, never a first response, and it never ships bare.** Work the actual cause first — fix the type, restructure the code, satisfy the rule. If, after that, you judge the diagnostic to be genuinely wrong (a known false positive, a library with broken types), suppress at the *narrowest possible scope* (one line, never a file or a rule globally) and attach a comment stating why it's a false positive and what would make the suppression removable. A bare `@ts-ignore` with no reason is a defect, not a fix. (The same principle governs failing tests — [[tdd]]'s "the test is innocent" rule — and runtime errors, section 7. The pattern is one and the same: don't make the signal go away without addressing what it signals.)

## 13. API design: the response is a contract, not a data dump

An endpoint's response gets baked into clients you don't control — so what you return, you maintain forever, and what you leak, you can't unleak. Four rules:

**Serialize an allowlist, never the entity.** The single worst API habit is `res.json(user)` — whatever the ORM row happens to contain goes over the wire: password hashes, reset tokens, internal flags, soft-delete markers, columns added next year by someone who never saw this endpoint. Responses are built from an explicit allowlist (a DTO, a serializer, a projection) naming exactly the fields the client needs. This is the response-side twin of section 9's input validation: allowlist in, allowlist out. The same applies to *related* data — embedding `order.customer` can drag another entity's private fields along.

```js
// Avoid: returns whatever the table contains, today and forever
app.get('/api/users/:id', async (req, res) => {
  const user = await User.findByPk(req.params.id);
  res.json(user); // password_hash, mfa_secret, internal_notes — all shipped
});

// Prefer: the response shape is explicit, minimal, and stable
app.get('/api/users/:id', async (req, res) => {
  const user = await User.findByPk(req.params.id, {
    attributes: ['id', 'first_name', 'last_name', 'created_date'], // §5: select what you need
  });
  if (!user) return res.status(404).json({ error: 'Not found' });
  res.json({
    id: user.id,
    first_name: user.first_name,
    last_name: user.last_name,
    created_date: user.created_date,
  });
});
```

**Status codes tell the truth.** Never `200` with `{ "error": ... }` in the body — machines branch on the status code, and a lying 200 breaks retries, caching, monitoring, and every client that trusted it. Use the boring standards: `201` created, `400` invalid input, `401` unauthenticated, `403` forbidden, `404` not found, `409` conflict, `422` semantically invalid, `5xx` our fault. Remember section 10: keep `403` vs `404` indistinguishable where existence itself is sensitive.

**One error shape, everywhere.** Every error response uses the same envelope (a stable machine-readable code, a human message, a correlation id) — and section 10 governs what goes *in* it: generic message out, detail logged internally. Ten endpoints with ten error formats means every client writes ten parsers.

**Breaking a response is a migration.** Removing or renaming a field, changing a type, tightening semantics — clients break exactly like the database clients in [[safe-migrations]], and the fix is the same staged path: add the new field alongside the old, migrate consumers, retire the old one deliberately (deprecation header/date, then removal) — never in one step. Additive changes are free; that asymmetry is why the allowlist starts *minimal*. And any list endpoint paginates from day one (section 5) — retrofitting pagination onto a shipped unbounded endpoint is itself a breaking change.

## Self-check before you call it done

Run this against your own diff. If any answer is "no," fix it before finishing:

- Did you surface assumptions and ambiguity up front rather than silently guessing, and is this the minimum that solves the problem (nothing speculative)?
- Does every changed line trace directly to the request — no unrequested refactors, reformatting, or "improvements" to adjacent code, and no fossils of abandoned attempts layered under the final fix?
- Zero new bare suppressions — no `as any` / `@ts-ignore` / `eslint-disable` / `# type: ignore` / `--force` without a narrowest-scope placement and a comment justifying the false positive?
- No debugging debris left behind — `console.log` / `print` / `debugger` statements, temp scripts, or commented-out experiments from this session?
- Is every claim in your report something you actually observed (command run, output seen this session) — nothing asserted on faith?
- Could a new teammate guess what every name means without reading its definition? No generic `data`/`temp`/`Manager` survivors?
- Do schema and API names follow the conventions — `snake_case` fields, UUID ids, `created_date`/`last_modified_date` (or `*_at`) timestamps in UTC, and first/last name stored as separate fields?
- Does every query select only the columns it uses and bound its result set (pagination), instead of `SELECT *` or loading everything into memory? Is filtering/counting done in the database, with no N+1 loops?
- Is every external input validated against a schema at the boundary (allow-list, server-side) before use, with client-side checks treated as UX only?
- Are all SQL queries parameterized — never built by concatenating input — and is shell/`eval`/file-path/HTML use sanitized or encoded with a safe primitive?
- Before writing any helper or algorithm, did you search the codebase, the standard library, and the project's dependencies for an existing implementation — no re-implemented utilities in the diff?
- Does each function do one thing, at one level of abstraction — and is every function that crossed the ~30–40-line tripwire either split or explicitly justified?
- Is the happy path flat (guard clauses), not buried in nested `if`s?
- Are presentation, logic, and data access in separate places? Zero inline styles, zero magic numbers/strings?
- Do comments explain *why*, with no commented-out code and no comments that just restate the code?
- Are errors handled where there's context, never silently swallowed?
- Does all logging go through the shared logger (not raw `console.log`/`print`), with a level, a scannable message, structured context, a correlation id, and zero secrets or PII dumps?
- Do API failures return a generic message (real detail logged internally), and do auth/lookup responses avoid revealing whether an account or record exists ("email or password is incorrect")?
- Is every API response built from an explicit field allowlist (never a serialized entity), with honest status codes, one consistent error shape, and pagination on every list endpoint?
- If you edited existing code, does your change match the file's existing conventions?

These aren't bureaucracy — each one is a thing that bites the next person to open the file. Leave the code so the next reader thanks you.
