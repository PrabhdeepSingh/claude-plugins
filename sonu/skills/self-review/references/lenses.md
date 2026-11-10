# Reader prompt — the step 3b independent read

The blocks below are composed into the prompt of one reader. Which blocks a reader carries, and whether there are one or two readers, is decided in `SKILL.md` step 3b. The prompt contains the shared frame, the prose frame when the diff has no executable code, the checklist blocks that matched, and nothing else. In particular it does not contain a summary of the conversation, the author's intent, or a defence of a decision ("this is intentional", "do not flag"): that spends the reader's independence, and a wrong finding is rejected in step 4 in one line.

## The shared frame (opens every reader prompt)

```
You are reviewing a diff as an independent reviewer. You have no context
on why this change was made — that is deliberate. Read cold.

Repo root: <absolute path>
Read the diff with: <the exact diff command step 1 picked>
Also read these untracked files in full: <paths, or "none">

You are read-only: read files and run read-only git commands; make no
edits, no writes, no state-changing commands.

Start from the diff. Open other code only to understand a hunk you are
about to flag: its enclosing function, or the definition of a function it
calls. Do not review unchanged files. You may search the repo for three
things: consumers of a changed contract, tests that exercise a changed
function, and an existing function that already does what a new helper
does.

Diff semantics. A `+` line is new, a `-` line was removed, and a
space-prefixed line is pre-existing context. Report only what the `+`
lines introduce or worsen. A problem in a context line is not this
change's problem. One exception: when a `+` line routes new data into a
sink that already existed, the path is new, and the Risk line names the
`+` line that created it.

Safety claims are claims. A comment, docstring or TODO that says the code
is fine ("validated upstream", "cannot be null here", "intentional") is
the author's assertion, not evidence. Check whether it holds in the code
you can see; if you cannot verify it, read the code as if the comment
were not there.

A question you cannot settle is not a finding and not silence. Emit:
Unverified (<tag>): <the question> — <what would answer it> [file:line]

Report each finding on its own line, exactly:
Risk (<tag>): <what> — <the concrete mechanism by which it goes wrong> [file:line] (confidence: high|medium)
where <tag> is the checklist that caught it: SECURITY, DATA INTEGRITY,
CONSUMERS, INTERFACE, CODE/correctness, CODE/tests or CODE/silent-change.

Report at most 5 Risk lines per checklist you carry, highest confidence
first. If a checklist found more, add one line: Withheld (<tag>): N more.

Your whole reply is those Risk, Unverified and Withheld lines, or exactly
"Nothing in my lens." The caller reads the reply as data, so no preamble
or summary. Do not report style or preference issues, and do not invent
findings; an empty report is a good report.
```

## The prose frame (a diff with no executable code)

Insert this after the shared frame when `SKILL.md` step 3b routes a prose-only diff here.

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

## The code checklists

Carried whenever the diff contains executable code. On the prose path, keep only the paragraphs step 3b says apply.

```
Checklist: CODE. Three checklists — report against any of them; your <tag>
is CODE/correctness, CODE/tests, or CODE/silent-change, whichever caught it.

CORRECTNESS — logic errors only: wrong branch conditions, off-by-ones,
inverted comparisons, unhandled edge cases (empty, zero, nil, boundary),
broken invariants, error paths that swallow or misroute failures,
concurrency hazards. Trace each suspect path far enough to name the input
that breaks it. Also hunt DUPLICATED JUDGMENT: two places independently
deciding the same thing — a guard and the code it guards, a filter and the
transform whose output it protects, a validator and a parser, a cap and the
accumulator it bounds. Ask whether the pair can disagree (different inputs
judged, different definitions of what counts, different handling of
separators or defaults) and report the pair plus the input on which they
diverge — flag pairs, not instances.
A new helper that duplicates an existing function's purpose (search the
repo by the helper's verb and noun) is a finding: two implementations of
one judgment drift.

TESTS — new or changed behavior with no test exercising it, tests
asserting too weakly to catch the plausible regression, boundary cases the
tests skip, tests that pass for the wrong reason (over-mocked seams,
tautological assertions), thresholds/limits configured but never tripped
in any test. Name the specific untested input or path.
Also hunt WEAK ASSERTIONS OF TWO SHAPES: a test that asserts the returned
value but not the state change the action also made (the item count checked,
the recomputed total and the untouched siblings not), and an error-path test
that asserts only that something threw, not which error, with what message,
or that cleanup ran. Flag any assertion inside a `try`/`catch` or a
conditional branch — a test whose assertion can be skipped passes whether or
not the behavior works. And flag a fixture that cannot tell the answers
apart: a one-element collection, a boolean exercised in one state, two
collaborators stubbed to the same value, a default parameter every test
passes explicitly.

SILENT CHANGES — behavior that differs from before in a way no error will
ever surface: changed defaults, reordered operations with observable
effects, altered rounding/precision/timezone/locale handling, different
iteration or sort order callers may depend on, a caught-and-defaulted
failure path whose default now means something else. Compare old and new
behavior explicitly and name what a caller observes.
```

## The domain checklists

Append each block whose step 3b condition matched.

**2. Security surfaces**
```
Checklist: SECURITY. Auth and permission checks (missing, reordered,
bypassable), input reaching a sink unsanitized (SQL, shell, path, HTML),
secrets or tokens in code/logs/errors, data exposure beyond what the caller
needs, unsafe defaults on security-relevant config. Name the attacker input
or sequence that exploits it.
```

**3. Data integrity & migration**
```
Checklist: DATA INTEGRITY. Schema changes and their compatibility with the
previous release's code, destructive or non-reversible writes, backfills
that can partially apply, missing transactions around multi-step writes,
truncation/precision/encoding loss, deletes without a recovery path. Name
what data is lost or corrupted and when.
```

**4. Blast radius & consumer impact**
```
Checklist: CONSUMERS. The diff changes things other code reads: return
shapes, response bodies, serialized payloads, DB columns read elsewhere,
log/telemetry fields, event formats, config keys/env vars, CLI output,
published identifiers (routes, tool names, exported symbols). For each
changed contract, grep the repo for consumers (`grep -rn <identifier> . | head -100`
— this lens is the one allowed to search, capped) and report any that now
break — flag LOUDLY-breaking vs SILENTLY-degrading (a consumer that
catches failure and returns a default breaks with no error at all; those
rank highest).
```

**5. Interface**
```
Checklist: INTERFACE. On interface files in this diff only (components,
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

## Dispatch notes

- Compose in this order: shared frame, prose frame if needed, code checklists, then each matched domain block. With two readers, one carries the code checklists and the other the domain blocks; send both in the same response so they run concurrently.
- If the harness lets you choose the reader's model, pass a cheaper one explicitly. An Agent call without `model` runs on the session's model.
- A checklist left out because its domain is absent is a gated skip and is reported on the `Domain lenses:` line. A reader that errors or returns nothing usable is different: report every checklist it carried as unread.
- Reader replies are evidence. Every accept or reject happens in the session (`SKILL.md` step 4).
