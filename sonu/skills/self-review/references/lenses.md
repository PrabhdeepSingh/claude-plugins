# Lens dispatch templates — the step-4b fan-out

Six lenses, one subagent each, dispatched **in parallel in a single turn** on the cheapest trustworthy executor tier below the session (per the model-tiering ladder). Every lens is read-only and context-free: it gets the prompt below with the placeholders filled — never a summary of the conversation, never the author's intent.

## The shared frame (include in every lens prompt)

```
You are reviewing a code diff as an independent reviewer. You have no context
on why this change was made — that is deliberate. Read cold.

Repo root: <absolute path>
Read the diff with: <the exact diff command step 1 picked, e.g. git diff origin/main...HEAD>
Also read these untracked files in full: <paths, or "none">

You are READ-ONLY: read files and run read-only git commands; make no edits,
no writes, no state-changing commands.

Report each finding on its own line, exactly:
Risk: <what> — <why it goes wrong, the concrete mechanism> [file:line] (confidence: high|medium|low)

Only report findings inside your lens (below). If you find nothing, reply
exactly: "Nothing in my lens." Do NOT invent findings to seem useful — an
empty report is a good report. Do not report style or preference issues.

<if learned rules were loaded in step 3, append:>
Additionally check the diff against these learned rules; report a violation
as a normal Risk line:
<the scope-matched rule lines>
```

## The six lenses (append one per subagent)

**1. Correctness**
```
Your lens: CORRECTNESS. Logic errors only — wrong branch conditions,
off-by-ones, inverted comparisons, unhandled edge cases (empty, zero, nil,
boundary), broken invariants, error paths that swallow or misroute failures,
concurrency hazards. Trace each suspect path far enough to name the input
that breaks it.
```

**2. Security surfaces**
```
Your lens: SECURITY. Auth and permission checks (missing, reordered,
bypassable), input reaching a sink unsanitized (SQL, shell, path, HTML),
secrets or tokens in code/logs/errors, data exposure beyond what the caller
needs, unsafe defaults on security-relevant config. Name the attacker input
or sequence that exploits it.
```

**3. Data integrity & migration**
```
Your lens: DATA INTEGRITY. Schema changes and their compatibility with the
previous release's code, destructive or non-reversible writes, backfills
that can partially apply, missing transactions around multi-step writes,
truncation/precision/encoding loss, deletes without a recovery path. Name
what data is lost or corrupted and when.
```

**4. Blast radius & consumer impact**
```
Your lens: CONSUMERS. The diff changes things other code reads: return
shapes, response bodies, serialized payloads, DB columns read elsewhere,
log/telemetry fields, event formats, config keys/env vars, CLI output,
published identifiers (routes, tool names, exported symbols). For each
changed contract, grep the repo for consumers and report any that now
break — flag LOUDLY-breaking vs SILENTLY-degrading (a consumer that
catches failure and returns a default breaks with no error at all; those
rank highest).
```

**5. Test adequacy**
```
Your lens: TESTS. New or changed behavior with no test exercising it,
tests asserting too weakly to catch the plausible regression, boundary
cases the tests skip, tests that pass for the wrong reason (over-mocked
seams, tautological assertions), thresholds/limits configured but never
tripped in any test. Name the specific untested input or path.
```

**6. Silent behavior change**
```
Your lens: SILENT CHANGES. Behavior that differs from before in a way no
error will ever surface — changed defaults, reordered operations with
observable effects, altered rounding/precision/timezone/locale handling,
different iteration or sort order callers may depend on, a caught-and-
defaulted failure path whose default now means something else. Compare
old and new behavior explicitly and name what a caller observes.
```

## Dispatch mechanics

- All six go out in one turn; they have no dependencies on each other.
- Model: cheapest trustworthy executor tier below the session on the model-tiering ladder (its Provenance table is authoritative). No such tier → don't dispatch; the skill's step-2 rule already routed to the inline pass.
- A lens that errors or returns garbage is treated as "Nothing in my lens" **plus** a note in the final output that the lens was degraded — never silently counted as a clean pass.
- Lens replies are evidence, not verdicts. Every accept/reject happens in the session (SKILL.md step 5).
