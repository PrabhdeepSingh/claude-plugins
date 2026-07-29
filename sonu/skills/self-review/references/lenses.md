# Lens dispatch templates — the step-4b fan-out

Six lenses, one subagent each, plus a seventh **interface** lens dispatched only when the diff touches user-facing interface files — all **in parallel in a single turn** on the cheapest trustworthy executor tier below the session (per the model-tiering ladder). Every lens is read-only and context-free: it gets the prompt below with the placeholders filled — never a summary of the conversation, never the author's intent.

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

Your ENTIRE final reply must be those Risk lines alone (or exactly
"Nothing in my lens.") — no preamble, no summary, no closing prose. The
caller consumes your final reply verbatim; a finding narrated anywhere
else is lost.

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

## The conditional seventh lens — interface

Dispatch this one **only when** the diff touches user-facing interface files: components, screens, templates, stylesheets, or interface copy. On a diff with no such files, don't dispatch it — there is nothing in its lens by construction.

```
Your lens: INTERFACE. Apply the sonu:interface-review methodology in `quick`
mode, scoped to the interface files in this diff, and report only what it
finds. Cover its six domains through their owning skills — accessibility,
layout, ux-writing, typography, colors, ui-polish.

OVERRIDE, and this matters: ignore interface-review's own Review Output
Format entirely — no Scope and Coverage section, no severity table, no
Considered-but-Rejected table, no Verdict. Report findings ONLY as the
shared Risk lines defined above. A verdict cannot be consumed here.

Also override interface-review's "Verify What Can Be Verified" step: do not
run project lifecycle scripts, installers, servers, or any command that
mutates state. Read source and the diff only; if runtime verification would
matter, say so in the Risk line as unverified, not by executing it. The
shared read-only frame above wins over that methodology step.

Report a finding only when the diff itself introduces or worsens it. A
pre-existing interface problem the diff merely sits near is not a finding.
```

## Dispatch mechanics

- All lenses go out in one turn; they have no dependencies on each other. The interface lens joins the same batch when its condition is met.
- Model: cheapest trustworthy executor tier below the session on the model-tiering ladder (its Provenance table is authoritative). No such tier → don't dispatch; the skill's step-2 rule already routed to the inline pass.
- A lens that errors or returns garbage is treated as "Nothing in my lens" **plus** a note in the final output that the lens was degraded — never silently counted as a clean pass.
- Lens replies are evidence, not verdicts. Every accept/reject happens in the session (SKILL.md step 5).
