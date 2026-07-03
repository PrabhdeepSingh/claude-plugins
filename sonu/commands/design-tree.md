---
description: Build (or update) the design tree for the current problem — enumerate decision points, alternatives, the chosen branch with rationale, and the recorded rejected branches. Prints in-chat, or offers to fold into the active plan file. To design AND implement in one flow, use /sonu:build instead — it runs this as its design gate.
argument-hint: "[topic or decision to tree]"
allowed-tools: Skill, Read, Write, Edit
---

# /design-tree — map a decision as a branching tree

Invoke the `sonu:design-tree` skill against the user's request (`$ARGUMENTS` — the text typed after the command; if that token appears literally or is empty, use the current design, plan, or discussion in context instead).

If a specific design topic or decision is named, tree that. If nothing is named, apply the skill to the current design, plan, or discussion in context.

**Where the tree goes** (matches the skill's Section 8 — that section is the canonical rule):
- Print the tree in-chat.
- If a plan file is active (you are in plan mode and the plan file path is known), offer to write it into that file's `## Design Tree` section as well.

**When NOT to use this:** if you're about to implement the change end-to-end, run `/sonu:build` instead — it already runs design-tree as its Phase 1 design gate, then builds and self-reviews.

Start with the interview step — ask the user the 2–4 clarifying questions needed to establish shared understanding before mapping any decision point.
