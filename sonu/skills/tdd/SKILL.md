---
name: tdd
description: Test-driven development — the red-green-refactor discipline for writing code that's correct by design, not by accident. INVOKE PROACTIVELY — even when the user never says "TDD" or "tests" — whenever writing or changing code, implementing a feature, fixing a bug, adding/running/naming tests, structuring a test suite, or choosing what to mock. (Pairs with [[code-standards]] — tests are code held to the same bar.)
---

# TDD — test-first, every time

Tests aren't something you write when the feature is done. They're the tool you use to *design* the feature. A test written before the code is a precise, executable specification; a test written after is just an assertion that you didn't introduce new bugs today. Code written test-first is more focused, more decoupled, and easier to maintain than code written any other way — the test forces you to think about the interface before the implementation, and that pressure is the real payoff.

Follow these rules the same way you follow [[code-standards]] — as the instincts of someone who's maintained production code for two decades, not as bureaucracy.

## How to apply this

Before writing a single line of production code, write the failing test. Let the test drive the design. Then write the minimum code to make it pass. Then refactor. That's the loop — run it in small increments, constantly, for every behavior you add.

When you finish a change, run the self-check at the bottom against your own diff.

---

## 1. The red-green-refactor loop

Every increment of behavior follows three steps, in order:

1. **Red** — write a failing test for the *next* small, specific behavior. Run it and confirm it actually fails. A test that passes before you've written any implementation proves nothing.
2. **Green** — write the *minimum* code to make the test pass. Not the best code — the simplest thing that works. Resist the urge to generalize.
3. **Refactor** — with the test green, clean up the implementation: better names, remove duplication, clarify intent. The test catches any regression. Then go back to step 1.

Keep steps small. A step that feels too big is too big — shrink it.

→ `references/examples.md` §1 — a full step-by-step example building an `Account.withdraw` method, read when you want to see the loop applied end-to-end.

## 2. Test-first is the default — honest carve-outs

Writing the test before the code is the rule, not a suggestion. **Spikes are the one exception**: when exploring an unfamiliar API or approach, throwaway spike code is fine to learn the shape of the problem — but a spike is **disposable by definition**. Once you understand the territory, throw it away entirely and build the real thing test-first. Never let the spike become the production code with tests retrofitted onto it.

Everything else ships test-first — bug fixes, new features, refactors that change behavior. "I'll add tests later" is a promissory note that almost never gets paid.

## 3. Test behavior, not implementation

A test that reaches into private internals couples itself to *how* the code works, not *what* it does — every refactor breaks it. Write tests that assert observable outcomes, so you can freely improve the implementation underneath without touching the tests. If a refactor makes a test break without changing observable behavior, the test was wrong — not the refactor.

→ `references/examples.md` §3 — private-internals vs. observable-outcome example.

## 4. Arrange-Act-Assert

Every test has three phases, in order: set up the starting state, perform the one action under test, assert the outcome. One behavior per test; one reason to fail. If a test needs more than one Act-Assert pair to make a point, split it into two tests.

→ `references/examples.md` §4 — interleaved-vs-phased example.

## 5. Name tests as spec sentences

A test name is the behavior it documents. When it fails, the name alone should tell you what broke and under what condition — no reading the body required. Apply the same naming instinct as [[code-standards]]: describe the domain, not the mechanism.

```js
// Avoid: names that say nothing about what's being tested
test('withdraw')
test('it works')

// Prefer: behavior + condition in plain language
test('reduces balance by the withdrawal amount')
test('rejects withdrawal when balance is insufficient')
```

A good test suite, reading its names only, should function as a specification of what the system does.

## 6. Test qualities — fast, isolated, deterministic

Non-negotiable — a test suite without these isn't a safety net, it's noise you learn to ignore.

- **Fast.** Unit tests finish in milliseconds; anything I/O-bound belongs at the integration layer.
- **Isolated.** No shared mutable state between tests, no ordering dependency.
- **Deterministic.** No real clock, random numbers, or file system in the unit core — inject dependencies so tests control them. A test that sometimes passes and sometimes fails trains you to ignore red.
- **Self-validating.** The test itself asserts pass or fail — no human reads output to decide.

→ `references/examples.md` §6 — injecting a frozen clock instead of depending on real time.

## 7. Test doubles — mock only at the seams

A test double is a stand-in for a real collaborator: **stubs** return canned data, **mocks** assert call behavior, **fakes** are working lightweight implementations, **spies** record what was called. **Mock only at architectural seams** — things that cross a process boundary: a real database, a payment gateway, the network, the clock. These are slow, unreliable, or have real-world consequences you don't want tests to trigger. **Don't mock your own domain objects** — a test that mocks the unit under test or its value objects tests nothing. The rule of thumb: the more you mock, the less you're testing.

→ `references/examples.md` §7 — over-mocked vs. seam-only example.

## 8. The testing pyramid

Tests live at three layers: **unit** (base, most — one unit in isolation, real domain objects, faked I/O), **integration** (middle, fewer — real database/HTTP/queue, catches boundary mismatches), **end-to-end** (top, fewest — the whole system through its actual interface, slow and brittle at the edges). Invert the pyramid and you get a slow, brittle suite that makes shipping painful. Push as much as possible down to the unit level.

## 9. Coverage is a byproduct, not a target

Coverage tells you which lines ran, not whether the behavior was verified. A test that calls a function without asserting anything meaningful moves the number but catches no bugs — negative value: maintenance cost with no protection. Don't chase 100%; use coverage to find gaps in behavior, not to hit a number.

→ `references/examples.md` §9 — coverage-theater vs. a real assertion.

## 10. What to test — and the bug-fix reflex

Test **behavior** (what happens when things go right), **boundaries** (empty, max, zero, null), **error paths** (collaborator failure, invalid input), and **business rules** (non-obvious domain logic). Skip trivial pass-throughs and generated code.

**The bug-fix reflex.** Before fixing any bug, write a test that reproduces it. Confirm it fails. Then fix the bug. Confirm it passes. This is non-negotiable — it proves the fix works, prevents the regression's return, and often reveals the bug was more general than it first appeared. (Finding the root cause is [[debugging]]'s territory; this reflex is how the found fix gets pinned.)

**Thresholds must trip in the test.** When code enforces a limit — a rate limit, quota, timeout, retry cap, buffer size, pagination bound — configure the test with a value small enough to actually reach (a limit of 2–3, a timeout of milliseconds) and assert **both sides**: under the limit passes, at/over the limit is enforced. Why: testing with production-scale thresholds means the enforcement branch never executes — the feature reads as covered while the code path that actually limits has never run once. A limit that has never tripped in a test is untested, whatever the coverage report says. This forces the threshold to be injectable rather than hardcoded — which is exactly the design pressure the test is supposed to apply.

→ `references/examples.md` §10 — reproduce-before-fix and trip-the-threshold examples.

## 11. When a test fails, the test is innocent

A failing test is the system working. The default assumption — always — is that **the code is wrong, not the test**. Weakening a test to make it pass converts a loud, findable failure into a silent bug with a green checkmark, which is strictly worse than no test at all.

These moves are banned as a *first response* to a red test:

- Updating the expected value to whatever the code currently produces.
- Deleting, skipping, or quarantining the test (`.skip`, `xfail`, commenting it out).
- Broadening the assertion until it can't fail (`toBe(42)` → `toBeDefined()`).
- Adding sleeps or retries to outlast a timing failure instead of finding the race.
- Wrapping the failing call in try/catch so the assertion is never reached.
- Mocking away the collaborator that's failing so the broken path is no longer exercised.

A test change is legitimate in exactly two cases, and both require saying so out loud: (1) **the specification actually changed** — point to where that decision came from; or (2) **the test violated section 3** — it asserted implementation details, and a refactor broke it without changing observable behavior. If you can't articulate which case applies, the code is wrong — go fix it.

→ `references/examples.md` §11 — the "fixing the test to match the bug" anti-pattern.

## 12. Tests are first-class code

A test suite that's hard to read is a test suite nobody trusts. Apply [[code-standards]] to test code with the same discipline as production code: intention-revealing names, small focused helpers, guard clauses, no magic numbers, no commented-out tests, no duplication. If the test body is long enough to need scrolling, extract helpers.

---

## Self-check before you call it done

Run this against your own diff. Fix any "no" before finishing:

- Did you write the failing test *before* the production code, or discard and rebuild a spike test-first?
- Does every new or changed behavior have a test that would fail if you deleted the implementation?
- Are you asserting observable outcomes — not private state, not internal call sequences?
- Does each test have a clear Arrange-Act-Assert structure, one behavior, one reason to fail?
- Does every test name read as a spec sentence — what broke, under what condition — without reading the body?
- Are tests fast, isolated (no shared mutable state or order dependency), and deterministic (no real clock/network/random)?
- Are test doubles used only at architectural seams — real domain objects throughout the core?
- If this was a bug fix, did you write a failing test that reproduced the bug *before* fixing it?
- If the change enforces a threshold (limit, quota, timeout, cap): does a test configure a value small enough to trip, and assert both sides — under the limit passes, at/over is enforced?
- Does each new test actually fail before the implementation and pass after?
- If any existing test changed: can you name which legitimate case applies (spec change or implementation-detail cleanup), with zero weakening (no updated-to-actual expectations, no skips, no broadened assertions, no sleeps, no try/catch around the failure)?
- Is the test code held to the same naming, clarity, and structure bar as [[code-standards]]?

A passing test suite is only as trustworthy as the discipline behind it. If you're not confident the tests would catch a regression, they wouldn't.

## Reference files

| File | What it answers |
|------|-----------------|
| `references/examples.md` | Full worked code for every rule above — the red-green-refactor build, behavior-vs-implementation, AAA, clock injection, mock-only-at-seams, coverage theater, bug-fix reflex, and the test-is-innocent anti-pattern |
