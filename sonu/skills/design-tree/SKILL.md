---
name: design-tree
description: Make design decisions as an explicit branching tree instead of one linear narrative — interviewing the user for shared understanding first, then weighing real alternatives at each decision point. INVOKE THIS PROACTIVELY whenever planning or designing an implementation approach, weighing how to build a feature, choosing between architectures / libraries / data models / APIs, or working in plan mode — even when the user never says "design tree."
---

# Design Tree — decide by branching, not by marching

Design isn't a straight line from question to answer. It's a tree: each decision point is a node with multiple branches; you pick one, sometimes backtrack, and move forward. The genealogy of what you chose *and what you didn't* is the record that keeps decisions from getting silently relitigated later, and that lets you backtrack to a real fork when a downstream choice invalidates an earlier one.

This skill makes you design that way: interview first to confirm you're solving the right problem, then expose the real decision points, enumerate genuine alternatives, record the chosen branch with its rationale, and preserve the rejected branches with the reasons they lost.

## How to apply this

Before mapping any decision, **interview the user.** Before branching, before enumerating anything — ask. A few sharp, targeted questions about intent, constraints, success criteria, and non-goals are the highest-leverage work in any design task. Designing the right problem is worth more than any number of well-reasoned branches on the wrong one.

When planning or designing an approach, run through this in sequence: interview → find decision points → enumerate alternatives → record choices and rationale → emit the tree. When called as `/sonu:design-tree`, apply it to `$ARGUMENTS` or the current active discussion. When in plan mode, write the tree into the plan file's `## Design Tree` section; otherwise print it in-chat.

---

## 1. Interview first — confirm the problem before solving it

**This is the highest-leverage step.** Ask before you branch. The goal is a *shared understanding* of what you're designing: intent, constraints, success criteria, and non-goals. A few sharp questions up front save context, tokens, and the headache of treeing the wrong thing.

How to interview well:

- **Target the unknowns, not the obvious.** Skip what's already clear from the request. Ask only what a wrong assumption would make expensive to unwind.
- **Keep it tight.** 2–4 questions, not an interrogation. If you find yourself wanting to ask more, the problem is underspecified enough to warrant a scoping pass before designing anything.
- **Pin down success criteria explicitly.** "How will we know it's working?" is often the most clarifying question — it reveals scope, correctness bars, and non-goals at once.
- **Ask about constraints early.** Constraints eliminate whole branches before you draw them, so surfacing them first makes the rest cheaper.

**Example:** before designing an auth system, don't tree OAuth vs. magic links vs. passwords — ask: Is this customer-facing or internal? What's the session-lifetime expectation? Do we own the identity store or integrate with an existing one? One answer might make three branches irrelevant.

## 2. Find the real decision points

Only fork where the design could genuinely go in ≥2 consequential ways. Don't tree trivia.

A real decision point changes something downstream — the architecture, the user experience, the data shape, the deployment model, a hard constraint. A non-decision is anything where one option is obviously right given the constraints already established, or where the choice is reversible with trivial cost.

**When to fork:**
- The choice meaningfully affects what gets built or how it can evolve.
- Two or more options are genuinely in play given what you know.
- The chosen path closes other paths off.

**When not to fork:**
- One option is forced by constraints already agreed on (skip it, state the constraint once instead).
- The choice is a detail that can be changed later without ripple effects.
- Treeing it would add overhead that buries the real decisions.

## 3. Enumerate genuine alternatives

At every real fork, name ≥2 honest options — **no strawmen.**

A strawman is an option invented only to be knocked down: *"Option A is to store everything in a single JSON blob (obviously wrong)…"* That's not a branch, it's theater. It wastes the reader's attention and hides the real trade-off.

What makes an alternative genuine:
- A real team or real constraint could have chosen it.
- It has at least one meaningful advantage over the others in some context.
- Its costs are real and worth stating, not invented to make another option look better.

Aim for 2–3 alternatives per node. More than 3 is usually a sign the decision has a hidden dimension that should be its own node.

## 4. Record the chosen branch + why

State the decisive reason, not a vague preference.

**What counts as a decisive reason:**
- A constraint that rules other options out ("can't use X — we don't control the database")
- A concrete trade-off that tips the balance given the priorities established ("latency matters more than storage cost here")
- An irreversibility that makes getting it right now worth the cost ("changing the wire format later requires migrating all clients")

**What doesn't count:**
- "It's simpler." (Simpler for whom, at what cost?)
- "The team prefers it." (State what the preference is actually grounded in.)
- "It's the standard approach." (Standard is a prior, not a reason — name why it applies here.)

If you can't state the decisive reason in one sentence, the decision is probably still open.

## 5. Keep the pruned branches

Every rejected branch stays in the tree with **the reason it lost.**

Two things this buys you:
- **Stops silent relitigation.** When someone asks "why aren't we using X?" later, the answer is already there.
- **Preserves the fork for backtracking.** If a downstream decision makes your original choice untenable, you return to a recorded branch rather than inventing a new path from scratch.

What goes in a rejected branch:
- The option name
- One sentence: why it was rejected, or what would have to change for it to be the right call

Don't elaborate. A rejected branch that consumes a paragraph signals the decision wasn't really made yet.

## 6. Backtrack deliberately

When a downstream node invalidates an earlier choice, **return to the recorded fork and take a different branch.** Don't patch forward.

Patching forward means accepting the original choice and compensating — layering workarounds on top of a decision that no longer fits. It produces complexity that's hard to explain because the root cause is hidden upstream. A recorded tree makes backtracking surgical: you know exactly which node changed, which branch you're now taking, and what downstream choices need to be revisited.

**How to backtrack:**
1. Identify the upstream node that's now wrong.
2. State what changed (the new constraint, the new finding, the failed assumption).
3. Mark the previous choice as rejected, with the reason it's no longer viable.
4. Take a different branch and continue from there.

## 7. Notation — how to write the tree

Use a compact nested-bullet format. Scannable at a glance, not prose.

```
[decision point: one-line statement of what's being decided]
  ✓ [chosen option] — [decisive reason in one line]
  ✗ [rejected option 1] — [why it lost in one line]
  ✗ [rejected option 2] — [why it lost in one line]
    [sub-decision: only if this branch requires its own fork]
      ✓ [chosen sub-option] — [reason]
      ✗ [rejected sub-option] — [why]
```

**Example:**
```
[how to handle user sessions]
  ✓ JWT with short expiry + refresh token rotation — stateless, scales horizontally, expiry is enforceable without a server round-trip
  ✗ server-side sessions (Redis) — requires shared session store; adds infra dependency not justified at current scale
  ✗ long-lived JWTs with no refresh — can't invalidate compromised tokens without server-side blocklist, defeating the benefit

  [where to store the refresh token on the client]
    ✓ httpOnly cookie — not accessible to JS, survives page refresh, CSRF-mitigated with SameSite
    ✗ localStorage — accessible to any JS on the page; XSS leaks it
```

## 8. Output — where the tree goes

**In plan mode:** write the tree as a `## Design Tree` section in the plan file — placed after the Context section and before the implementation steps, so the decision genealogy is part of the plan record.

**Manual call (`/sonu:design-tree`):** print the tree in-chat. If a plan file is active, offer to write it into the plan file.

**On partial information:** if the interview didn't surface enough to fill all branches, emit what's known and mark open nodes with `[?]` — that's more honest than filling them speculatively.

---

## Self-check before you call it done

- Did you interview the user before treeing anything? Did you confirm intent, constraints, success criteria, and at least one non-goal?
- Does every node represent a genuinely consequential fork — not a trivial or forced choice?
- Does every node have ≥2 honest alternatives, with no strawmen invented to be knocked down?
- Is the chosen branch's rationale a decisive reason (constraint, concrete trade-off, irreversibility), not a vague preference?
- Does every rejected branch have a one-line reason? Could someone read it and understand why that path was closed?
- Are rejected branches concise enough to read as decided, not still-open?
- Is the tree traceable end-to-end — can you follow the chosen branches from root to leaf without gaps?
- If you backtracked, did you return to a recorded fork rather than patching forward?
- Is the tree written into the plan file (plan mode) or printed in-chat (manual call)?
