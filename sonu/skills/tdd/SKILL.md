---
name: tdd
description: Test-driven development — the red-green-refactor discipline for writing code that's correct by design, not by accident. INVOKE THIS PROACTIVELY — even when the user never says "TDD" or "tests" — whenever writing or changing code, implementing a feature, fixing a bug, adding or running tests, naming or structuring a test suite, or choosing what to mock. The test comes first; the code earns its keep by making the test pass. Covers the red-green-refactor loop, test-first discipline with honest carve-outs, testing behavior not implementation, AAA structure, spec-sentence naming, test qualities (fast/isolated/deterministic), test doubles and when not to mock, the testing pyramid, coverage as a byproduct not a target, and the bug-fix reflex. (Pairs with the [[code-standards]] skill — tests are code held to the same bar.)
---

# TDD — test-first, every time

Tests aren't something you write when the feature is done. They're the tool you use to *design* the feature. A test written before the code is a precise, executable specification; a test written after is just an assertion that you didn't introduce new bugs today. Twenty years of shipping software keeps reinforcing the same lesson: code written test-first is more focused, more decoupled, and easier to maintain than code written any other way. The test forces you to think about the interface before the implementation — that pressure is the real payoff.

Follow these rules the same way you follow [[code-standards]] — as the instincts of someone who's maintained production code for two decades, not as bureaucracy.

## How to apply this

Before writing a single line of production code, write the failing test. Let the test drive the design. Then write the minimum code to make it pass. Then refactor. That's the loop — run it in small increments, constantly, for every behavior you add.

When you finish a change, run the self-check at the bottom against your own diff.

---

## 1. The red-green-refactor loop

Every increment of behavior follows three steps, in order:

1. **Red** — write a failing test for the *next* small, specific behavior. Run it and confirm it actually fails. Don't skip this step: a test that passes before you've written any implementation is a test that proves nothing.
2. **Green** — write the *minimum* code to make the test pass. Not the best code. The simplest thing that works. Resist the urge to generalize.
3. **Refactor** — with the test green, clean up the implementation: better names, remove duplication, clarify intent. The test catches any regression you introduce. Then go back to step 1.

Keep steps small. A step that feels too big is too big — shrink it.

**Example — incrementally building a `withdraw` method:**

```js
// Step 1 (Red): write the failing test first.
// The class doesn't exist yet — that's expected. Confirm the test fails.
test('reduces balance by withdrawal amount', () => {
  const account = new Account({ balance: 100 });
  account.withdraw(50);
  expect(account.balance).toBe(50);
});

// Step 2 (Green): the simplest code that makes the test pass.
class Account {
  constructor({ balance }) { this.balance = balance; }
  withdraw(amount) { this.balance -= amount; }
}

// Test passes. Now Step 3 (Refactor): add the guard clause cleanly, still green.
class Account {
  constructor({ balance }) { this.balance = balance; }
  withdraw(amount) {
    if (amount > this.balance) throw new Error('Insufficient funds');
    this.balance -= amount;
  }
}
// Run the test — still green. Then write the next failing test for the error case.
```

The loop enforces discipline: the test defines the expectation; the code satisfies it; neither grows beyond what's needed.

## 2. Test-first is the default — honest carve-outs

Writing the test before the code is the rule, not a suggestion. The carve-outs are real but narrow.

**Spikes are the one exception.** When you're exploring an unfamiliar API, library, or approach and don't yet know the right design, it's fine to write throwaway spike code first — to learn the shape of the problem. But a spike is **disposable by definition**: once you understand the territory, throw it away entirely and build the real thing test-first. Never let the spike become the production code with tests retrofitted onto it.

**Everything else ships test-first.** Bug fixes, new features, refactors that change behavior — all of it. "I'll add tests later" is a promissory note that almost never gets paid, and code without tests is a liability from day one.

## 3. Test behavior, not implementation

A test that reaches into private internals couples itself to *how* the code works, not *what* it does. Every refactor breaks it. Write tests that assert observable outcomes — what the caller can see — so you can freely improve the implementation underneath without touching the tests.

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

If a refactor makes a test break without changing the observable behavior, the test was wrong — not the refactor.

## 4. Arrange-Act-Assert

Every test has three phases, in order: set up the starting state, perform the one action under test, assert the outcome. One behavior per test; one reason to fail. If a test needs more than one Act-Assert pair to make a point, split it into two tests.

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

## 5. Name tests as spec sentences

A test name is the behavior it documents. When it fails, the name alone should tell you what broke and under what condition — no reading the body required. Apply the same naming instinct as [[code-standards]]: describe the domain, not the mechanism.

```js
// Avoid: names that say nothing about what's being tested
test('withdraw')
test('it works')
test_1()
it('should work correctly')

// Prefer: behavior + condition in plain language
test('reduces balance by the withdrawal amount')
test('rejects withdrawal when balance is insufficient')
test('throws when withdrawal amount is negative')
test('does not charge a fee on the first withdrawal of the month')
```

A good test suite, reading its names only, should function as a specification of what the system does. If you scan the names and can't tell what's covered, the names need fixing.

## 6. Test qualities — fast, isolated, deterministic

These properties are non-negotiable. A test suite that doesn't have them isn't a safety net — it's noise you learn to ignore.

**Fast.** Tests run on every save. If they're slow, developers stop running them and the feedback loop breaks. Unit tests should finish in milliseconds. Anything I/O-bound (database, network, file system) belongs at the integration layer — keep it out of the unit test core.

**Isolated.** Each test is independent: no shared mutable state between tests, no ordering dependency. Tests that fail differently depending on run order are a smell — some other test's state is leaking in. Start from a known state; clean up after yourself.

**Deterministic.** A test that sometimes passes and sometimes fails is worse than useless — it trains you to ignore red. The usual culprits are the real clock, random numbers, and the file system. Inject dependencies so tests control them.

```js
// Avoid: depends on the real clock — breaks at a future date, non-deterministic
test('marks payment as overdue after 30 days', () => {
  const payment = new Payment({ dueDate: new Date('2026-01-01') });
  // This passes only when the wall-clock date is after 2026-01-31
  expect(payment.isOverdue()).toBe(true);
});

// Prefer: inject the clock so the test controls time
test('marks payment as overdue after 30 days', () => {
  const frozenClock = { now: () => new Date('2026-02-15') };
  const payment = new Payment({ dueDate: new Date('2026-01-01'), clock: frozenClock });
  expect(payment.isOverdue()).toBe(true);
});
```

**Self-validating.** The test itself asserts pass or fail — no human reads output to decide. If there's no assertion, there's no test.

## 7. Test doubles — mock only at the seams

A test double is a stand-in for a real collaborator. Stubs return canned data. Mocks assert call behavior. Fakes are working lightweight implementations. Spies record what was called. They're useful. They're also overused.

**Mock only at architectural seams** — things that cross a process boundary: the real database, a payment gateway, an email service, the network, the file system, the clock. These are the right places to substitute a double because they're slow, unreliable, or have real-world consequences you don't want tests to trigger.

**Don't mock your own domain objects.** If you're mocking a value object, an entity, or the unit under test itself, you've built a test that tests nothing — it's a mock returning another mock. Use the real thing.

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

The rule of thumb: the more you mock, the less you're testing. When a test passes in the presence of a bug, you mocked too much.

## 8. The testing pyramid

Tests live at three layers, and the distribution matters:

- **Unit tests (base, most)** — one unit in isolation, real domain objects, faked I/O. Fast, focused, cheap. The bulk of the suite.
- **Integration tests (middle, fewer)** — components wired together: real database, real HTTP, real queue. Slower but necessary to catch boundary mismatches.
- **End-to-end tests (top, fewest)** — drive the whole system through its actual interface (browser, CLI, API). Slow, expensive, brittle at the edges. Protect the critical paths, not every branch.

Invert the pyramid — many end-to-end tests with a thin unit base — and you get a slow, brittle suite that makes shipping painful. Push as much as possible down to the unit level.

## 9. Coverage is a byproduct, not a target

Code coverage tells you which lines were executed. It doesn't tell you whether the behavior was verified. A test that calls a function without asserting anything meaningful moves the number but catches no bugs — it's negative value: maintenance cost with no protection.

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

Don't chase 100%. Trivial getters, generated boilerplate, and pure configuration don't need unit tests. Use coverage to find gaps in behavior, not to hit a number. Gaming a coverage threshold is strictly worse than not having one.

## 10. What to test — and the bug-fix reflex

Test the things that can go wrong:

- **Behavior** — what does the unit do when things go right?
- **Boundaries** — what happens at the edges: empty inputs, maximum values, zero, null?
- **Error paths** — what does it do when a collaborator fails or input is invalid?
- **Business rules** — the non-obvious domain logic that lives nowhere else.

Skip trivial pass-throughs, generated code, and pure configuration wiring.

**The bug-fix reflex.** Before fixing any bug, write a test that reproduces it. Confirm the test fails. Then fix the bug. Confirm the test passes. This is non-negotiable: it proves the fix works, prevents the regression from returning, and often reveals the bug was more general than it first appeared.

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

## 11. Tests are first-class code

A test suite that's hard to read is a test suite nobody trusts. Apply [[code-standards]] to test code with the same discipline as production code: intention-revealing names, small focused helpers, guard clauses, no magic numbers, no commented-out tests, no duplication. Test factories and shared fixtures deserve the same care as the code they exercise. If the test body is long enough to need scrolling, extract helpers.

---

## Self-check before you call it done

Run this against your own diff. Fix any "no" before finishing:

- Did you write the failing test *before* the production code, or if this was a spike, have you discarded it and rebuilt test-first?
- Does every new or changed behavior have a test that would fail if you deleted the implementation?
- Are you asserting observable outcomes — not private state, not internal call sequences?
- Does each test have a clear Arrange-Act-Assert structure, one behavior, and one reason to fail?
- Does every test name read as a spec sentence — what broke, under what condition — without reading the body?
- Are tests fast (milliseconds for unit), isolated (no shared mutable state, no order dependency), and deterministic (no real clock, network, or random values)?
- Are test doubles used only at architectural seams — real domain objects throughout the core, fakes only at I/O boundaries?
- If this was a bug fix, did you write a failing test that reproduced the bug *before* you fixed it?
- Does each new test actually fail before the implementation and pass after? (A test that's always green proved nothing.)
- Is the test code held to the same naming, clarity, and structure bar as [[code-standards]]?

A passing test suite is only as trustworthy as the discipline behind it. If you're not confident the tests would catch a regression, they wouldn't.
