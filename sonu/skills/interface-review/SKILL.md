---
name: interface-review
description: >-
  Cross-discipline review of a whole screen, flow, or feature — coordinates [[accessibility]], [[layout]], [[ux-writing]], [[typography]], [[colors]], and [[ui-polish]] into one ranked findings table and a single verdict. INVOKE for a holistic interface review or UI audit; quick and full modes. Orchestration only — it carries no domain rules. For one domain, or for build-time interface work, use the domain skills directly.
argument-hint: "[quick|full] [scope]"
allowed-tools: Skill, Read, Bash, Grep, Glob
---

# Review the interface as one system

A strong interface is not six independent audits stapled together. Review the whole experience, let each domain skill own its rules, then consolidate the evidence into one prioritized verdict.

This skill owns orchestration only. Accessibility rules belong to [[accessibility]]; structure to [[layout]]; copy to [[ux-writing]]; type to [[typography]]; color to [[colors]]; visual polish and motion to [[ui-polish]]. Never duplicate or override their rules here.

## Core Principles

### 1. Resolve Scope and Mode First

Apply this to `$ARGUMENTS` — the text typed after the invocation, which carries an optional mode (`quick` or `full`) and an optional scope. If that token appears literally or is empty, infer the screen, flow, feature, or repository scope from the request, the current discussion, and the workspace. State the resolved scope in the output. Use `full` when no mode is supplied.

| Mode | Coverage | Finding cap |
| --- | --- | --- |
| `quick` | Primary user path and highest-traffic states; report only `HIGH` and `MEDIUM` issues | 5 |
| `full` | Entire requested scope across all six domain skills, including empty, loading, error, and narrow-width states when present | 15 |

If the requested scope is too large to inspect credibly, narrow it to the highest-traffic complete flow and state the boundary. Never imply uninspected surfaces were reviewed.

### 2. Recon Before Judgment

Identify the framework, styling system, component library, design tokens, supported viewports, and available preview or test commands. Follow the project's established Tailwind, plain CSS, CSS-in-JS, token, and component conventions.

### 3. Use Domain Skills as the Sources of Truth

Before reviewing, confirm that all six owning skills below are available. Load and apply every available owner. In `quick` mode, inspect all six domains but spend depth only where the primary flow has evidence. In `full` mode, complete each available domain review before consolidation.

Review in this order so foundational failures are not hidden by polish:

1. `Skill(sonu:accessibility)`
2. `Skill(sonu:layout)`
3. `Skill(sonu:ux-writing)`
4. `Skill(sonu:typography)`
5. `Skill(sonu:colors)`
6. `Skill(sonu:ui-polish)`

This skill owns the final response. When a domain skill is loaded through this coordinator, apply its principles and references but ignore its standalone **Review Output Format**. Use the consolidated format, shared severity, and finding cap in this file instead.

If an owning skill is unavailable, mark that domain `Not reviewed`, name the missing skill, and continue with the remaining domains. Do not recreate its rules from memory, substitute a neighboring skill, or claim holistic coverage.

When two skills appear to cover the same issue, assign it to the skill that owns the underlying rule and mention secondary effects in the **Why** cell. Report it once.

### 4. Require Evidence

Every finding cites `path/to/file:line` and shows the current implementation. If the review artifact has no source files, cite the exact screen and component. Do not report a code-level finding from visual appearance alone or a visual finding from source code alone when runtime behavior determines the result.

### 5. Rank by User Impact

Use one shared severity scale:

- `HIGH`: blocks a task, misleads the user, hides content or controls, causes data-loss risk, or creates a repeated systemic failure.
- `MEDIUM`: meaningfully harms comprehension, efficiency, adaptability, or consistency.
- `LOW`: isolated polish with limited task impact. Include only in `full` mode.

Within a severity, rank by reach and leverage. A token or shared-component fix outranks the same symptom in one leaf component.

### 6. Consolidate Systemic Findings

One root cause is one finding. List every confirmed location in the same row rather than producing a row per occurrence. Do not pad the report to reach the finding cap; a short review or no findings is a valid result.

### 7. Make Restraint Visible

Record candidates considered but deliberately rejected. A candidate is rejected when the owning skill permits the current implementation, evidence is insufficient, the project convention is intentional, or the proposed change would add complexity without user benefit.

### 8. Verify What Can Be Verified

Run safe, relevant checks available in the project. Inspect the rendered interface when runtime behavior or visual judgment matters. Report the exact command or interaction and observed result. If a check cannot be run, label it **Not verified** and state what remains; never convert a verification gap into a finding.

### 9. Review Without Mutating by Default

Treat a review request as read-only. Do not edit source code unless the user also asks to implement the findings. When implementation is requested, preserve the consolidated report as the change scope and re-run the relevant verification afterward.

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Six disconnected domain reports | Consolidate into one ranked findings table |
| Same issue reported by multiple skills | Assign it to the skill that owns the underlying rule |
| Finding with no exact location | Cite `path/to/file:line` and the current implementation |
| Visual claim inferred only from source | Inspect the rendered state or mark it not verified |
| Unlimited low-impact polish | Respect the mode cap; omit `LOW` findings in `quick` |
| Silent gaps in coverage | Show which domains and states were actually inspected |
| Missing owning skill silently treated as covered | Mark the domain `Not reviewed` and name the unavailable skill |
| No rejected candidates | Include the required considered-but-rejected table |
| Review silently edits code | Stay read-only unless implementation was requested |
| “Approve” with pending actionable findings | Use `Needs changes` or `Block` |

## Review Output Format

Always use the following sections.

### Scope and Coverage

State the mode, exact scope, stack and styling conventions, and any review boundary. Then show coverage:

| Domain | Evidence inspected | Result |
| --- | --- | --- |
| Accessibility | Files, components, states, or checks | Findings count or `Clear` |

Include all six domains. `Clear` means inspected with no actionable finding; `Not reviewed` must explain why.

### Findings

Use one table ordered by severity, then reach and leverage:

| # | Severity | Domain | Location | Before | After | Why |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | HIGH | Accessibility | `src/Dialog.tsx:42` | `<button><XIcon /></button>` | Add `aria-label="Close"` and hide the icon from the accessibility tree | The icon-only control has no accessible name |

Each row is one root cause. The **Domain** value names the owning domain: `Accessibility`, `Layout`, `Writing`, `Typography`, `Colors`, or `UI`. Respect the mode's finding cap. If there are no findings, omit the table and state "No actionable interface findings."

### Considered but Rejected

Include 1–3 candidates in `quick` mode and 2–5 in `full` mode:

| Location | Candidate | Rejected because |
| --- | --- | --- |
| `src/Card.tsx:28` | Increase the shadow | Existing depth matches the shared surface token; changing one card would reduce consistency |

These are real candidates inspected during the review, not invented filler. If the scope genuinely contains fewer borderline candidates, include the ones that exist and say so.

### Verification

List each check or interaction, the exact command or steps, and the observed result. Separate checks that passed from checks marked **Not verified**.

### Verdict

End with exactly one:

- `Block` — one or more `HIGH` findings remain.
- `Needs changes` — only `MEDIUM` or `LOW` findings remain.
- `Approve` — no actionable findings remain and the claimed coverage was verified.

**One exception to this whole output format:** when this methodology is applied as a review *lens* rather than as a standalone review — [[self-review]]'s interface lens is the case that exists today — the lens's own reporting contract wins. Report findings in the lens's requested line format and omit the sections, severity tables, and verdict above; a verdict returned into a fan-out synthesis is unusable there.
