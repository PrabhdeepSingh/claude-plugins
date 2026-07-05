# Security examples — error handling, validation, and leak prevention

Depth and worked examples for `code-standards` §7, §9, and §10. Read this file when the change handles external input, authentication, or an error/fallback path. The rules themselves (with their *why*) live inline in `SKILL.md` — this file is the illustration.

## §7 — the silent fallback, worked

A silent fallback (`catch { return [] }`, `?? defaultValue` on a failure path) looks like robustness but converts a crash (loud, findable, fixed today) into wrong data (silent, trusted, discovered months later):

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

A fallback is legitimate only when the degraded behavior is a *product decision* ("show cached prices if the pricing service is down") — and even then it logs the failure and is visibly a fallback.

## §9 — parameterize, validate at the boundary

**SQL injection — the value is data, never executable SQL:**

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

**Boundary validation — reject what doesn't conform before it touches any logic:**

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

The same instinct extends past SQL: never hand unsanitized input to a shell (pass an argument array, not a shell string), to `eval`, to a file path (path traversal), or into an HTML page (encode on output — XSS). When validation rejects something, return a generic "invalid input" to the caller and log the specifics internally (§10) — a detailed validation error is itself a hint an attacker can probe with.

## §10 — the login-enumeration case, worked

The textbook leak: confirming *which half* of a credential pair was wrong lets an attacker enumerate valid emails, then brute-force just the password.

```js
// Avoid: confirms the email is valid, so an attacker now only has to brute-force the password
if (!user)            return res.status(404).json({ error: 'No account found for that email' });
if (!passwordValid)   return res.status(401).json({ error: 'Incorrect password' });

// Prefer: one generic message for the pair; the real reason is logged, not returned
if (!user || !passwordValid) {
  // Full email is OK *here* because authLog is access-controlled with short
  // retention — the audit-event exception in §8, not the general app log.
  authLog.warn('Failed login attempt', {
    request_id: req.id,
    reason: user ? 'bad_password' : 'no_account',
    email_attempted: email,
  });
  return res.status(401).json({ error: 'Email or password is incorrect' });
}
```

Apply the same thinking to password reset and signup: respond identically whether or not the account exists ("If that email is registered, we've sent a reset link"). Anywhere there's an existence or authorization check, keep `not found` and `forbidden` indistinguishable (shape, and ideally timing), so the caller can't probe for what exists or what they're not allowed to see.
