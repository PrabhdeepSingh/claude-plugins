# TDD worked examples

Full code examples for each rule in `tdd/SKILL.md`. Read the section matching whatever you're currently doing — writing a new test, checking a naming/structure question, or deciding whether a test double is appropriate. The rules and their *why* live inline in `SKILL.md`; this file is the illustration.

## §1 — the red-green-refactor loop, step by step

Each block below is the full state of the file *after* that step — not a single file to concatenate and run:

```js
// Step 1 (Red): write the failing test first.
// The class doesn't exist yet — that's expected. Confirm the test fails.
test('reduces balance by withdrawal amount', () => {
  const account = new Account({ balance: 100 });
  account.withdraw(50);
  expect(account.balance).toBe(50);
});
```

```js
// Step 2 (Green): the simplest code that makes the test pass.
class Account {
  constructor(opts) { this.balance = opts.balance; }
  withdraw(amount) { this.balance -= amount; }
}
```

```js
// Step 3 (Refactor): clean up under green — clearer names, no NEW behavior.
class Account {
  constructor({ balance }) { this.balance = balance; }
  withdraw(amount) { this.balance -= amount; }
}
// Run the test — still green. The loop restarts for the next behavior:
```

```js
// Step 1 again (Red): failing test for the error case.
test('rejects withdrawal when balance is insufficient', () => {
  const account = new Account({ balance: 30 });
  expect(() => account.withdraw(50)).toThrow('Insufficient funds');
});
```

```js
// Step 2 again (Green): the guard clause, driven by the failing test above.
class Account {
  constructor({ balance }) { this.balance = balance; }
  withdraw(amount) {
    if (amount > this.balance) throw new Error('Insufficient funds');
    this.balance -= amount;
  }
}
// Both tests green. Refactor if needed, then on to the next behavior.
```

## §3 — behavior, not implementation

```js
// Avoid: asserting private state and internal call sequences
test('withdraw', () => {
  const account = new Account({ balance: 100 });
  account.withdraw(50);
  expect(account._ledger.entries[0].type).toBe('debit'); // private internals
  expect(account._notifier.send).toHaveBeenCalledTimes(1); // internal call count
});

// Prefer: assert what the caller observes
test('reduces balance by withdrawal amount', () => {
  const account = new Account({ balance: 100 });
  account.withdraw(50);
  expect(account.balance).toBe(50);
});

test('rejects withdrawal when balance is insufficient', () => {
  const account = new Account({ balance: 30 });
  expect(() => account.withdraw(50)).toThrow('Insufficient funds');
});
```

## §4 — Arrange-Act-Assert

```js
// Avoid: setup, action, and assertions interleaved — hard to see what's being tested
test('transfer', () => {
  const src = new Account({ balance: 200 });
  const dst = new Account({ balance: 0 });
  src.deposit(50);           // extra setup buried mid-test
  src.transferTo(dst, 100);
  expect(dst.balance).toBe(100);
  src.withdraw(25);          // a second unrelated action
  expect(src.balance).toBe(125);
});

// Prefer: three clean phases, one logical behavior per test
test('transfers the amount from source to target balance', () => {
  // Arrange
  const source = new Account({ balance: 200 });
  const target = new Account({ balance: 0 });

  // Act
  source.transferTo(target, 100);

  // Assert
  expect(source.balance).toBe(100);
  expect(target.balance).toBe(100);
});
```

## §6 — determinism: inject the clock

```js
// Avoid: depends on the real clock — non-deterministic across time
test('marks payment as overdue after 30 days', () => {
  const payment = new Payment({ dueDate: new Date('2099-01-01') });
  // Fails today; passes after 2099-01-31 — the result changes with the calendar
  expect(payment.isOverdue()).toBe(true);
});

// Prefer: inject the clock so the test controls time
test('marks payment as overdue after 30 days', () => {
  const frozenClock = { now: () => new Date('2099-02-15') };
  const payment = new Payment({ dueDate: new Date('2099-01-01'), clock: frozenClock });
  expect(payment.isOverdue()).toBe(true);
});
```

## §7 — mock only at seams

```js
// Avoid: mocking everything, including domain objects you own
test('processes payment', async () => {
  const mockOrder = { id: '123', total: 99.00, markPaid: jest.fn() };
  const mockGateway = { charge: jest.fn().mockResolvedValue({ success: true }) };
  const mockEmailer = { send: jest.fn() };
  // Lots of mock wiring, and you're not testing the Order — you're testing the mock
  await paymentService.process(mockOrder, mockGateway, mockEmailer);
  expect(mockOrder.markPaid).toHaveBeenCalled();
});

// Prefer: real domain objects, fake only the I/O boundary
test('marks order paid after successful charge', async () => {
  const order = new Order({ id: '123', total: 99.00 });  // real domain object
  const fakeGateway = { charge: async () => ({ success: true }) }; // fake the I/O seam

  await paymentService.process(order, fakeGateway);

  expect(order.isPaid).toBe(true); // observable state on the real object
});
```

## §9 — coverage theater vs. a real assertion

```js
// Avoid: executes code with no meaningful assertion — coverage theater
test('withdraw runs', () => {
  const account = new Account({ balance: 100 });
  account.withdraw(50); // code runs, coverage goes up, nothing is verified
});

// Prefer: the assertion pins a real behavior
test('reduces balance by the withdrawal amount', () => {
  const account = new Account({ balance: 100 });
  account.withdraw(50);
  expect(account.balance).toBe(50);
});
```

## §10 — the bug-fix reflex, worked

```js
// Avoid: patch the bug and ship — no proof it's fixed, no protection against regression
// Found: withdraw allows negative amounts. Fix applied directly to withdraw().

// Prefer: reproduce with a failing test first, then fix
test('rejects negative withdrawal amount', () => {
  const account = new Account({ balance: 100 });
  // Run this before the fix — confirm it fails (the bug is real)
  expect(() => account.withdraw(-50)).toThrow('Amount must be positive');
});
// Watch it fail → add the guard in withdraw() → watch it pass → done
```

## §10 — thresholds that actually trip

```js
// Avoid: production-scale limit — the enforcement branch never executes in any test
test('rate limiter allows requests', () => {
  const limiter = new RateLimiter({ maxRequestsPerMinute: 10000 });
  expect(limiter.allow('client-a')).toBe(true); // never gets near 10000 — the limiting code is untested
});

// Prefer: a limit the test can reach — assert both sides of the boundary
test('allows requests under the limit', () => {
  const limiter = new RateLimiter({ maxRequestsPerMinute: 2 });
  expect(limiter.allow('client-a')).toBe(true);
  expect(limiter.allow('client-a')).toBe(true);
});

test('blocks the request that exceeds the limit', () => {
  const limiter = new RateLimiter({ maxRequestsPerMinute: 2 });
  limiter.allow('client-a');
  limiter.allow('client-a');
  expect(limiter.allow('client-a')).toBe(false); // the enforcement branch actually runs
});
```

## §11 — when a test fails, the test is innocent

```js
// Bug: withdraw() lets the balance go negative. The test correctly fails.

// Avoid: "fixing" the test to match what the code does
test('withdrawal can overdraw the account', () => {
  const account = new Account({ balance: 30 });
  account.withdraw(50);
  expect(account.balance).toBe(-20);   // the bug is now the spec, and it's green
});

// Prefer: the test stays; the code changes until it passes
test('rejects withdrawal when balance is insufficient', () => {
  const account = new Account({ balance: 30 });
  expect(() => account.withdraw(50)).toThrow('Insufficient funds');
});
```
