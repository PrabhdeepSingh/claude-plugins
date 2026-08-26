---
name: interface-review
description: >-
  Cross-discipline review of a whole screen, flow, or feature — coordinates [[accessibility]], [[layout]], [[ux-writing]], [[typography]], [[colors]], and [[ui-polish]] into one ranked findings table and a single verdict. INVOKE for a holistic interface review or UI audit; quick and full modes. Orchestration only — it carries no domain rules. For one domain, or for build-time interface work, use the domain skills directly.
argument-hint: "[quick|full] [scope]"
allowed-tools: Skill, Read, Bash, Grep, Glob
---

# Review the interface as one system

A strong interface is not six independent audits stapled together. Review the whole experience, let each domain skill own its rules, then consolidate the evidence into one prioritized verdict. This skill owns orchestration only: accessibility rules belong to [[accessibility]], structure to [[layout]], copy to [[ux-writing]], type to [[typography]], color to [[colors]], polish and motion to [[ui-polish]]. Never duplicate or override their rules here.

## 1. Resolve Scope and Mode

Apply this to `$ARGUMENTS` — the text typed after the invocation, carrying an optional mode (`quick` or `full`) and an optional scope. If that token appears literally or is empty, infer the scope from the request and workspace, and use `full`. State the resolved scope in the output.

| Mode | Coverage | Finding cap |
| --- | --- | --- |
| `quick` | Primary user path and highest-traffic states; report only `HIGH` and `MEDIUM` | 5 |
| `full` | Entire requested scope across all six domains, including empty, loading, error, and narrow-width states | 15 |

If the scope is too large to inspect credibly, narrow it to the highest-traffic complete flow and state the boundary — never imply uninspected surfaces were reviewed.

## 2. Load the Domain Skills

Availability is discovered by attempting the load — there is no way to enumerate installed skills without trying. Invoke each in this order, so foundational failures are not hidden by polish: `Skill(sonu:accessibility)`, `Skill(sonu:layout)`, `Skill(sonu:ux-writing)`, `Skill(sonu:typography)`, `Skill(sonu:colors)`, `Skill(sonu:ui-polish)`. A load that fails marks that domain `Not reviewed` — name the missing skill, continue with the rest, and never recreate its rules from memory, substitute a neighbor, or claim holistic coverage.

This skill owns the final response: when a domain skill is loaded through this coordinator, apply its principles and references but ignore its standalone **Review Output Format** — the consolidated format, shared severity, and finding cap here take precedence. When two skills appear to cover the same issue, assign it to the skill that owns the underlying rule, mention secondary effects in the **Why** cell, and report it once.

## 3. Evidence and Severity

Every finding cites `path/to/file:line` and the current implementation (or the exact screen and component when there are no source files). Do not report a code-level finding from visual appearance alone, or a visual finding from source alone when runtime behavior determines the result — inspect the rendered state or mark it not verified. One shared scale:

- `HIGH`: blocks a task, misleads the user, hides content or controls, causes data-loss risk, or creates a repeated systemic failure.
- `MEDIUM`: meaningfully harms comprehension, efficiency, adaptability, or consistency.
- `LOW`: isolated polish with limited task impact — `full` mode only.

Within a severity, rank by reach and leverage: a token or shared-component fix outranks the same symptom in one leaf component. One root cause is one finding — list every confirmed location in the same row. Never pad toward the cap; a short review or no findings is a valid result.

## Review Output Format

Always use these sections.

**Scope and Coverage** — the mode, exact scope, stack conventions, and any boundary; then a coverage table (`| Domain | Evidence inspected | Result |`) listing all six domains, where `Clear` means inspected with no actionable finding and `Not reviewed` must explain why.

**Findings** — one table ordered by severity, then reach and leverage: `| # | Severity | Domain | Location | Before | After | Why |`. Each row is one root cause; **Domain** names the owner (`Accessibility`, `Layout`, `Writing`, `Typography`, `Colors`, `UI`). Respect the mode's cap. With no findings, omit the table and state "No actionable interface findings."

**Considered but Rejected** — up to 3 candidates in `quick`, 5 in `full`: `| Location | Candidate | Rejected because |`. A candidate is rejected when the owning skill permits the implementation, evidence is insufficient, the convention is intentional, or the change adds complexity without user benefit. These are real candidates inspected during the review — zero is a legal count ("none — nothing borderline was inspected"); never invent filler.

**Verification** — each check or interaction, the exact command or steps, and the observed result, separating passes from **Not verified**. Run safe, relevant checks; never convert a verification gap into a finding. A production page should also show a clean console — zero errors and warnings before shipping — and remember unit tests don't test CSS, layout, or real rendering: the DOM being correct is verified by looking, not by a green suite.

**Verdict** — exactly one: `Block` (one or more `HIGH` remain), `Needs changes` (only `MEDIUM`/`LOW` remain), `Approve` (no actionable findings and the claimed coverage was verified).

**One exception to this whole output format:** when this methodology is applied as a review *lens* rather than a standalone review — [[self-review]]'s interface lens is the case that exists today — the lens's own reporting contract wins: report in the lens's requested line format and omit the sections, tables, and verdict above; a verdict returned into a fan-out synthesis is unusable there.
