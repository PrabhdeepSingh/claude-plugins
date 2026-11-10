---
name: interface-review
description: >-
  Review a whole screen, flow or feature across [[accessibility]], [[layout]], [[ux-writing]], [[typography]], [[colors]] and [[ui-polish]] and report what was found, most serious first. Use when asked for an interface review or UI audit. Not for one domain or for build-time work: use that skill directly.
argument-hint: "[scope]"
allowed-tools: Skill, Read, Bash, Grep, Glob
---

# Review the interface as one system

The project's own instructions win: where its AGENTS.md, CLAUDE.md, design system or established conventions say otherwise, follow them and skip that part of this. Use the project's existing kit and tokens; do not add components, on-screen text or options that the project's design does not call for.

Review the whole experience against the project's own design intent first: its design system, its instructions about how much belongs on a screen, and its conventions. The domain skills supply defaults for what the project leaves open. A finding that asks for more components, more text or more options than the project's design allows is not a finding.

Apply this to `$ARGUMENTS`, the scope to review. If that token appears literally or is empty, infer the scope from the request and the workspace, and state it. If the scope is too large to inspect credibly, narrow it to the most used complete flow and say where the boundary is.

A review is read-only. Change the interface only when the person also asks for that.

## How to review

Find the framework, styling system, design tokens and preview or test commands, so findings follow the project's conventions. Load the domain skills that bear on the scope: `Skill(sonu:accessibility)`, `Skill(sonu:layout)`, `Skill(sonu:ux-writing)`, `Skill(sonu:typography)`, `Skill(sonu:colors)`, `Skill(sonu:ui-polish)`. If one does not load, say that domain was not reviewed.

Look at the rendered interface where you can, including its loading, empty and error states and a narrow width. Unit tests do not test layout or rendering, so a claim about how something looks rests on having looked. Do not report a visual finding from source alone when runtime behavior decides the result; mark it as not verified.

## What to report

State the scope and what was and was not inspected. Then list the findings, most serious first: something that blocks a task, misleads, hides content or risks data loss comes before something that slows comprehension, which comes before isolated polish. For each finding give the location (`path/to/file:line`, or the screen and component), what is there now, what to change it to, and why. One root cause is one finding with all its locations; a fix in a token or shared part ranks above the same symptom in one place.

Report what you found and no more. A short review, or none, is a valid result. Say which checks you ran and which you could not. Give no verdict: the person who asked decides what to do with the findings.

When this is used as a lens inside another review, such as the interface checklist of [[self-review]], report in that review's own format.
