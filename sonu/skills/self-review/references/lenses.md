# Lens dispatch templates — the step-3b fan-out

Two tiers of lens — three **code lenses** and four **domain lenses** — one subagent each, dispatched **in parallel in a single turn** on the cheapest trustworthy executor tier below the session (per the model-tiering ladder). This file carries the prompts; **which lenses go out is decided by the dispatch conditions in `SKILL.md` step 3b**, which is their one home — don't restate or re-derive them here. Every lens is read-only and context-free: it gets the prompt below with the placeholders filled — never a summary of the conversation, never the author's intent. **Every lens prompt is self-contained:** the criteria live in the block itself. Do not tell a subagent to `Skill(…)` load or read plugin skill files — those live outside the customer repo root the shared frame supplies, and many harnesses give subagents no Skill tool.

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
```

## The prose frame (prepend on a prose-only diff, in place of the code framing)

`SKILL.md` step 3b routes a diff with no executable code down its prose path. The lens prompts below hunt code constructs, so on that path prepend this block to whichever lenses that step says are live — it replaces the code framing, and without it a lens has nothing usable to read.

```
This repo's product IS its documents: it ships Markdown that instructs a
model, so the prose in this diff is the behavior. Read a changed rule the
way you would read changed code — it is executed, by a reader, word for
word.

Read your lens's criteria below as being about the BEHAVIOUR THE PROSE
GOVERNS, not about program syntax: a "logic error" is a rule that
contradicts another rule or can never fire; a "silent behavior change" is
a rule whose default flipped with nothing announcing it; a "consumer" is
another file that cites this one's rules, sections, fields, or output
format. Report nothing about wording, tone, or formatting.
```

## The code lenses (append one per subagent)

Dispatched whenever the diff contains executable code — see `SKILL.md` step 3b, including what to do when it contains none.

**1. Correctness**
```
Your lens: CORRECTNESS. Logic errors only — wrong branch conditions,
off-by-ones, inverted comparisons, unhandled edge cases (empty, zero, nil,
boundary), broken invariants, error paths that swallow or misroute failures,
concurrency hazards. Trace each suspect path far enough to name the input
that breaks it.
Also hunt DUPLICATED JUDGMENT: two places independently deciding the
same thing — a guard and the code it guards, a filter and the transform
whose output it protects, a validator and a parser, a cap and the
accumulator it bounds. Ask whether the pair can disagree (different
inputs judged, different definitions of what counts, different handling
of separators or defaults) and report the pair plus the input on which
they diverge — flag pairs, not instances.
```

**2. Test adequacy**
```
Your lens: TESTS. New or changed behavior with no test exercising it,
tests asserting too weakly to catch the plausible regression, boundary
cases the tests skip, tests that pass for the wrong reason (over-mocked
seams, tautological assertions), thresholds/limits configured but never
tripped in any test. Name the specific untested input or path.
```

**3. Silent behavior change**
```
Your lens: SILENT CHANGES. Behavior that differs from before in a way no
error will ever surface — changed defaults, reordered operations with
observable effects, altered rounding/precision/timezone/locale handling,
different iteration or sort order callers may depend on, a caught-and-
defaulted failure path whose default now means something else. Compare
old and new behavior explicitly and name what a caller observes.
```

## The domain lenses (append one per subagent)

Dispatched only when the diff carries the lens's domain. The four conditions live in `SKILL.md` step 3b — read them there; a lens whose condition is not met is not dispatched at all, because there is nothing in its lens by construction.

**4. Security surfaces**
```
Your lens: SECURITY. Auth and permission checks (missing, reordered,
bypassable), input reaching a sink unsanitized (SQL, shell, path, HTML),
secrets or tokens in code/logs/errors, data exposure beyond what the caller
needs, unsafe defaults on security-relevant config. Name the attacker input
or sequence that exploits it.
```

**5. Data integrity & migration**
```
Your lens: DATA INTEGRITY. Schema changes and their compatibility with the
previous release's code, destructive or non-reversible writes, backfills
that can partially apply, missing transactions around multi-step writes,
truncation/precision/encoding loss, deletes without a recovery path. Name
what data is lost or corrupted and when.
```

**6. Blast radius & consumer impact**
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

**7. Interface**
```
Your lens: INTERFACE. On interface files in this diff only (components,
screens, templates, stylesheets, interface copy), report regressions the
diff introduces or worsens. Criteria are embedded here — do not load skills
or open plugin files. Read source and the diff only; do not run project
lifecycle scripts, installers, servers, or any state-mutating command. If
runtime verification would matter, say so in the Risk line as unverified.

ACCESSIBILITY — native control replaced by a non-semantic div/span without
the matching roles/keyboard handlers; missing or invisible focus rings;
keyboard traps or focus not restored on dismiss; hit targets the diff made
smaller than ~24×24 (or ~44×44 for touch primary actions); unlabeled
icon-only controls; errors that do not announce; status conveyed by color
alone; dynamic content with no live region; headings/landmarks broken so
structure is no longer navigable; layout that fails at 200% zoom / text
resize.

LAYOUT — controls that blend into content; misaligned edges on a shared
grid the file already uses; primary actions buried below secondary ones;
overflow/clipping that hides content or actions at supported widths;
targets cramped without breathing room; structure that collapses instead
of reflowing when content grows.

UX WRITING — buttons that are not verb-first or that misstate the
consequence; links whose text does not describe the destination; errors
that name the failure without a fix, or that sit far from the broken
field; empty states with no next step; placeholders used as the only
label; settings copy that describes the OFF state instead of ON.

TYPOGRAPHY — heading sizes that do not descend with level; line length
far past a readable measure on body text; truncated text with no title/
tooltip/expand escape; inputs below 16px on mobile (iOS zoom trap);
body text below a usable size/contrast floor; useful text made
unselectable.

COLORS — text/icon/border contrast that the diff made unreadable on its
surface; palette/token changes that break dark or light mode; status or
state encoded only as a hue change with no second cue.

UI POLISH — focus/hover/active affordances removed or invisible;
`transition: all`; animations that cannot be interrupted or that ignore
reduced-motion; loading/disabled states missing for async actions the
diff added; press/scale feedback that breaks layout; icon stroke weight
that no longer matches adjacent text.

Report a finding only when the diff itself introduces or worsens it. A
pre-existing interface problem the diff merely sits near is not a finding.
```

## Dispatch mechanics

- All dispatched lenses go out in one turn; they have no dependencies on each other. Every lens whose `SKILL.md` step-3b condition is met joins that same single batch, code and domain alike; one whose condition is not met is not dispatched at all.
- **A gated skip is not a degraded lens.** A degraded lens ran and came back unusable (next bullet); a gated lens never ran, because its domain is absent from the diff. Both are reported — the gated skips on `SKILL.md` step 5's single dispatch line, the degraded ones as their own risk entry — and they are reported as different things, because conflating them hides a real failure inside a routine one.
- Model: cheapest trustworthy executor tier below the session on the model-tiering ladder (its Provenance table is authoritative). No such tier → don't dispatch; the skill's step-2 rule already routed to the inline pass.
- A lens that errors or returns garbage is treated as "Nothing in my lens" **plus** a note in the final output that the lens was degraded — never silently counted as a clean pass.
- Lens replies are evidence, not verdicts. Every accept/reject happens in the session (SKILL.md step 4).
