---
name: model-tiering
description: >-
  Tag each plan step with the cheapest model tier that can execute it reliably, then delegate tagged steps to subagents at execution — keeping an orchestrator-class session clean for judgment, integration, and review. INVOKE when writing an implementation plan or executing one whose steps carry [delegate]/[delegate-heavy] tags; it locates its own tier and no-ops when no trustworthy tier sits below. Not for design-only exploration with no plan artifact.
---

# Model Tiering — route the work, keep the judgment

A strong model's scarcest resource mid-build is clean context. Every mechanical, fully-specified step it executes itself dilutes the attention that architecture, integration, and review need — so the plan's author grades each step for the cheapest tier that can execute it reliably, and execution honors those grades with subagents. The motivation is quality via focus, not cost. The failure modes are symmetric, and the bar in Section 3 resolves both: doubt about any criterion keeps the work up; a step that clears all four goes down — hoarding it spends the exact context this skill exists to protect.

## How to apply this

Locate your tier first — before any step is tagged (Section 1). Then the skill fires at two moments:

- **While authoring a plan** — grade each step *in place* per Sections 2–4, tagging as you write. Don't defer tiering to a "who builds what" appendix: that retrofit framing is what produces all-or-nothing calls like "the session builds everything."
- **While executing a plan** whose steps carry tags — honor them per Section 5.

An orchestrating harness may carry the owner's standing disposition for a run — pre-authorize delegation, or suppress it entirely. Pre-authorization resolves borderline calls toward tagging *among steps that already clear all four criteria of Section 3*; it never relaxes those criteria, never overrides Section 4's categorical list, and cannot invent a tier that isn't there.

## 1. Know your position on the ladder

Identify the session model however the harness exposes it (in Claude Code the system prompt states it), find it on its **own model family's** ladder under Provenance, and orchestrate only if at least one trustworthy executor tier sits strictly below you. Otherwise note it in one line — "no executor tier below; executing inline" — and stop applying this skill: no tags, no subagents.

**Never route upward, and never across families.** The judgment this skill protects has to sit in the session — where the user's context and the approval gate are — not inside a stronger subagent that returns a summary; and a model whose family has no ladder under Provenance has no position, so it has no executor tiers. (Same-tier subagents exist only for `[delegate-heavy]` — Section 2.)

**The uncertainty asymmetry:** if you cannot locate yourself on the ladder with confidence, assume no tier sits below you and skip routing. A strong session that mistakenly skips loses only focus; a session that wrongly believes tiers sit below it delegates work it cannot be trusted to verify.

## 2. Two grades, session-relative mapping

Only delegable steps carry a marker — **an untagged step stays in the session**. Absence is safe by design: a plan written without this skill, an old plan, or a harness that ignores tags all behave exactly as today. Markers go on the step's first line, echoing the `[?]` bracket vocabulary plans already use.

- `[delegate]` — mechanical, transcription-grade work: tests from enumerated cases, renames, boilerplate, a stated pattern across listed files. Maps to the **cheapest trustworthy executor tier**.
- `[delegate-heavy]` — substantive-but-contained work: real logic to interfaces and constraints the plan has settled. Maps to the **strongest trustworthy tier strictly between the `[delegate]` tier and the session; when none sits in that gap, a same-tier (lateral) subagent.** Heavy work never lands on the tier the light grade already uses — capability must not drop on exactly the steps graded as needing the most. Lateral applies to this grade only.

```
3. [delegate] Add the `parse_totals` unit tests in tests/test_totals.py covering the
   three boundary cases listed in step 2. Verify: `pytest tests/test_totals.py` green.
5. [delegate-heavy] Implement src/cache/lru.py to the class signature in step 4.
   Verify: `pytest tests/cache/` green.
```

Grades — not model names — go in the plan: a concrete model name rots when the ladder moves; a grade stays portable across sessions and generations. Grades never map above the session tier. Worked mappings illustrate the current Provenance table — when the table changes, re-derive from it, never from an example.

## 3. What earns a tag, and which grade

Check Section 4 first — it removes whole classes of work no matter how they score here. For everything it leaves eligible, all four criteria are the *whole* test; doubt on any one removes the tag:

1. **Executor-ready** — the bar every plan step already owes ([[design-tree]]'s executor-ready section): exact paths, conventions settled in place, a verification check, no unmarked judgment.
2. **Self-contained** — the step's own text plus the files it names is everything the work needs. If the subagent's prompt would need conversation context, untag it.
3. **Mechanical verification** — the check is a command whose pass/fail the orchestrator runs and reads. "Looks right" disqualifies.
4. **Loud failure** — a wrong result trips the check immediately; no judgment-graded prose surface.

A step that clears all four carries a tag — that is the skill working. A worry that doesn't reduce to a specific failure of one criterion or a Section 4 category ("it's fiddly," "feels important") is not a disqualifier: the specification effort that made a step executor-ready is precisely what made it delegable.

**Grade rule:** pure transcription of decisions already written → `[delegate]`; authoring substantive new logic within settled boundaries → `[delegate-heavy]`. Doubt between grades → the heavier; doubt about delegability at all → no tag.

Avoid:

```
4. [delegate] Implement the caching layer per the design above.
```

(Depends on conversation context and hides judgment — fails criteria 1 and 2. The Section 2 example pair is the shape that passes.)

## 4. What never delegates

Categorical — no tag, regardless of Section 3 score:

- Steps containing, or downstream of, an unresolved `[?]` fork.
- Architecture and design decisions.
- Integration and wiring across components.
- Debugging and diagnosis.
- Security- or auth-sensitive work.
- **The review and verification of delegated output itself.** Checking a subagent's work is precisely the judgment this skill exists to protect — it never goes down the same chute.

## 5. Honoring tags at execution

For each tagged step, spawn a subagent (the Agent tool in Claude Code), model set per Section 2's mapping against the Provenance ladder. The prompt is the step's text verbatim plus the exact paths and settled conventions it names — nothing more. When it returns, **run the step's verification yourself**; never accept the subagent's own report. Delegation changes who types, not the bars: any discipline in force for the build — a failing test observed before the implementation, a standards constraint — still applies, and observing it stays in the session.

**Scheduling:** adjacent tagged steps with no dependency between them may run in parallel — independent slices, tests for already-implemented behavior, docs. Migrations, shared-state changes, and dependency chains stay sequential. Steps that share an API contract need the contract defined first; then they parallelize against it.

If the check fails: fix the specification if the failure was a specification gap and re-delegate **once** (a failing `[delegate]` may re-delegate at the heavier grade); otherwise take the step over inline. Never loop a subagent against a failing check.

**Degrade gracefully.** A harness with no subagent facility — or one that rejects the mapped model value — executes every step inline, in order: untagged and inline are the same behavior, so the plan needs no rewriting. A tag marks what *may* be delegated, never what must.

## Self-check before the plan goes to the gate

- Tier located first, at least one trustworthy tier strictly below — and no grade mapping above the session or across families (lateral only for `[delegate-heavy]`)?
- Every tagged step self-contained with a mechanical check, no `[?]` fork upstream, nothing from Section 4?
- Every step that clears all four criteria actually tagged — none held back on a hunch that isn't a specific criterion failure?
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
