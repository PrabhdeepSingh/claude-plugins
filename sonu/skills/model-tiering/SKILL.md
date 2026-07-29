---
name: model-tiering
description: >-
  Tag each plan step with the cheapest model tier that can execute it reliably, then delegate tagged steps to subagents at execution — keeping an orchestrator-class session clean for judgment, integration, and review. INVOKE when writing an implementation plan or executing one whose steps carry [delegate]/[delegate-heavy] tags — on every session model; it locates its own tier and no-ops when no trustworthy tier sits below (never escalates upward). Not for design-only exploration with no plan artifact.
---

# Model Tiering — route the work, keep the judgment

A strong model's scarcest resource mid-build is clean context. Every mechanical, fully-specified step it executes itself — boilerplate, enumerated test cases, pattern application across listed files — dilutes the attention that architecture, integration, and review actually need. This skill routes that work down the capability ladder: the plan's author decides, step by step, the cheapest model tier that can execute each step reliably, and execution honors those decisions with subagents. The motivation is quality via focus, not cost — when in doubt, the work stays up. But the reverse is a failure too: leaving a step that plainly clears the bar in the session spends the exact context this skill exists to protect on typing a subagent could have done. The caution governs genuine doubt; it does not reach steps that plainly qualify — those get tagged. An orchestrating harness may also carry the owner's standing disposition for a run — pre-authorize delegation or suppress it entirely (keep every step in-session). Pre-authorization resolves borderline calls in favor of tagging *among steps that already clear all four criteria of Section 3* — it does not relax the rule that doubt about any criterion removes the tag, and it never overrides Section 4's categorical list or the no-executor-tier-below no-op: it cannot delegate design, integration, debugging, security, or the review of delegated output, and cannot invent a tier that isn't there.

## How to apply this

Locate your tier first — before any step is tagged. You may orchestrate only if at least one trustworthy executor tier sits strictly below your session model on the ladder (see Provenance). If none does, note it in one line — "Session model has no executor tier below it; executing inline without model routing" — and stop applying this skill: write and execute the plan exactly as you would without it, no tags, no subagents. **Never route upward.** A session must not spawn a subagent above its own tier to compensate — the judgment this skill protects has to sit in the session itself, where the user's context and the approval gate are, not inside a subagent that returns a summary. (Same-tier subagents are allowed only for `[delegate-heavy]` steps on a qualifying orchestrator session — see Section 2; a session that doesn't qualify skips routing entirely.)

The skill then applies at two moments:

- **While authoring a plan** — grade each step *in place* per Sections 2–4 and tag the delegable ones as you write them. Don't defer tiering to a prose "who builds what" section appended at the end; that retrofit framing is what produces all-or-nothing calls like "the session builds everything" — grading step by step is what surfaces that most steps delegate and only a few stay up.
- **While executing a plan** whose steps carry tags — honor them per Section 5.

## 1. Know your position on the ladder

Identify the session model however the harness exposes it — in Claude Code the system prompt states it (e.g. "You are powered by the model named …"); other harnesses may state it elsewhere or not at all. Find that model on its **own model family's** capability ladder under Provenance; your executor tiers are everything strictly below you on that ladder that clears the trustworthy bar.

**Never route across families** — a subagent always runs a model from the session's own family. A model whose family has no ladder under Provenance — including any non-Claude model a multi-model harness may be running — has no position and therefore no executor tiers: skip routing. (Provenance currently carries the Claude family ladder only; a new family is supported by adding its verified ladder there, with no change to this section.)

Names and ladder order are volatile facts whose only authoritative home is the Provenance table. Worked examples elsewhere in this skill illustrate the current table; when the table changes, re-derive from it — never treat an example as a second source.

**The uncertainty asymmetry:** if you cannot locate yourself on the ladder with confidence, assume no tier sits below you and skip routing. A strong session that mistakenly skips loses nothing but focus; a session that wrongly believes tiers sit below it delegates work it cannot be trusted to verify.

## 2. Two grades, session-relative mapping

Only delegable steps carry a marker; **an untagged step stays in the session**. That direction is deliberate: absence is safe. A plan written without this skill, an old plan, or a harness that ignores tags all behave exactly as today — a missing tag can never accidentally delegate; only a deliberately placed one can. Markers go on the step's first line, echoing the `[?]` bracket vocabulary plans already use for open forks.

- `[delegate]` — mechanical, transcription-grade work: tests from enumerated cases, renames, boilerplate, applying a stated pattern across listed files. Maps to the **cheapest trustworthy executor tier** on the ladder.
- `[delegate-heavy]` — substantive-but-contained work: writing real logic to interfaces and constraints the plan has already settled (a module to given signatures, an algorithm to a stated contract). Maps to the **strongest trustworthy tier sitting strictly between the `[delegate]` tier and the session model; when no tier sits in that gap, a same-tier (lateral) subagent**. Heavy work never lands on the tier the light grade already uses — the point is offloading the session's context, and capability must not drop on exactly the steps graded as needing the most. Lateral applies to this grade only — `[delegate]` work is transcription-grade by definition and never needs session-grade capability.

```
3. [delegate] Add the `parse_totals` unit tests in tests/test_totals.py covering the
   three boundary cases listed in step 2. Verify: `pytest tests/test_totals.py` green.
5. [delegate-heavy] Implement src/cache/lru.py to the class signature in step 4.
   Verify: `pytest tests/cache/` green.
```

Worked mapping, illustrating the current Provenance ladder (the table is authoritative — re-derive these if it changes): on a Fable session, `[delegate]` → Sonnet and `[delegate-heavy]` → Opus; on an Opus session, `[delegate]` → Sonnet and `[delegate-heavy]` → an Opus subagent (lateral — Sonnet is the only trustworthy tier below, and it is already where the light grade goes; forcing heavy work down to it would trade quality for cost). Grades never map above the session tier.

Grades — not model names — go in the plan. A tag naming a concrete model rots when the ladder moves and breaks when a different session executes the plan; a grade stays portable across sessions and model generations.

## 3. What earns a tag, and which grade

All four must hold for either grade; doubt on any one removes the tag entirely:

1. **It clears the executor-ready bar** every plan step already owes ([[design-tree]], its executor-ready plans section) — exact paths, conventions settled in place, a verification check, no unmarked judgment. That bar is defined there, not here.
2. **It is self-contained.** The step's own text plus the files it names is everything the work needs. If writing the subagent's prompt would require adding context from the conversation, the step was not delegable — untag it.
3. **Its verification is mechanical.** The step's check is a command whose pass/fail the orchestrator can run and read. "Looks right" disqualifies.
4. **Failure is loud.** A wrong result trips the check immediately; the step's output has no silent near-miss surface (no judgment-graded prose).

These four operate *within* Section 4's categorical exclusions, not around them: Section 4 removes whole classes of work (design, integration, debugging, security/auth, review of delegated output) from delegation no matter how they score here — check it first. For everything Section 4 leaves eligible, the four criteria are the *whole* test. A step that clears all four **should** carry a tag — that is the skill working, not optional caution to be overridden by a hunch. Do not withhold a tag on vague grounds outside the four ("it's fiddly," "pattern-heavy," "feels important"): if a worry doesn't reduce to a specific failure of criterion 1, 2, 3, or 4 — or to a Section 4 category — it is not a disqualifier. In particular, the effort you spent making a step executor-ready is what *makes* it delegable — once its paths, conventions, and check are settled in place, "the patterns are tricky" no longer holds, because the patterns are now written down (criterion 2). A step is kept up because it fails a criterion or falls in Section 4, never because specifying it well made it feel important.

**Grade rule:** if the step is pure transcription of decisions already written in the plan, it's `[delegate]`; if the subagent must author substantive new logic within the step's settled boundaries, it's `[delegate-heavy]`. Doubt between grades → the heavier grade. Doubt about delegability at all → no tag.

Avoid:

```
4. [delegate] Implement the caching layer per the design above.
```

(Depends on conversation context the subagent doesn't have, and hides judgment inside the step — fails criteria 1 and 2.)

Prefer:

```
4. [delegate-heavy] Implement src/cache/lru.py to the class signature in step 3:
   capacity from config key cache.max_entries, evict least-recently-used on insert
   past capacity. Verify: `pytest tests/cache/` green.
5. [delegate] Add the three boundary-case tests enumerated in step 3 to
   tests/cache/test_lru.py. Verify: `pytest tests/cache/test_lru.py` green.
```

## 4. What never delegates

Categorical — no tag on any of these, regardless of how well they score against Section 3:

- Steps containing, or downstream of, an unresolved `[?]` fork.
- Architecture and design decisions.
- Integration and wiring across components.
- Debugging and diagnosis.
- Security- or auth-sensitive work.
- **The review and verification of delegated output itself.** Checking a subagent's work is precisely the judgment this skill exists to protect — it never goes down the same chute.

## 5. Honoring tags at execution

For each tagged step, spawn a subagent with the harness's subagent tool (the Agent tool in Claude Code), its model set per the grade mapping in Section 2 against the ladder in Provenance. The subagent's prompt is the step's text verbatim plus the exact paths and settled conventions the step names — nothing more. When it returns, **run the step's verification check yourself**; never accept the subagent's own report of success. Adjacent tagged steps with no dependency between them may run in parallel.

Delegation changes who does the typing, not the bars the build is under. Any discipline in force for the build — a failing test observed before the implementation that makes it pass, a standards skill's constraint — still applies to delegated steps, and observing it stays in the session: when a delegated step's tests exist before its implementation, run them yourself and see them fail before you delegate the implementation step.

If the check fails: fix the specification if the failure was a specification gap and re-delegate **once** (a failing `[delegate]` step may re-delegate at the heavier grade); otherwise take the step over inline. Never loop a subagent against a failing check.

**Degrade gracefully.** Tagging and honoring are independent: a plan authored in one harness may be executed in another. If the executing harness has no facility for spawning a subagent on a chosen model (Cursor has no equivalent of Claude Code's Agent tool with a model override; restricted sessions may lack it too), tags are advisory: execute every step inline, in order — the plan needs no rewriting, because untagged and inline are the same behavior. A tag marks what *may* be delegated, never what must be.

## Self-check before the plan goes to the gate

- Did you locate your tier before tagging, and does at least one trustworthy tier sit strictly below you on your family's ladder?
- Is every tagged step self-contained, with a mechanical verification check?
- Is no `[?]` fork upstream of any tagged step?
- Does no tag sit on design, integration, debugging, or security work?
- Does no grade map above the session tier or outside the session's model family, and is lateral used only for `[delegate-heavy]`?
- When torn between grades, did you pick the heavier one — and drop the tag entirely when delegability itself was in doubt?
- Is every step that clears all four criteria actually tagged — or did a bar-clearing step get kept in-session on a hunch that isn't a specific failure of one of the four?
- Would the plan execute identically, just slower, if every tag were ignored?
- Does the review of delegated output stay in the session?

## Provenance and maintenance

The methodology above — tier by ladder position, the four-part delegation bar, orchestrator-verifies, absence-is-safe — is durable. The table below is not: model names, ladder order, and the subagent tool's accepted model values drift with harness releases. Last verified 2026-07; re-verify against the harness's current model listing and its subagent-tool documentation whenever a new model generation lands. One ladder per model family; only the Claude family ships today — another family (OpenAI, Gemini) is added by appending its verified ladder here, with no change to the sections above.

**Claude family ladder:**

| Ladder (strongest first) | Agent tool `model` value | Trustworthy executor? |
|---|---|---|
| Fable | `fable` | yes — executor tier when the session sits above it |
| Opus | `opus` | yes — executor tier when the session sits above it |
| Sonnet | `sonnet` | yes |
| Haiku | `haiku` | no — below this skill's bar (deliberate omission) |

A model is an *executor tier* only relative to a stronger session; the same row serves both roles. Fable appears for ladder completeness — nothing currently sits above it, so no session routes to it.
