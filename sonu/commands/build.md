---
description: Sequence decide → build → hand back for any feature or fix. Triage the change, run design-tree in plan mode (with you as the approval gate), build test-first under code-standards, then surface the riskiest parts before handing back to /sonu:ship. Never commits or merges.
argument-hint: "[feature, fix, or task to build]"
allowed-tools: Skill, Read, Write, Edit, Bash, EnterPlanMode, ExitPlanMode
---

# /build — decide → build → hand back

**Contract:** this command sequences the implementation lifecycle — design, build, self-review — and then hands back. It pauses **once** (for you to approve the design) and stops **once** (at a green suite, before shipping). It never commits, never merges. It delegates to existing skills; it does not re-implement them. Run `/sonu:ship` when you're ready to push.

Apply this to `$ARGUMENTS` — the text typed after the command. If that token appears literally or is empty, derive the task from the current discussion in context.

---

## Phase 0 — Triage

**Read the working tree:**
```bash
git status
git diff --stat HEAD
```

Classify the change on three axes and surface the result in **one line**:

- **Size** — trivial (≤~10 lines, no logic, e.g. comment, config, copy change) vs. substantial.
- **Kind** — bug fix, feature, refactor, docs, or mixed.
- **Surface** — touches web pages or public-facing content? → SEO bars apply in Phase 2.

**If trivial:** skip Phase 1 (no design needed). Jump straight to Phase 2.

---

## Phase 1 — Design (plan mode) + GATE

*Skip entirely for trivial changes.*

Enter plan mode to run the design phase. This is the only pause in the flow — your approval of the design is the gate.

1. `EnterPlanMode` — switches to read-only; the design tree will be written into the plan file.
2. `Skill(sonu:design-tree)` — interview first (2–4 questions to establish shared understanding, intent, constraints, done-when); then tree the real decision points. The tree lands in the plan file's `## Design Tree` section per design-tree's existing plan-mode behavior.
3. `ExitPlanMode` — **this is the gate.** Your approval of the plan = approval of the design. Do not proceed to Phase 2 until ExitPlanMode is called and approved.
   - **If the plan is rejected:** stay in plan mode, revise the design tree or re-interview the user to address the concern, and call `ExitPlanMode` again. Repeat until approved. Do not proceed to Phase 2 on a rejected plan.

After the gate, you are back in execution mode (writes are legal again).

---

## Phase 2 — Build

Build the change test-first under the active quality bars:

1. `Skill(sonu:tdd)` — drive the implementation with the red-green-refactor loop. Write the failing test first; write the minimum code to pass; refactor under green. Apply `Skill(sonu:code-standards)` as you go.
2. If the surface is web / public-facing, apply `Skill(sonu:seo-standards)` and/or `Skill(sonu:content-seo)` as appropriate.
3. **Run the suite via `Bash`.** Don't take green on faith:
   ```bash
   # derive the right command from the repo's package.json / Makefile / README
   # e.g.: npm test / pytest / go test ./... / cargo test
   ```
   If tests are red, fix and re-run until green. Do not proceed while the suite is failing.

---

## Phase 3 — Self-review + hand back

1. `Skill(sonu:self-review)` — list the 3–5 riskiest things in the diff in plain language. A pointer for your review, not a score or an approval.
2. Print a brief **built summary**:
   - What was built (one sentence).
   - Diff stat: `git diff --stat HEAD`.
   - The self-review risk list from step 1.
3. Stop with:

> **Green and ready.** Review the diff, then run `/sonu:ship` when you're satisfied.

Do not commit. Do not merge. The turn ends here.

---

## Pitfalls

- **Don't re-implement skills.** If design-tree or tdd need to do something, invoke them — don't inline their logic here.
- **Don't commit.** Phase 3 ends at the hand-back message. Committing and shipping are deliberate acts that belong to `/sonu:ship`.
- **Coexistence.** `/sonu:design-tree`, `/sonu:tdd`, and `/sonu:ship` work identically when run standalone. This conductor is additive, not a replacement.
- **Trivial changes skip plan mode.** Entering plan mode for a one-line fix is overhead that blocks write tools unnecessarily. Use the size classification in Phase 0.
