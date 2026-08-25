---
name: design-tree
description: >-
  Make design decisions as an explicit branching tree — genuine alternatives, decisive rationale, rejected branches preserved. INVOKE PROACTIVELY when planning or designing any implementation approach, or choosing between architectures, libraries, or data models — especially in plan mode. Skip trivial or forced changes with no real alternatives.
argument-hint: "[topic or decision to tree]"
allowed-tools: Skill, Read, Write, Edit
---

# Design Tree — decide by branching, not by marching

Design isn't a straight line from question to answer. It's a tree: each decision point is a node with multiple branches; you pick one, sometimes backtrack, and move forward. The genealogy of what you chose *and what you didn't* is the record that keeps decisions from getting silently relitigated later, and that lets you backtrack to a real fork when a downstream choice invalidates an earlier one.

This skill makes you design that way: interview first to confirm you're solving the right problem, then expose the real decision points, enumerate genuine alternatives, record the chosen branch with its rationale, and preserve the rejected branches with the reasons they lost.

## How to apply this

Before mapping any decision, **interview the user.** Before branching, before enumerating anything — ask. A few sharp, targeted questions about intent, constraints, success criteria, and non-goals are the highest-leverage work in any design task. Designing the right problem is worth more than any number of well-reasoned branches on the wrong one.

When planning or designing an approach, run through this in sequence: interview → load applicable standards → find decision points → enumerate alternatives → record choices and rationale → emit the tree. When invoked directly as `/sonu:design-tree`, apply it to `$ARGUMENTS` — the text typed after the invocation; if that token appears literally or is empty, apply it to the current design, plan, or discussion in context; and if the intent is to design AND implement end-to-end, run `/sonu:build` instead — it runs this methodology as its design gate, then builds test-first and self-reviews. When in plan mode, write the tree into the plan file's `## Design Tree` section; otherwise print it in-chat.

---

## 1. Interview first — confirm the problem before solving it

**This is the highest-leverage step.** Ask before you branch. The goal is a *shared understanding* of what you're designing: intent, constraints, success criteria, and non-goals. A few sharp questions up front save context, tokens, and the headache of treeing the wrong thing.

How to interview well:

- **Target the unknowns, not the obvious.** Skip what's already clear from the request. Ask only what a wrong assumption would make expensive to unwind.
- **Keep it tight.** 2–4 questions, not an interrogation. If you find yourself wanting to ask more, the problem is underspecified enough to warrant a scoping pass before designing anything.
- **Pin down success criteria explicitly.** "How will we know it's working?" is often the most clarifying question — it reveals scope, correctness bars, and non-goals at once.
- **Ask about constraints early.** Constraints eliminate whole branches before you draw them, so surfacing them first makes the rest cheaper.
- **Know when not to interview.** If the request already states intent, constraints, and success criteria, don't ask rote questions to satisfy this step — say what you understood and proceed. And if no user can answer (a non-interactive run, a subagent, a headless session), skip the interview entirely: state the assumptions you're proceeding on, and mark any node those assumptions leave genuinely open with `[?]` (Section 8) instead of stalling.

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

**House standards are pre-decided constraints.** Before hunting for decision points, load the standards skills that will govern the build — [[code-standards]] always; [[safe-migrations]] when the design moves schema or data; [[seo-standards]] (templates, routes, metadata) and/or [[content-seo]] (prose meant to rank — both when the change has both) when the surface is public web; [[infra-standards]] when it touches IaC, containers, or CI; [[observability]] when it adds a service, endpoint, or background job; [[blast-radius]] when it changes the shape, format, or semantics of data another component consumes (return values, response bodies, serialized payloads, DB columns read elsewhere, log/telemetry fields, events, config values/env vars, parsed CLI output); [[accessibility]], [[layout]], and [[ui-polish]] when the surface is user-facing interface (components, screens, styles, templates, interface copy), plus [[typography]], [[colors]], or [[ux-writing]] as the change touches text styling, color, or interface copy. Skip any of these already loaded this session (under `/sonu:build`, Phase 1 loads them before this skill runs), and skip the load entirely when the tree isn't about code — a product or process decision has no build to govern. A fork that one of them already settles (id type, field casing, error envelope, migration staging) is not a genuine decision point — state the constraint and cite the skill instead of treeing it. "Settles" means the standard's default *plus* its match-the-repo rule: on a brownfield surface where the codebase already holds a contrary convention, that established convention is itself the settled answer ([[code-standards]]' consistency-beats-correctness rule) — not a fork to reopen, and not a license to plan against the repo. This resolves the plan-vs-standards conflict at design time, where the strong model and the approval gate are, instead of leaving it for the executor to discover mid-build.

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

**In plan mode:** write the tree as a `## Design Tree` section in the plan file — placed after the Context section and before the implementation steps, so the decision genealogy is part of the plan record. Directly under the heading, add a one-line `Constraints:` note naming the standards skills the design was drawn under (e.g. `Constraints: code-standards, safe-migrations`), and one sentence addressed to the executor who picks the plan up later: if implementation reality invalidates a chosen branch, return to the recorded fork in the Design Tree above and take a recorded alternative — do not improvise a new path or deviate from the standards to save the plan. (Write that sentence self-contained: the executor has this plan file, not this skill, so it must not cite sections of this document.)

**Executor-ready plans.** A plan file outlives this session. Assume its executor is a smaller model in a fresh session with zero conversation context, following the text word-for-word — every judgment call the plan leaves open is a decision silently delegated to the model least equipped to make it. Before the plan goes for approval:

- **Name exact paths** — every file to create or modify, and every existing function or utility to reuse (with its path), so the executor never has to search, guess, or re-implement.
- **Settle every convention in place** — where a loaded standard decides something (field casing, id type, error envelope, migration staging), state the settled choice at the point of use. "Follow code-standards" alone defers the decision to the executor.
- **Give each step a verification check** — the command to run and what success looks like, so a step that landed wrong is caught immediately, not three steps later.
- **Hide no judgment calls** — if writing a step requires a decision, that's an unfinished fork: resolve it in the tree, or mark it `[?]` so the owner rules on it at the approval gate. A visible `[?]` is honest and allowed; what's forbidden is a decision buried inside an implementation step for the executor to make silently.

On an orchestrator-class session model, this bar is also the delegation bar — within [[model-tiering]]'s categorical exclusions: that skill grades the steps that clear it for delegated execution by subagents, but clearing this bar never delegates design, integration, debugging, or security work, no matter how well specified. The routing methodology, the exclusions, and which model tier each grade maps to all live there.

**Manual call (`/sonu:design-tree`):** print the tree in-chat. If a plan file is active, offer to write it into the plan file.

**On partial information:** if the interview didn't surface enough to fill all branches, emit what's known and mark open nodes with `[?]` — that's more honest than filling them speculatively.

**`[?]` vs not-yet-specifiable.** A `[?]` is a sharp question the owner can rule on at the approval gate. Work you can sense coming but cannot yet phrase that sharply is not a `[?]` and not a fork — leave it out of the tree entirely. The fog test that draws this line lives in [[ticket-lifecycle]] section 9; do not invent speculative nodes to stand in for fog.

---

## Self-check before you call it done

- Did you interview the user before treeing anything (or, if the request was already fully specified or no user could answer, did you state your assumptions explicitly)? Did you confirm intent, constraints, success criteria, and at least one non-goal?
- Did you load the applicable standards skills before branching, and does no node relitigate a choice a standard already settles?
- Does every node represent a genuinely consequential fork — not a trivial or forced choice?
- Does every node have ≥2 honest alternatives, with no strawmen invented to be knocked down?
- Is the chosen branch's rationale a decisive reason (constraint, concrete trade-off, irreversibility), not a vague preference?
- Does every rejected branch have a one-line reason? Could someone read it and understand why that path was closed?
- Are rejected branches concise enough to read as decided, not still-open?
- Is the tree traceable end-to-end — can you follow the chosen branches from root to leaf without gaps?
- If you backtracked, did you return to a recorded fork rather than patching forward?
- Is the tree written into the plan file (plan mode) or printed in-chat (manual call)?
- If a plan file was written: could a smaller model in a fresh session execute it without making a single *unmarked* design decision — exact paths, conventions settled in place, a verification check per step, every genuinely open fork visibly `[?]`, a `Constraints:` line naming the standards applied?
