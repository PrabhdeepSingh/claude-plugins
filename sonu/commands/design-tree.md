---
description: Build (or update) the design tree for the current problem — enumerate decision points, alternatives, the chosen branch with rationale, and the recorded rejected branches. Folds into the active plan file, or prints in-chat.
argument-hint: "[topic or decision to tree]"
allowed-tools: Skill, Read, Write, Edit
---

# /design-tree — map a decision as a branching tree

Invoke the `design-tree` skill against `$ARGUMENTS`.

If `$ARGUMENTS` is provided, it names the specific design topic or decision to tree. If empty, apply the skill to the current design, plan, or discussion in context.

**Where the tree goes:**
- If a plan file is active (you are in plan mode and the plan file path is known), write or update the `## Design Tree` section in that file.
- Otherwise, print the tree in-chat.

Start with the interview step — ask the user the 2–4 clarifying questions needed to establish shared understanding before mapping any decision point.
