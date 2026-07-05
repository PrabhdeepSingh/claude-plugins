# Data & API examples — schema, queries, and response contracts

Depth and worked examples for `code-standards` §2, §5, and §13. Read this file when the change touches a database schema, a query, or an API endpoint's request/response shape. The rules themselves (with their *why*) live inline in `SKILL.md` — this file is the illustration, not a second copy of the rule.

## §2 — a well-named record

Field-naming rules in one example: `snake_case`, UUID ids, separate first/last name, boolean-as-question, integer minor units for money, UTC timestamps.

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

## §5 — data access

**SQL — select only what's used, filtered and bounded:**

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

**ORM — same discipline, same shape:**

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

The N+1 case is the same principle in a different shape: fetching a list and then firing one more query per row turns one round trip into a thousand. Use a join or a batched `IN (...)` fetch instead of looping queries per row.

## §13 — API response contract, worked

**The allowlist, not the entity:**

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

**Status codes, spelled out:** `201` created, `400` invalid input, `401` unauthenticated, `403` forbidden, `404` not found, `409` conflict, `422` semantically invalid, `5xx` our fault. Never `200` with `{ "error": ... }` in the body — a lying 200 breaks retries, caching, monitoring, and every client that trusted it. Remember `code-standards` §10: keep `403` vs `404` indistinguishable wherever existence itself is sensitive.

**One error envelope, everywhere:** every error response uses the same shape — a stable machine-readable code, a human message, a correlation id. §10 governs what goes *in* it (generic message out, detail logged internally). Ten endpoints with ten error formats means every client writes ten parsers.

**Breaking a response is a migration:** removing/renaming a field, changing a type, tightening semantics — clients break exactly like the database clients in `[[safe-migrations]]`, and the fix is the same staged path: add the new field alongside the old, migrate consumers, retire the old one deliberately (deprecation header/date, then removal) — never in one step. Additive changes are free, which is why the allowlist starts *minimal*. Retrofitting pagination onto a shipped unbounded list endpoint is itself a breaking change for the same reason.
