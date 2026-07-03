---
description: Surface the 3–5 riskiest parts of the current diff so a reviewer knows where to look hardest — a pointer, not a score or an approval. Runs the self-review skill on demand; /sonu:build and /sonu:ship already run it automatically, and for an actual bug-hunting review use /code-review instead.
argument-hint: "[optional: what to review — defaults to the current diff]"
allowed-tools: Skill, Bash, Read
---

# /self-review — where should a reviewer look hardest?

Invoke the `sonu:self-review` skill against the current change (`$ARGUMENTS` — the text typed after the command; if that token appears literally or is empty, review the working tree / current branch per the skill's diff-picker).

The skill handles everything: picking the right diff command for the current state (uncommitted, staged, just-committed, or whole-branch — including untracked files), identifying the 3–5 riskiest spots, and ending with the explicit non-approval line.

**When NOT to use this:** inside `/sonu:build` or `/sonu:ship` — both already run self-review automatically at the right moments (build's hand-back, ship's PR creation). And it is not a code review: it points attention, it does not find or fix bugs — for an actual review, use `/code-review` or open the PR to reviewers via `/sonu:ship`.
