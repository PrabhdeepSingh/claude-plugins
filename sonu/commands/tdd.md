---
description: Drive a feature, bug fix, or behavior change test-first using the red-green-refactor loop.
argument-hint: "[feature, bug, or behavior to drive with tests]"
allowed-tools: Skill, Read, Write, Edit
---

# /tdd — drive this change test-first

Invoke the `tdd` skill against `$ARGUMENTS`.

If `$ARGUMENTS` names a feature, bug, or behavior, start the red-green-refactor loop on it: write the failing test first, then write the minimum code to make it pass, then refactor under the green test. If empty, apply the methodology to the current change in context.

**What it produces:** writes test and implementation files directly to the working tree using the Edit/Write tools — not a printed plan. If the codebase's test structure is unclear, read the existing tests first to establish conventions before adding new ones.
