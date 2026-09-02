# Reviewer tuning — review once, report Important-only

Section F's two knobs, per reviewer in `/sonu:ship`'s registry. Each entry names the setting and where it lives; nothing here changes how ship runs — it changes how much the reviewers give ship to do.

**How this interacts with ship.** After a fix push, ship's Phase 6 re-requests Copilot explicitly (`gh pr edit --add-reviewer "@copilot"`) and drops the re-review mention for **every** bot that participated in Phase 2 — the mentions it knows are listed there (CodeRabbit, Sourcery, Greptile, Ellipsis, Cubic, Qodo). For those reviewers, turning off review-on-every-push costs no coverage inside a ship run: the one re-review ship wants, it asks for; what the setting removes is the *unasked* re-review on every fix commit. Three reviewers are outside that guarantee, and the entries below say so: ship posts no `@claude review`, and Aikido and Korbit have no mention at all — keep those three on their default trigger.

## Claude Code Review (hosted)

- **Review behavior → "Once after PR creation"** (runs when the PR opens or leaves draft). Not **Manual**: ship never posts `@claude review`, so a manual-only reviewer would see the first push and nothing after it. "After every push" is the most expensive mode and the one that produces re-rolls.
- **`REVIEW.md` at the repo root** — the reviewer reads it. Four clauses do the work:
  - After the first review, suppress new nits and post Important findings only.
  - Report at most five nits; mention the rest as a count.
  - Every behavior claim cites `file:line`.
  - Skip paths CI already enforces (formatting, lint, lockfiles, generated code).
- Findings are severity-tagged and never block the merge; gate on Important, ignore nits.

## claude-code-action (self-hosted review workflow)

- Trigger on `pull_request: [opened, ready_for_review, reopened]` — **drop `synchronize`** to get once-per-PR behavior.
- The bundled `code-review` plugin posts only findings scoring ≥ 80 confidence and skips drafts, trivial PRs, and PRs it already commented on — leave that threshold alone.
- Keep `allowed_bots` empty so a bot's comment never triggers another review (bot-on-bot loops).

## GitHub Copilot code review

- In the repository ruleset that enables automatic review, leave **"Review new pushes" off** (its default). Re-review then happens only on an explicit re-request — which ship does.
- Review depth **Lite** (default) over Balanced when cycle time matters more than recall.
- `.github/copilot-instructions.md` is read from the PR's head branch — put the "no nits on unchanged lines, cite `file:line`" expectation there; `REVIEW.md` and `CLAUDE.md` are also read.
- Copilot posts Comment-state reviews, never Request Changes; nothing it says blocks a merge.

## CodeRabbit

In `.coderabbit.yaml`:

```yaml
reviews:
  profile: chill                 # never "assertive" — that is the nitpick profile
  request_changes_workflow: false
  path_filters:
    - "!**/*.lock"
    - "!**/generated/**"
  auto_review:
    drafts: false
    auto_incremental_review: false   # review on open; @coderabbitai review for the one re-pass
```

`@coderabbitai resolve` closes every open CodeRabbit thread at once when a batch of fixes lands.

## Everything else in the registry (Aikido, Qodo, Greptile, Ellipsis, Sourcery, Cubic, Korbit)

Each has an equivalent pair of settings — an on-open-only or draft-skipping trigger and a severity or profile floor. Find them in the tool's repo config file or org dashboard and apply the same two rules **only where ship has a re-review mention for the tool** (`/review` for Qodo, `@greptileai`, `@ellipsis-dev`, `@sourcery-ai review`, `@cubic-dev-ai`). Aikido and Korbit have none — leave their review-on-push trigger on, or a fix commit goes unreviewed by them.

## Provenance

Last verified 2026-09. Setting names and defaults drift with vendor releases; re-verify against:

- Claude Code Review — `https://code.claude.com/docs/en/code-review`
- claude-code-action — `https://code.claude.com/docs/en/github-actions`
- Copilot code review configuration — `https://docs.github.com/en/copilot/how-tos/agents/copilot-code-review/configuring-automatic-code-review-by-copilot`
- CodeRabbit configuration reference — `https://docs.coderabbit.ai/reference/configuration`
