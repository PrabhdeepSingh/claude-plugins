---
name: design-tree
description: >-
  Records design decisions as a tree: real alternatives, the deciding reason, and rejected branches kept. Use when choosing between architectures, libraries or data models, or writing an implementation plan. Not for trivial or forced changes. Unclear goal: [[intent-interview]]; standards that pre-decide forks: [[code-standards]].
argument-hint: "[topic or decision to tree]"
allowed-tools: Skill, Read, Write, Edit
---

# Design tree — decide by branching

The project's own instructions win: where its AGENTS.md, CLAUDE.md or established conventions say otherwise, follow them and skip that part of this. When the project has its own plan or brief format, record the decisions in that format and at its length; the tree notation below is the default, not a requirement.

A design has decision points where it could go more than one way. Recording what was chosen and what was rejected, with reasons, stops decisions being reopened silently and gives you a known fork to return to when a later choice invalidates an earlier one.

Invoked directly as `/sonu:design-tree`, apply this to `$ARGUMENTS`; if that token appears literally or is empty, apply it to the current design or plan. To design and implement together, run `/sonu:build`.

## 1. Confirm the problem

Ask the user only what the repository cannot answer and a wrong guess would make expensive: scope first, then anything touching security or personal data, then user-visible behaviour, then technical detail. Read the code before asking, and cite the `file:line` in the question. If the request already states intent, constraints and what done looks like, say what you understood and proceed. If the project says not to wait for the owner, or nobody can answer (a subagent, a headless run), state your assumptions and mark any node they leave open with `[?]`. When the goal itself is unclear, that is [[intent-interview]].

Write each answer down where it will survive, one line (`Q: … → A: …`) in the plan file or ticket, because a human's ruling cannot be regenerated from the repository.

## 2. Find the real decision points

Fork only where the choice changes what gets built or how it can evolve, two or more options are in play, and choosing one closes the others. Where constraints force one option, state the constraint once and do not tree it.

**House standards are constraints already decided.** Load the standards that govern the build before looking for forks: [[code-standards]] always; [[safe-migrations]] when the design moves schema or data; [[seo]] for public web pages and content; [[infra-standards]] for infrastructure, containers or CI; [[observability]] for a new service, endpoint or background job; [[blast-radius]] when it changes data another component consumes; [[security]] for auth, personal or payment data, uploads, outbound requests to user-influenced destinations, new dependencies, webhooks, queues, or model output reaching an executable sink; [[accessibility]], [[layout]] and [[ui-polish]] for user-facing interface, plus [[typography]], [[colors]] or [[ux-writing]] as the change touches text styling, colour or interface copy; [[performance]] when the goal is itself a measurement. Skip any already loaded, and skip the load when the decision is not about code.

A fork one of them settles (id type, field casing, error envelope, migration staging) is not a decision point: state the constraint and cite the skill. Where the codebase already holds a contrary convention, that convention is the settled answer.

## 3. Name real alternatives

Give two or three honest options per fork. An honest option is one a real team could have chosen, with at least one real advantage. Do not invent an option to knock it down.

## 4. Record the choice and the deciding reason

A deciding reason is a constraint that rules the others out, a concrete trade-off given the stated priorities, or an irreversibility that makes getting it right now worth the cost. "It's simpler" and "it's the standard approach" are not reasons until you say for whom and why here. If you cannot state the reason in one sentence, the decision is still open.

## 5. Keep the rejected branches

Each rejected option stays, with one sentence on why it lost or what would have to change for it to win.

## 6. Backtrack to the fork

When a later decision invalidates an earlier one, return to the recorded fork, mark the old choice rejected with what changed, and take another branch. Do not layer workarounds on a decision that no longer fits.

## 7. Notation

```
[decision point: one-line statement of what's being decided]
  ✓ [chosen option] — [deciding reason in one line]
  ✗ [rejected option] — [why it lost in one line]
    [sub-decision: only if this branch requires its own fork]
      ✓ [chosen sub-option] — [reason]
      ✗ [rejected sub-option] — [why]
```

An open node the owner must rule on is marked `[?]`. Work you can sense coming but cannot yet phrase as a sharp question is left out of the tree; do not invent nodes for it.

## 8. Where the tree goes, and what a plan carries

In plan mode, write the tree as a `## Design Tree` section of the plan file, after the context and before the implementation steps, with a `Constraints:` line naming the standards it was drawn under. Otherwise print it in chat. When the repo keeps decision records, record the outcome in that convention too.

A plan file may be executed later by a session without this conversation, so it carries what that session cannot recover:

- **Paths and sources.** Name the files to create or change and the existing functions to reuse. A claim about how the code behaves today carries the `path:line` it was read at. Counts are taken, not estimated, and name the search that produced them.
- **Settled conventions at the point of use.** "Follow code-standards" leaves the decision to the executor; write the choice.
- **A check per step.** Label it `Verify (automated):` with the command and expected output, or `Verify (observed):` when only a person or a real run can see the proof, naming what must be observed. The build must capture evidence for observed checks and not let a passing suite stand in.
- **A `Gates:` block.** List the repo's own check commands once, cheapest first (format or lint, type-check, tests, any validate script the repo ships), read from its scripts, CI workflow or instructions, each with what a pass proves. Write `none` where the repo has no gate at a level. `/sonu:build` runs this block.
- **No hidden decisions.** A step that needs a decision is an unfinished fork: resolve it in the tree or mark it `[?]`. A periodic guarantee the plan relies on ("purge at 90 days") names the component that owns the schedule.
- **Criteria against steps.** Number the acceptance criteria and note which steps satisfy each. A criterion no step covers is a hole; a step that covers none is scope creep or a follow-up.
- **Vertical slices, riskiest first.** Each step delivers working, testable behaviour end to end, so a failed bet is found early. When steps share a contract, the step that defines it goes first.
- **Size.** A reviewer reads a very large change less carefully per line. When the plan would produce one, well past a few hundred changed lines or ten files, say so at the approval gate and propose a split along the plan's slices; the owner may waive it for a mechanical change.

Before the plan goes for approval, cut anything that changes neither what gets built nor what the owner must decide, and any flexibility no stated requirement needs. If implementation later invalidates a chosen branch, the executor returns to a recorded alternative; say that in the plan in one sentence.
