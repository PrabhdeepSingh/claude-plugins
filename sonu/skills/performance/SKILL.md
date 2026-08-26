---
name: performance
description: >-
  Performance work as measurement discipline — baseline first, one change at a time, beat the noise, and a keep/revert verdict where neutral is a revert. INVOKE when making anything faster or lighter, investigating slowness, or reviewing an optimization — "optimize", "too slow", "reduce latency/bundle size" — even mid-build. NOT for diagnosing functional bugs ([[debugging]]) or query-shape hygiene while writing ordinary code ([[code-standards]] §5).
argument-hint: "[what to make faster, or the optimization to verify]"
allowed-tools: Skill, Read, Write, Edit, Bash
---

# Performance — the measurement is the work

Most "optimizations" are guesses wearing confidence: a change lands because it *should* be faster, nobody measures, and the codebase accretes complexity that never bought anything. The discipline that prevents this is not knowing the tricks — it's the loop around them: measure a baseline, change one thing, re-measure the same way, and **keep only what provably paid**. Code you keep, you maintain forever; make it pay for itself.

## How to apply this

Run the loop in order: baseline → identify the bottleneck → one change → verify against the baseline → keep or revert → record the attempt. When invoked directly as `/sonu:performance`, apply it to `$ARGUMENTS` — the text typed after the invocation; if that token appears literally or is empty, apply it to the performance concern in the current discussion.

---

## 1. Baseline before touching anything

No baseline, no optimization — without a starting number, "faster" is a feeling. Capture the metric that matters to the user (page-load milestones, interaction latency, API p95, job duration, bundle bytes — whatever the complaint names), under stated conditions (dataset size, cache state, hardware/environment), with a fixed budget (sample count, wall-clock, or request count). Write the number down; the verdict in §4 is computed against it.

**Profile, don't deduce.** The bottleneck is where the time *measured* goes, not where the code looks slow — profilers, query plans, and waterfall traces exist because intuition about hot paths is reliably wrong. Route by symptom first: slow first render → network and render path; slow interaction → main-thread work; slow API → the server and its queries; then profile inside that region.

## 2. One change at a time

Land one optimization per measurement cycle. Three optimizations measured together produce one number that can't be attributed — if the total improved, you may be keeping two regressions paid for by one win ([[debugging]]'s one-change rule, applied to speed). Small, separately-verified changes also revert cleanly when §4 says revert.

## 3. Re-measure the way you measured

- **Same command, same conditions, same budget as the baseline.** A cold-cache baseline against a warm-cache result measures the cache, not the change.
- **Beat the noise, not the mean.** Repeat the measurement and compare the delta against run-to-run variance: a 3% gain inside ±5% variance is not a gain — it's a different sample. If the variance swallows plausible wins, reduce the noise (fixed inputs, quiet machine, more samples) before trusting any verdict.
- **Synthetic and field numbers answer different questions.** A synthetic benchmark catches regressions in CI and isolates causes; only field measurement (real users, real data) validates that a fix improved what users feel. Ship a "win" validated only synthetically as *provisional*, and say so.

## 4. The verdict: neutral is a revert

Correctness gates the metric — an "optimization" that wins by dropping work the product needed (a skipped validation, caching what must be fresh, removing an `await` that was load-bearing) is a regression with a good number. Then:

| Result vs. baseline | Action |
| --- | --- |
| Past the stated threshold, suite green | **Keep** — commit with the before/after numbers in the message |
| Within noise (no measurable change) | **Revert** |
| Worse | **Revert** |
| Improved, but a test went red | **Revert** — a regression wearing a win's clothing |

"Neutral is a revert" is the row teams skip: the change is already written, discarding it feels wasteful, so it lands unmeasured — and the codebase pays maintenance forever on complexity that bought nothing. Sunk cost is not a keep criterion; the measurement doesn't care how long the change took to write.

## 5. Record the attempts — including the reverted ones

Reverted work leaves no trace in git history, which is exactly why the same dead idea gets tried again next quarter. Keep a short ledger — in the PR description's risk section, or a `PERF.md` where the work is ongoing:

```
| Idea                        | Baseline → Result   | Verdict  | Why                                   |
| Memoize the row component   | INP 240ms → 235ms   | reverted | Inside noise (±15ms); rows weren't it |
| Virtualize the list         | INP 240ms → 90ms    | kept     | Long tasks gone from the trace        |
| Preconnect to the API host  | LCP 2.8s → 2.8s     | reverted | Already same-origin                   |
```

This is the same argument as [[design-tree]]'s preserved rejected branches: a discarded idea stays discarded only if the discard is recorded.

## 6. Guard what you won

A win nobody watches erodes one innocent commit at a time. Pin the metric that justified the work: a budget the pipeline checks (bundle bytes, a benchmark threshold with headroom for noise), or an alert on the field metric ([[observability]]'s symptom rules). And speculative defenses — memoizing everything, caching "just in case" — are §4 material like any other change: measured, or reverted.

---

## Self-check before you call the optimization done

- Is there a written baseline — metric, conditions, budget — and was the result measured the same way?
- One change per cycle, attributed to its own number?
- Does the delta beat the run-to-run variance, not just the mean — and is a synthetic-only win labeled provisional?
- Did every neutral or worse attempt actually get reverted — none kept for sunk cost?
- Suite green, with no work the product needed dropped for the number?
- Are the attempts — kept and reverted — recorded where the next person will look?
- Is the win guarded by a budget, threshold, or alert?

## Provenance and maintenance

The loop (baseline → one change → beat-the-noise → neutral-is-a-revert → ledger → guard) is durable and carries no external claims. Deliberately absent: named metric thresholds (Core-Web-Vitals-style numbers drift and are project-specific — the baseline defines "better" here) and per-tool profiler instructions (use whatever the project's stack provides). Nothing here should need re-verification against the outside world.
