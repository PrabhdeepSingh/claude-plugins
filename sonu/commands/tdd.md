---
description: Drive a feature, bug fix, or behavior change test-first using the red-green-refactor loop.
argument-hint: "[feature, bug, or behavior to drive with tests]"
allowed-tools: Skill, Read, Write, Edit
---

# /tdd — drive this change test-first

Invoke the `tdd` skill against `$ARGUMENTS`.

If `$ARGUMENTS` names a feature, bug, or behavior, start the red-green-refactor loop on it: write the failing test first, then write the minimum code to make it pass, then refactor under the green test. If empty, apply the methodology to the current change in context.
