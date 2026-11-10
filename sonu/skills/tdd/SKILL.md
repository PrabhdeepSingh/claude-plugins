---
name: tdd
description: >-
  Test-first rules: see each test fail before the code that passes it, prove after-the-fact tests by breaking the code, and never weaken a failing test. Use when writing or changing behaviour, fixing a bug or adding tests. Root-cause work: [[debugging]]; code quality: [[code-standards]].
argument-hint: "[feature, bug, or behavior to drive with tests]"
allowed-tools: Skill, Read, Write, Edit, Bash
---

# TDD — test first

The project's own instructions win: where its AGENTS.md, CLAUDE.md or established conventions say otherwise, follow them and skip that part of this.

Write the failing test, write the minimum code that passes it, then tidy with the test green. A test written first states what the code must do before any code can bias it, and it has been seen to fail, which is the only proof that it can.

**Find the project's commands first.** Read the manifest, the README or CONTRIBUTING file and the CI workflow, and prefer checked-in wrappers and repo scripts over global tools. Learn the command for one focused test (used during the loop) and the command for the tests the change can reach (run before completion). The full suite runs in the project's ship gate or CI, not from the agent's loop. Do not assume a default such as `npm test`: a wrong guess reads as "no tests exist".

**Invoked directly as `/sonu:tdd`:** apply this to `$ARGUMENTS`, the text typed after the invocation. If that token appears literally or is empty, apply it to the current change. For design, build and hand-off together, run `/sonu:build`.

## 1. Red, green, refactor

1. **Red.** Write a failing test for the next small behaviour. Run it and confirm it fails for the reason the test names. A test that passes before any implementation exists proves nothing.
2. **Green.** Write the simplest code that passes it.
3. **Refactor.** Tidy with the test green, then return to step 1.

When the same step fails twice, the step is too large. Return to the last green state, split the behaviour and take the smaller half. If that also stalls, hand it over with the last green state, the two attempts and what each ruled out; that is the summary [[debugging]] section 9 asks for.

After a green run, do not re-run the same command on unchanged code. The exception is a suspected flake, where repetition is the experiment.

## 2. Spikes and existing code

Throwaway code to learn an unfamiliar API is fine. Discard it and build the real thing test-first.

Before changing existing code that has no test, pin its current behaviour with a characterization test: call it, assert something deliberately wrong, read the failure to learn what it returns, and assert that. If this reveals a bug, pin the wrong behaviour, note it, and fix it as its own change, because callers may depend on it. The characterization test does not replace the test for the behaviour you are adding.

## 3. Behaviour, names and coverage

Assert observable outcomes, so a refactor that keeps behaviour does not break the test. Name each test for the behaviour and its condition. Coverage shows which lines ran, not what was verified; keep snapshots small enough to read in a diff.

## 4. Assert everything the action changed

One behaviour per test. After writing the action, ask what else it changed (a recomputed total, a flag, neighbours that should be untouched) and assert the one or two a regression would break. On an error path, assert the error's type or message and the cleanup that should have run; "it threw" passes when it threw for the wrong reason. Do not put an assertion inside a `try`/`catch` or an `if`, because an assertion that can be skipped passes either way.

## 5. Deterministic tests, and flakes

Tests do not share mutable state or depend on order, and the logic under test does not read the real clock or an unseeded random source; inject those.

A flaky test has a source of nondeterminism, and the fix removes it. Before fixing, measure the failure rate by running that one test on repeat, shuffled and in parallel where possible, and record it ("fails 23 of 1,000 runs"). Pin the seed for anything randomized. After fixing, prove it the same way with about twenty randomized repeats of that test plus one run of the affected tests, then remove the `retry`, `skip` or `flaky` annotation that was masking it. A sleep, a raised timeout, a retry wrapper or a skip is not a fix. If the interleaving that breaks the test can happen in production, the product has the race: fix the code and say so in the hand-off.

## 6. Test doubles at the seams only

Replace only what crosses a process boundary or has real-world consequences: a database, a payment gateway, the network, the clock. Do not mock your own domain objects. When a double is needed, prefer the real implementation, then a fake, then a stub, then a mock.

## 7. Which kinds of test

The kinds of test a project uses, and how many of each, are the project's choice. Follow its instructions and the shape of its existing suite. Where it says nothing, test through the same interface real callers use.

## 8. What to test

Test behaviour, boundaries (empty, maximum, zero, null), error paths and business rules. Skip trivial pass-throughs and generated code.

**Reproduce a bug before fixing it.** Write a test that reproduces the bug, confirm it fails, fix the bug, confirm it passes. This proves the fix and often shows the bug was more general than it looked.

**A test written after the code is proven by breaking the code.** A test that has never been red may not be able to fail. Run it green; revert only the implementation line it covers (invert the condition or return the empty value; not a bare `git stash`, which takes the uncommitted test with it); run again and confirm it fails for the reason the test names, not a missing import; restore and run green. If it stayed green, rewrite the assertion. One quick reading of any test: name what the implementation would have to return for it to pass. If `null`, `""`, `[]` or "no exception" passes it, it asserts nothing. When an existing test changed in the same diff, read that change first against section 9.

**Test data must be able to tell the answers apart.** Four fixture shapes make a test unable to fail: a boolean exercised in only one state (test the other); a collection with one element, which makes "first", "last" and "each" identical (use two with different values); two collaborators or fields stubbed to the same value (make them differ); a parameter with a default that every test passes explicitly (add a call that omits it). The question for a finished test is whether flipping one branch or reading the other field would turn it red.

**Boundaries another system defines get a table.** When code meets an external limit, an encoding, a timezone or calendar rule, or a platform difference such as path separators, write the boundary rows as parametrized cases first: the maximum, the maximum plus one, both ends of any interval, an impossible value, and for time a DST transition and a non-UTC zone.

**Visible behaviour needs observed evidence.** A green suite does not show that a screen renders or a flow completes. When a change alters UI, CLI output or an end-to-end flow, exercise the real flow and capture a screenshot, the terminal output or the response. When the environment cannot do that, say so and name what remains unverified.

**Thresholds must trip in the test.** For a rate limit, quota, timeout, retry cap or pagination bound, configure a value small enough to reach (a limit of 2 or 3) and assert both sides: within the limit passes, the first case beyond it fails. With production-scale values the enforcement branch never runs. This requires the threshold to be injectable.

## 9. When a test fails, the code is wrong until shown otherwise

Weakening a test to make it pass turns a visible failure into a hidden bug. These are not acceptable first responses to a red test: updating the expected value to what the code produces; deleting, skipping or quarantining the test; broadening the assertion until it cannot fail; adding sleeps or retries; wrapping the call so the assertion is not reached; mocking away the failing collaborator.

A test change is legitimate in two cases, and you say which: the specification changed (point to where that was decided), or the test asserted implementation details and a refactor broke it without changing observable behaviour.

## 10. Tests are code

Tests meet the same bar as production code ([[code-standards]]), with one exception: a test should read as a complete story, so duplication between tests is acceptable when it keeps each one understandable alone.
