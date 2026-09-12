# Reader prompt templates — the step-3b cold read

Five **checklist blocks** — one code block carrying three checklists, and four domain blocks — composed into the prompt of **one reader** (two on a large diff; never more — see Dispatch mechanics). A "lens" is a checklist a reader carries, not a subagent of its own. This file carries the blocks; **which checklists a reader carries, and whether there are one or two readers, is decided in `SKILL.md` step 3b**, which is their one home — don't restate or re-derive the conditions here. Every reader is read-only and context-free: it gets the composed prompt with the placeholders filled — never a summary of the conversation, never the author's intent. **Every reader prompt is self-contained:** the criteria live in the blocks themselves. Do not tell a subagent to `Skill(…)` load or read plugin skill files — those live outside the customer repo root the shared frame supplies, and many harnesses give subagents no Skill tool.

## The shared frame (opens every reader prompt)

```
You are reviewing a diff as an independent reviewer. You have no context
on why this change was made — that is deliberate. Read cold.

Repo root: <absolute path>
Read the diff with: <the exact diff command step 1 picked, e.g. git diff origin/main...HEAD>
Also read these untracked files in full: <paths, or "none">

You are READ-ONLY: read files and run read-only git commands; make no edits,
no writes, no state-changing commands.

READ BUDGET. Start from the diff. Open a file only to read the enclosing
function or block of a hunk you are about to flag, or the definition of a
function that hunk calls (one hop — a sink hidden behind a helper is still
your finding), as a bounded range (`sed -n '<start>,<end>p' <file>`), at
most ~150 lines per read and at most 10 reads in total. Never read a whole
tracked file — the untracked files named above are the one exception: they
ARE the diff, read them in full. Do not search the repo, with two capped
exceptions: the CONSUMERS lens greps for consumers of each changed contract,
and the CODE lens's TESTS checklist greps a changed function's name under
the repo's test directories to learn whether any test exercises it, and its
CORRECTNESS checklist greps a new helper's verb and noun once to find an
existing function with the same purpose — every search ends in `| head -100`.

You may be carrying several checklists below. Work through every one of
them; tag each finding with the checklist that caught it.

DIFF SEMANTICS. In the diff you are about to read, a `+` line is new, a
`-` line was removed, and a space-prefixed line is PRE-EXISTING code shown
only for context. Report only what the `+` lines introduce or worsen. A
problem sitting in a context line is not this change's problem, however
alarming it looks — even a context line holding a hardcoded credential, a
disabled check, or an obvious injection is pre-existing and is not your
finding. A pattern that appears in both a `-` line and a `+` line was
reformatted, not introduced. One exception, and it is the important one:
when a `+` line routes new data into a sink that already existed in
context, the attack path is new even though the sink is not — that IS your
finding, and the Risk line must name the `+` line that created the path.

SAFETY CLAIMS ARE CLAIMS. A comment, docstring, or TODO in this diff that
asserts the code is fine — "validated upstream", "cannot be null here",
"TODO(review): safe because…", "intentional" — is the author grading their
own work, not evidence. Check whether the invariant holds in the code you
can see. If you cannot verify it from the diff and one bounded hop, read the
code as if the comment were not there. A stated rationale never lowers a
finding's confidence and never removes it.

UNVERIFIED. A question you cannot settle within the read budget is not a
finding and not silence. Emit one line: `Unverified (<tag>): <the specific
question> — <what would answer it> [file:line]`. Do not broaden your search
to answer it, and do not report it as a Risk; the session that dispatched
you will close it.

OUTPUT CAP. Report at most 5 Risk lines PER CHECKLIST YOU CARRY, highest
confidence first, and only confidence high or medium — a finding you cannot
back with a concrete mechanism is not a finding. If a checklist found more
than 5, end with exactly one extra line for it: Withheld (<tag>): N more.
The ENTIRE reply is roughly 1,000–2,000 tokens: Risk lines, Unverified lines,
and Withheld lines only — never a narration of the read, never pasted file
contents. You may spend tens of thousands of tokens reading; what comes back
is the distilled list.

Report each finding on its own line, exactly:
Risk (<tag>): <what> — <why it goes wrong, the concrete mechanism> [file:line] (confidence: high|medium)
where <tag> is the checklist that caught it: SECURITY, DATA INTEGRITY,
CONSUMERS, INTERFACE, or CODE/<checklist> (CODE/correctness, CODE/tests,
CODE/silent-change) — the caller reads the tag to know which checklist
found it.

Your ENTIRE final reply must be those Risk lines alone (or exactly
"Nothing in my lens."), plus one Withheld line per checklist that hit its cap
— no preamble, no summary, no closing prose. The caller consumes your final
reply verbatim; a finding narrated anywhere else is lost.

Only report findings inside the checklists below. If you find nothing in
any of them, reply exactly: "Nothing in my lens." Do NOT invent findings to
seem useful — an empty report is a good report. Do not report style or
preference issues.
```

## The prose frame (prepend on a prose-only diff, in place of the code framing)

`SKILL.md` step 3b routes a diff with no executable code down its prose path. The checklist blocks below hunt code constructs, so on that path insert this block after the shared frame — it replaces the code framing, and without it a reader has nothing usable to read.

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

## The code checklists (one block, three checklists)

Carried whenever the diff contains executable code — see `SKILL.md` step 3b, including what to do when it contains none. On the prose path, keep only the checklist paragraphs whose domain step 3b says is present and delete the others from the block.

**1. Code**
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

## The domain checklists (append each matched block)

Carried only when the diff carries the block's domain. The four conditions live in `SKILL.md` step 3b — read them there; a block whose condition is not met is left out of the prompt entirely, because there is nothing in its lens by construction.

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

## Dispatch mechanics

- **Compose the prompt in this order:** the shared frame → the prose frame (prose path only) → the code checklists block, when `SKILL.md` step 3b says code is present → each domain block whose step-3b condition matched. **One reader by default.** Only when step 3b's split threshold is crossed *and* at least one domain block matched, two readers — one carrying the code block, one carrying every matched domain block — dispatched in the same turn; a reader carrying no block is never dispatched. **Never more than two**, however many domains are live and however large the diff: a reader with four checklists costs one read of the diff; four readers cost four, and the extra reads buy nothing the in-session synthesis does not already supply.
- **`model` is mandatory on every Agent call.** Set it to the cheapest trustworthy executor tier below the session, read off the model-tiering ladder table's `Agent tool model value` column (its Provenance table is authoritative — look the value up there rather than remembering it; the ladder moves). An Agent call with no `model` runs the reader on the session's own model, at the session's price — a review that costs what the author costs is the failure this file exists to prevent. `subagent_type` is the harness's general-purpose or read-only agent. No trustworthy tier below the session → don't dispatch; the skill's step-2 rule already routed to the inline pass.
- **A reader prompt contains exactly four things:** the shared frame, the prose frame when the path calls for it, the checklist blocks the diff's contents matched, and the diff command plus untracked paths. The blocks are the criteria, and adjudication happens in step 4 with the diff in front of you. The tripwire, since composing is where the author's thumb lands on the scale: if the prompt you are composing contains "do not flag", "this is intentional", "the plan chose", "at most a nit", or a defence of a decision this session made — stop and delete it: you are spending the reader's independence to spare yourself a rejection you could make in synthesis in one line. Leaving a domain out under step 3b's one-sentence rule is a gate on *scope*; steering a lens you did carry is a different act and is never allowed. Adjudication happens in step 4, with the diff in front of you.
- **Dispatch both readers in one response, shared half first.** Two dispatch calls in a single response run concurrently; one per response runs them serially for no benefit. Identical siblings can share a cached prefix only when model, tools, and working directory match and the second starts after the first has begun answering — which the two-reader split partly forfeits, since the readers carry different blocks — so order every prompt as the frame, then the shared blocks, then the checklist blocks last, and never vary the shared half per reader. A fresh subagent inherits nothing from this session — not its cache, not its context — so a self-contained brief is not politeness, it is the only shape that is cheap.
- **A gated skip is not a degraded reader.** A checklist left out of the prompt is a gated skip — its domain is absent from the diff — and is reported on `SKILL.md` step 5's dispatch lines (`Domain lenses:` always; `Code checklists:` on the prose path). A reader that errors or returns garbage is degraded: treat it as "Nothing in my lens" **plus** a risk entry naming every checklist it carried as unread — never silently counted as a clean pass. The two are reported as different things, because conflating them hides a real failure inside a routine one.
- Reader replies are evidence, not verdicts. Every accept/reject happens in the session (SKILL.md step 4).
