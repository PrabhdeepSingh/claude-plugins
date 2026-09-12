---
name: tdd
description: >-
  Test-driven development — the red-green-refactor discipline for code that's correct by design, not by accident. INVOKE PROACTIVELY whenever writing or changing code, fixing a bug, adding or structuring tests, or choosing what to mock — even when nobody says "TDD" or "tests". (Tests are code held to [[code-standards]]'s bar.)
argument-hint: "[feature, bug, or behavior to drive with tests]"
allowed-tools: Skill, Read, Write, Edit, Bash
---

# TDD — test-first, every time

Tests aren't something you write when the feature is done. They're the tool you use to *design* the feature. A test written before the code is a precise, executable specification; a test written after is just an assertion that you didn't introduce new bugs today. Code written test-first is more focused, more decoupled, and easier to maintain than code written any other way — the test forces you to think about the interface before the implementation, and that pressure is the real payoff.

Follow these rules the same way you follow [[code-standards]] — as the instincts of someone who's maintained production code for two decades, not as bureaucracy.

## How to apply this

Before writing a single line of production code, write the failing test. Let the test drive the design. Then write the minimum code to make it pass. Then refactor. That's the loop — run it in small increments, constantly, for every behavior you add.

**Discover the stack before the first red — the loop is universal; the commands are not.** Find the build system from its manifest (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `Gemfile`, a `Makefile`); prefer checked-in wrappers (`./gradlew`, `./mvnw`, `make test`, a repo script) over globally installed tools; learn **both** the focused-single-test command (run during the loop) and the full-suite command (run before completion) — they differ in every framework; and read the README/CONTRIBUTING and CI workflow files, because those show the commands that actually gate merges. Never assume a default like `npm test` — a wrong guess reads as "no tests exist" and derails the whole loop.

**Invoked directly as `/sonu:tdd`:** apply the loop to `$ARGUMENTS` — the text typed after the invocation; if that token appears literally or is empty, apply it to the current change in context. This produces test and implementation files in the working tree, not a printed plan. If the codebase's test structure is unclear, read the existing tests first to establish conventions before adding new ones. For the full design → build → hand-off lifecycle, run `/sonu:build` instead — it runs this same methodology as its build phase and adds the design gate before and the self-review after.

One execution habit: after a green run, don't re-run the same command on unchanged code — that's reassurance-seeking, not verification; run again after the next edit. The one exception is hunting a suspected flake, where repetition *is* the experiment (that's [[debugging]]'s loop, and the fix is removing the nondeterminism §6 bans).

When you finish a change, run the self-check at the bottom against your own diff.

---

## 1. The red-green-refactor loop

Every increment of behavior follows three steps, in order:

1. **Red** — write a failing test for the *next* small, specific behavior. Run it and confirm it actually fails. A test that passes before you've written any implementation proves nothing.
2. **Green** — write the *minimum* code to make the test pass. Not the best code — the simplest thing that works. Resist the urge to generalize.
3. **Refactor** — with the test green, clean up the implementation: better names, remove duplication, clarify intent. The test catches any regression. Then go back to step 1.

Keep steps small. A step that feels too big is too big — shrink it.

**When the same step fails twice, shrink the step rather than trying harder.** A green that will not arrive after two honest attempts at one increment is telling you the increment is too big — the test asserts several behaviors at once, or the implementation it demands spans two decisions. Revert to the last green, split the behavior in half, take the smaller half first. If the split also stalls, stop and hand it over: a plain statement of the last green state, the two attempts, and what each ruled out is worth more than a third attempt at the same size — it is the escalation summary [[debugging]] §9 asks for. The tell that you are over the line is reaching for a bigger move than the step needs: an extracted collaborator when the test asked for a constant.

→ `references/examples.md` §1 — a full step-by-step example building an `Account.withdraw` method, read when you want to see the loop applied end-to-end.

## 2. Test-first is the default — honest carve-outs

Writing the test before the code is the rule, not a suggestion. **Spikes are the one exception**: when exploring an unfamiliar API or approach, throwaway spike code is fine to learn the shape of the problem — but a spike is **disposable by definition**. Once you understand the territory, throw it away entirely and build the real thing test-first. Never let the spike become the production code with tests retrofitted onto it.

Everything else ships test-first — bug fixes, new features, refactors that change behavior. "I'll add tests later" is a promissory note that almost never gets paid.

**Changing code that already exists and has no test: pin it before you touch it.** Test-first assumes the behavior is yours to specify; when the code already runs in production, the behavior is already the de facto spec and your job is to record it, not invent it. Write a *characterization* test: call the code in a harness, assert something deliberately absurd (`expect(total).toBe(-1)`), run it, and read the failure — it tells you what the code actually returns. Replace the absurd value with the observed one and you have a passing test that pins today's behavior, quirks included. Now make the change; the pinned test fails exactly when and where you altered existing behavior, which is the point. Two rules keep it honest. If characterizing reveals a bug, do not quietly fix it in the same breath — downstream callers may depend on the wrong answer; pin the wrong behavior, note it, and fix it as its own deliberate change. And a characterization test is the safety net you build before you climb, never a substitute for the real test of the behavior you are adding, which still arrives red-green-refactor.

## 3. Test behavior, not implementation

A test that reaches into private internals couples itself to *how* the code works, not *what* it does — every refactor breaks it. Write tests that assert observable outcomes, so you can freely improve the implementation underneath without touching the tests. If a refactor makes a test break without changing observable behavior, the test was wrong — not the refactor.

→ `references/examples.md` §3 — private-internals vs. observable-outcome example — read when a test wants to reach into internals.

## 4. Arrange-Act-Assert

Every test has three phases, in order: set up the starting state, perform the one action under test, assert the outcome. One behavior per test; one reason to fail. If a test needs more than one Act-Assert pair to make a point, split it into two tests.

**Assert the blast radius, not just the return value.** An action that changes state changes more than one thing: the call returns, a total is recomputed, a flag flips, neighbours are left alone. A test that asserts only the obvious output passes while the rest silently rots — so after writing the Act line, ask what *else* changed and assert the one or two of those a regression would break. On error paths the same rule means the error's type or message plus whatever cleanup was supposed to run; "it threw" is satisfied by throwing for the wrong reason. And never put an assertion inside a `try`/`catch` or an `if`: an assertion that can be skipped makes the test pass whether or not the behavior works, which is worse than no test because it reports as coverage.

→ `references/examples.md` §4 — interleaved-vs-phased example — read when a test interleaves its phases.

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

→ `references/examples.md` §6 — injecting a frozen clock instead of depending on real time — read when a test depends on time, randomness, or I/O.

## 6a. A flake is fixed at the cause, and proven by repetition

A test that sometimes fails has a source of nondeterminism — order dependence, a real clock, an unseeded RNG, an unawaited race, an unordered collection, a leaked resource — and the fix removes that source. Before fixing, establish the failure rate empirically rather than by report: run **that test** on repeat (shuffled, in parallel, under the race detector where one exists), record the rate as a measurement — "fails 23 of 1,000 runs" — and for anything randomized print and pin the seed so the failing scenario replays. After fixing, prove it the same way: one green run is not proof, because a test that fails once in forty passes thirty-nine times without help. Run roughly 20 randomized repeats of the test — never the whole suite — plus the suite once, then remove the `retry`/`skip`/`flaky` annotation that was masking it, because a fix that leaves the mask in place is indistinguishable from no fix. Two things are never the fix: a sleep, a raised timeout, a retry wrapper, or a skip (§11 bans them as a first response; this section bans them as a last one), and stabilising a test whose flakiness is the *product* racing — when the interleaving that breaks the test can happen in production, the test is the messenger; fix the code and say so in the hand-off.

## 7. Test doubles — mock only at the seams

A test double is a stand-in for a real collaborator: **stubs** return canned data, **mocks** assert call behavior, **fakes** are working lightweight implementations, **spies** record what was called. **Mock only at architectural seams** — things that cross a process boundary: a real database, a payment gateway, the network, the clock. These are slow, unreliable, or have real-world consequences you don't want tests to trigger. **Don't mock your own domain objects** — a test that mocks the unit under test or its value objects tests nothing. The rule of thumb: the more you mock, the less you're testing. Preference order when a double is genuinely needed: **real implementation → fake → stub → mock** — reach for the next rung only when the previous one is too slow, nondeterministic, or side-effecting.

→ `references/examples.md` §7 — over-mocked vs. seam-only example — read when choosing what to mock.

## 8. The testing pyramid

Tests live at three layers: **unit** (base, most — one unit in isolation, real domain objects, faked I/O), **integration** (middle, fewer — real database/HTTP/queue, catches boundary mismatches), **end-to-end** (top, fewest — the whole system through its actual interface, slow and brittle at the edges). Invert the pyramid and you get a slow, brittle suite that makes shipping painful. Push as much as possible down to the unit level.

## 9. Coverage is a byproduct, not a target

Coverage tells you which lines ran, not whether the behavior was verified. A test that calls a function without asserting anything meaningful moves the number but catches no bugs — negative value: maintenance cost with no protection. Don't chase 100%; use coverage to find gaps in behavior, not to hit a number.

**Snapshot tests are the same theater in a different costume.** A large snapshot nobody reviews breaks on any change and gets blindly regenerated — an assertion that asserts nothing. Use snapshots sparingly, keep them small enough to read in a diff, and review every regeneration as a deliberate spec change.

→ `references/examples.md` §9 — coverage-theater vs. a real assertion — read when tempted to chase a coverage number.

## 10. What to test — and the bug-fix reflex

Test **behavior** (what happens when things go right), **boundaries** (empty, max, zero, null), **error paths** (collaborator failure, invalid input), and **business rules** (non-obvious domain logic). Skip trivial pass-throughs and generated code.

**The bug-fix reflex.** Before fixing any bug, write a test that reproduces it. Confirm it fails. Then fix the bug. Confirm it passes. This is non-negotiable — it proves the fix works, prevents the regression's return, and often reveals the bug was more general than it first appeared. (Finding the root cause is [[debugging]]'s territory; this reflex is how the found fix gets pinned.)

**A test written after the code is proven by breaking the code, not by passing.** The red-green loop proves a test by construction — you watched it fail before anything existed to pass it. A test added *after* the implementation (the bug-fix reflex, a review-driven test) has no such proof, and a test that has never been red is indistinguishable from a comment. Four moves, in order: run it green; revert the implementation line it covers (`git stash`, invert the condition, or return the empty value); run it again and **confirm it fails, for the reason the test names** — not on a missing import or a setup error; restore and run green. If it stayed green, the assertion is not reaching the behavior — rewrite it before moving on. The proof is per behavior, not per parametrized row. One cheap reading of any test: name what the implementation would have to return for it to pass — if `null`, `""`, `[]`, or "no exception" passes it, it asserts nothing. And when an existing test changed in the same diff, read that change first, against §11: a widened tolerance, an assertion downgraded to "an error exists", or a new skip is the definition of correct moving.

**Test data has to be able to tell the answers apart.** A test can exercise the right line and still be unable to fail, because the fixture makes two different behaviors look identical. Four shapes cause almost all of it, each with a mechanical fix applied while writing: a boolean or flag exercised in only one of its two states (test the other); a collection fixture with exactly one element, which makes "the first", "the last", and "each" behave the same (use two, with different values); two collaborators or fields stubbed to the *same* value, so the test passes whichever one the code reads (make them differ); and a parameter with a default that every test passes explicitly, leaving the default path never executed (add a call that omits it). The question of a finished test is not "does it cover this line" but "if I changed one constant, flipped one branch, or read the other field, would this go red?"

**Boundaries the code does not own get a table before green.** When a function touches an external ceiling (an API's address limit), an encoding (bytes versus code units), a timezone or calendar rule, a documented limit, or a platform difference (path separators, shell quoting), write the boundary rows first as parametrized cases: the maximum, the maximum plus one, both ends of any interval (inclusive and exclusive), an impossible value, and for time a DST transition and a non-UTC zone. This is scoped on purpose — "any function handling strings or collections" is every function in a service; the table is for the boundaries some other system defines and this code merely meets.

**Visible behavior needs behavioral evidence.** A green unit suite does not prove that a screen renders, a button works, or a flow completes — it proves the units it covers behave as asserted. When a change alters visible or interactive behavior (UI, a CLI's output, an end-to-end flow), **exercise the real flow** and capture evidence the environment supports: a screenshot, a recording, the actual terminal output, or the observed response. Why: the most confident wrong claim in software is "tests pass, so it works" about a surface no test ever rendered — and the gap between a passing unit and a broken screen is invisible from the test report alone. When the environment genuinely cannot exercise the flow, say so explicitly and name what remains unverified rather than letting green stand in for proof.

**Thresholds must trip in the test.** When code enforces a limit — a rate limit, quota, timeout, retry cap, buffer size, pagination bound — configure the test with a value small enough to actually reach (a limit of 2–3, a timeout of milliseconds) and assert **both sides of the boundary**: within the threshold passes, the first case beyond it fails. Why: testing with production-scale thresholds means the enforcement branch never executes — the feature reads as covered while the code path that actually limits has never run once. A limit that has never tripped in a test is untested, whatever the coverage report says. This forces the threshold to be injectable rather than hardcoded — which is exactly the design pressure the test is supposed to apply.

→ `references/examples.md` §10 — reproduce-before-fix and trip-the-threshold examples — read when fixing a bug or testing a threshold.

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

→ `references/examples.md` §11 — the "fixing the test to match the bug" anti-pattern — read when a red test tempts you to edit the test.

## 12. Tests are first-class code

A test suite that's hard to read is a test suite nobody trusts. Apply [[code-standards]] to test code with the same discipline as production code: intention-revealing names, small focused helpers, guard clauses, no magic numbers, no commented-out tests. **One deliberate carve-out: in tests, descriptive beats DRY.** A test should read like a specification — a complete story without tracing through shared helpers — so duplication between tests is acceptable exactly when it makes each test independently understandable; extract a helper when it clarifies, never merely to deduplicate. If the test body is long enough to need scrolling, extract helpers.

---

## Self-check before you call it done

Run this against your own diff — the numbered sections above are the rest of the bar; these are the checks that have actually been skipped under pressure:

- Did you see each new test fail before the implementation and pass after — the failing test written first (or the spike discarded and rebuilt test-first)?
- Would every new or changed behavior's test fail if you deleted the implementation?
- If this was a bug fix: did a failing reproduction test exist *before* the fix?
- If the change enforces a threshold (limit, quota, timeout, cap): does a test configure a value small enough to trip it, asserting both sides of the boundary?
- If the change alters visible or interactive behavior: real flow exercised with evidence, or the unverified gap stated plainly — never a green suite standing in for proof?
- If any existing test changed: which legitimate case applies (spec change or implementation-detail cleanup), with zero weakening?
- Was every new test seen red for the reason it names — including any written after the code, proven by reverting the line it covers?
- Can the fixture tell the two behaviors apart, or would flipping one branch leave it green?
- If a flake was fixed: what was the measured rate before, and how many randomized repeats proved it after?

A passing test suite is only as trustworthy as the discipline behind it. If you're not confident the tests would catch a regression, they wouldn't.


## Reference files

| File | What it answers |
|------|-----------------|
| `references/examples.md` | Full worked code for every rule above — the red-green-refactor build, behavior-vs-implementation, AAA, clock injection, mock-only-at-seams, coverage theater, bug-fix reflex, trip-the-threshold, and the test-is-innocent anti-pattern |
