---
name: intent-interview
description: >-
  Pre-spec intent extraction — one question at a time, each with a guess attached, until you can predict the user's answers; for when the asked-for artifact itself might be wrong. INVOKE when a request's outcome, audience, or success measure is genuinely unclear, before any spec or design. NOT ticket speccing ([[ticket-triage]]), choosing approaches ([[design-tree]]), or any non-interactive run — there an underspecified ask is a blocker, never a license to guess.
argument-hint: "[the vague request to interview toward]"
allowed-tools: Skill, Read
---

# Intent Interview — find out what they actually want before building anything

The most expensive misunderstanding is not a wrong implementation of the right thing — it's the right implementation of the wrong thing. A user who asks for "a dashboard for our metrics" may, two questions later, turn out to need a list. Different artifact, different scope, different work — and no amount of good speccing or good design recovers from interviewing zero. [[design-tree]]'s interview confirms the problem *behind a design*; this skill runs earlier and cheaper, for the case where the requested artifact itself is the hypothesis to test.

## When to apply this

Interactively, when a request names an artifact but not the outcome ("build me X" with the why, the audience, or the success measure unstated), or when you notice you could not predict how the user would answer basic questions about it. Skip it when intent, constraints, and success criteria are already stated — restate them and proceed — and **never run it in a non-interactive context** (CI, a factory pass, a subagent, a scheduled run): nobody is there to answer, so an underspecified ask is written into the hand-off as a blocker instead.

## How to apply this

### 1. Write a hypothesis, with a confidence number

Before asking anything, write (for yourself) what you believe they actually want and how confident you are. The honesty test for the number: **can you predict the user's reactions to the next three questions you would ask?** If not, the number is inflated. Below ~70%, name in one line what's missing — that gap is what the questions must close, and it keeps the interview aimed instead of ritual.

### 2. One question at a time, each with a guess attached

Ask exactly one question per message, and attach your current best guess to it ("Q: who reads this every day? My guess: the on-call engineer, during incidents"). Why one: the user can't react to hypotheses buried in a list, batches get skim-read, and the third question usually depends on the first answer. Why the guess: people correct a wrong guess faster than they compose an answer from nothing — and the guess commits you to being visibly wrong, which is the point. Occasionally guess in a direction you expect pushback on; a guess engineered to be agreed with tests nothing.

Three or more questions in one message is batching. A question with no hypothesis attached is surveying. Both waste the user's finite attention.

### 3. Detect "want" vs. "should-want"

Users often report the answer they think they're supposed to give: best-practice vocabulary ("scalable", "clean architecture"), deference to convention ("the way most apps do it"), and hedges ("I should probably…") are the tells. When you hear one, ask the question that does more work than the previous five:

> "If you didn't have to justify this to anyone, what would you actually want?"

The answer to *that* is the requirement; the earlier answer was its costume.

### 4. Restate in six lines, and get a real yes

When you believe you've converged, restate the intent in exactly this shape and end with "yes / no / refine?":

- **Outcome** — what exists or is true when this is done
- **User** — who it's for, specifically
- **Why now** — what prompted this
- **Success** — how they'll know it worked
- **Constraint** — the binding limit (time, stack, budget, compatibility)
- **Out of scope** — what is deliberately not being built

"Out of scope" is non-negotiable: half of all misalignment is silent disagreement about what is *not* being built.

### 5. Know what does not count as yes

- **"Whatever you think is best."** — delegation, which means *they* don't have confidence either. Re-ask as a choice between two concrete options.
- **"Sounds good."** — ambiguous. Ask: "anything you'd refine?" Silence isn't confirmation.
- **"Sure, let's go."** — often a polite exit. Confirm the one line you're least sure of before proceeding.
- **Silence, then "okay, start."** — they gave up on the interview, not converged. Say so and offer the restate again, shorter.

Proceed only on an answer that engages the restate's content — a correction counts; enthusiasm doesn't.

### 6. Stop conditions — both directions

**Stop asking** when the confidence test from step 1 passes: you can predict their reaction to the next three questions you'd ask. More questions past that point is procrastination with a questionnaire.

**Stop the interview** when several rounds pass and confidence isn't rising — that's information about the ask, not a cue to grind: "I've asked N questions and still can't predict your answers; something foundational is missing — want to step back and talk about the problem instead of the artifact?" And never save or act on the intent before the user has confirmed it: writing the doc first implies a yes they didn't give.

---

## Self-check before handing off to spec or design

- Did you write the hypothesis and confidence *before* the first question — and did the confidence actually rise as answers arrived?
- One question per message, every question carrying a visible guess?
- Did any "should-want" tell (best-practice talk, convention deference, "I should probably") get the actually-want question?
- Does the restate carry all six lines — including a real "Out of scope" — and did the user engage its content rather than wave it through?
- Was any hedge ("sounds good", "whatever you think") treated as a yes?
- If this ran anywhere non-interactive: it shouldn't have — was the underspecified ask flagged as a blocker instead?
