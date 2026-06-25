---
name: pr-conventions
description: Author PR descriptions from the right per-change-type template (using the repo's own PULL_REQUEST_TEMPLATE when present), detect and embed links to the issue tracker (GitHub Issues, JIRA, Linear, Shortcut, etc.), keep the description current as fixes land, and reply to reviewer threads from humans and AI bots. INVOKE when opening or updating a PR, choosing a PR description format, linking a ticket to a PR, or responding to reviewer comments — including outside of /sonu:ship. Called by /sonu:ship Phase 1 (body authoring), after Phase 4 (description refresh), and Phase 5 (thread replies). (Pairs with the [[self-review]] skill, which seeds the risk section.)
---

# PR conventions — right template, living description, honest replies

A PR body is the reviewer's briefing: it should match the change type so reviewers apply the right lens, stay accurate as fixes land, and every thread should get a reply that closes the loop. This skill handles all three.

## When to apply this

- **Opening a PR** (`/sonu:ship` Phase 1) — scan for a team template, classify the change type, fill the matching built-in template, embed the `RISKS` list in the risk section (Section A → B).
- **After fixes land** (after `/sonu:ship` Phase 4 push, and within each Phase 6 re-review cycle) — re-render the description in place (Section C).
- **Responding to reviewer comments** (`/sonu:ship` Phase 5, or standalone) — reply to every open inline thread with the right wording; resolve bot threads but leave human threads for the reviewer to resolve (Section D).

---

## A — Discover the team template first

Before reaching for a built-in template, scan for the repo's own PR template. The team standard wins.

```bash
# Single-file locations take precedence over multi-template directories.
# Check files first; only look for a directory if no file is found.
TEMPLATE_FOUND=""
for f in \
  ".github/PULL_REQUEST_TEMPLATE.md" ".github/PULL_REQUEST_TEMPLATE.txt" \
  ".github/pull_request_template.md" ".github/pull_request_template.txt" \
  "PULL_REQUEST_TEMPLATE.md" "PULL_REQUEST_TEMPLATE.txt" \
  "docs/PULL_REQUEST_TEMPLATE.md" "docs/PULL_REQUEST_TEMPLATE.txt"; do
  [ -f "$f" ] && { TEMPLATE_FOUND="SINGLE:$f"; break; }
done
if [ -z "$TEMPLATE_FOUND" ]; then
  for d in ".github/PULL_REQUEST_TEMPLATE" ".github/pull_request_template" \
            "PULL_REQUEST_TEMPLATE" "docs/PULL_REQUEST_TEMPLATE"; do
    [ -d "$d" ] && { TEMPLATE_FOUND="MULTI:$d"; break; }
  done
fi
echo "$TEMPLATE_FOUND"
```

| Found | Action |
|-------|--------|
| Single file | Use it verbatim as the base. Fill its sections from the change context. If it has no risk or reviewer-attention section, append the `## Risk / reviewer attention` block at the end. If it has no issue-link placeholder, embed the detected issue link as the last Summary bullet (see Section B — Detect the linked issue). Preserve checklists exactly. |
| Multi-template dir | Classify the change type (Section B). Pick the file whose name best matches (`feature.md`, `bugfix.md`, …). If none match, use the most generic file in the dir. Same fill + risk-append + issue-link rule. |
| Nothing found | Fall through to the per-type built-in template in Section B. |

---

## B — Classify the change type + built-in templates

**Detect in this order (first signal wins):**

1. **Branch name prefix** — `feat/` or `feature/` → feature; `fix/` or `bugfix/` → bugfix; `hotfix/` → hotfix; `chore/` → chore; `refactor/` → refactor; `docs/` or `doc/` → docs; `perf/` → perf; `release/` → release.
2. **Conventional-commit prefix** on commits ahead of the base (`git log origin/$BASE..HEAD --oneline`): `feat:` → feature; `fix:` → bugfix; `chore:` → chore; `refactor:` → refactor; `docs:` → docs; `perf:` → perf.
3. **Diff heuristic** — new files with new tests → feature; test-file-only changes → chore; only `.md` / docs-folder files → docs; only dependency or config bumps → chore.
4. **Default** — feature.

### Detect the linked issue first

Before filling the template, look for an issue reference. It belongs in the PR body so reviewers (and the issue tracker) can link back to the original context.

**Search order (first hit wins):**

```bash
BRANCH=$(git branch --show-current)
# $BASE is set by /sonu:ship Phase 0. When invoked standalone, derive it:
BASE=${BASE:-$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|.*/||')}
BASE=${BASE:-main}
LOG=$(git log "origin/$BASE..HEAD" --oneline 2>/dev/null)

# Pattern 1 — GitHub issue number: #123 in branch or commits
GH_ISSUE=$(echo "$BRANCH $LOG" | grep -oE '#[0-9]+' | head -1)

# Pattern 2 — tracker key (JIRA, Linear, etc.): ABC-123 in branch or commits
# Note: grep -E does not support \b word boundaries (treats \b as backspace).
# The uppercase-only pattern is distinctive enough without them in this context.
TRACKER_KEY=$(echo "$BRANCH $LOG" | grep -oE '[A-Z]{2,10}-[0-9]+' | head -1)

# Pattern 3 — Shortcut: sc-123 or ch123 in branch or commits
SC_KEY=$(echo "$BRANCH $LOG" | grep -oiE '(sc-[0-9]+|ch[0-9]+)' | head -1)
```

**Format the link based on what you found (check in order; first hit wins):**

| Found | Formatted link |
|-------|----------------|
| GitHub issue `#123` (`$GH_ISSUE`) | `Closes #123` — GitHub closes the issue automatically on merge |
| JIRA/Linear key `ABC-123` (`$TRACKER_KEY`) | JIRA: `[ABC-123](https://<org>.atlassian.net/browse/ABC-123)` — fill org from `JIRA_URL` env var or git remote domain if it contains `atlassian.net`; Linear: `[LIN-123](https://linear.app/<workspace>/issue/LIN-123)` — fill workspace from `LINEAR_WORKSPACE` env var |
| Shortcut `sc-123` / `ch123` (`$SC_KEY`) | `[sc-123](https://app.shortcut.com/<workspace>/story/123)` |
| Key found, org/workspace unknown | Use `Fixes ABC-123` as plain text — better than a broken link |
| Nothing found | Omit the issue line entirely — don't invent a reference |

**Where to embed it:** add the formatted issue link as the **last bullet of `## Summary`** in whatever template is being filled (built-in or team), e.g. `- Closes #123` or `- Fixes [ABC-123](url)`. If the team template already has an issue-link placeholder, fill that instead.

---

Fill the matching template below. Drop any section that genuinely doesn't apply. Never leave a `<placeholder>` unfilled.

### Feature / feat

```
## Summary
- <what this adds and why>

## Motivation
- <the problem or user need this solves>

## Changes
- <key implementation decision or component added>

## Risk / reviewer attention
<RISKS list from self-review — 3–5 items>

## Test plan
- <happy-path verification>
- <edge case to verify>

## Screenshots
<!-- Remove this section if no UI changed -->
```

### Bugfix / fix

```
## Summary
- <what broke and what this fixes>

## Root cause
<one or two sentences on why it broke>

## Fix
- <what changed to address the root cause>

## Risk / reviewer attention
<RISKS list from self-review>

## Regression test / test plan
- <failing test added (or why there isn't one)>
- <manual steps to verify the fix>
```

### Hotfix

```
## Summary (URGENT)
- <what is broken in production and what this fixes>

## Impact
- Affected: <scope — users, feature, service>
- Severity: <P0 / P1 / …>

## Root cause
<brief — expand in the incident doc>

## Fix
- <surgical change made>

## Rollback plan
- <how to revert if this makes things worse>

## Verification
- <minimal steps to confirm before merging>

## Risk / reviewer attention
<RISKS list from self-review>
```

### Chore

```
## Summary
- <what maintenance task this performs>

## Changes
- <package / tool / config updated — version if relevant>

## Risk / reviewer attention
<RISKS list — often low; say so explicitly: "Low risk — no logic changed">

## Test plan
N/A — no logic changed  <!-- or: suite ran, all green -->
```

### Refactor

```
## Summary
- <what was restructured; behavior is preserved>

## Changes
- <module / file restructured and how>

## Behavior preserved
- <how verified: existing suite green / added tests for X>

## Risk / reviewer attention
<RISKS list — call out blast radius if a shared utility changed>

## Test plan
- <existing suite green; new tests if coverage expanded>
```

### Docs

```
## Summary
- <what docs were added, updated, or removed>

## What changed
- <file or section and what is different>

No test plan — docs only.
```

### Perf

```
## Summary
- <what was optimized and why it matters>

## Benchmark
| Metric | Before | After |
|--------|--------|-------|
| <metric> | <value> | <value> |

## Changes
- <algorithm / query / cache strategy changed>

## Risk / reviewer attention
<RISKS list>

## Test plan
- <how correctness is verified after the optimization>
```

### Release

```
## Summary
- Releasing v<version>

## Highlights
- <key feature or change>

## Breaking changes
<!-- Remove if none -->
- <change and migration path>

## Migration notes
<!-- Remove if none -->
<steps for callers>

## Verification / rollback
- <smoke test or deploy check>
- Rollback: <revert tag / feature flag / step>

## Risk / reviewer attention
<RISKS list>
```

---

## C — Keep the description current

Re-render the PR body in place after each fix pass and each re-review round. **Never duplicate sections.**

1. Fetch the current body: `gh pr view $PR --json body -q .body`.
2. Update **Summary / Changes / Fix** bullets to reflect what the latest commits actually changed — not what was planned, what landed.
3. If the fix surface touched non-trivial logic: run `Skill(sonu:self-review)` on the latest diff and replace the **Risk / reviewer attention** section with the new output.
4. Preserve everything else verbatim — team-template checklists, breaking-change notes, screenshots, migration sections.
5. **Dedup guard**: if `## Risk / reviewer attention` already exists in the body, replace it in-place. Never append a second copy.
6. Write back: `gh pr edit $PR --body "$UPDATED_BODY"`.

---

## D — Reply to reviewer comments

Every open inline thread deserves a reply that closes the loop. Pick the wording from the table; tone and resolve policy differ by commenter type.

### Reply wording

| Scenario | Wording |
|----------|---------|
| **Fixed** | `Fixed in <SHA> — <one line on what changed>.` |
| **Justified / keeping as-is** | `Keeping as-is — <reason referencing the convention, constraint, or trade-off>.` |
| **False positive** | `I think this is a false positive — <why>; left a \`// TODO(review): <note>\` in the code marking it safe.` |
| **Partially addressed** | `Addressed <X> in <SHA>; deferring <Y> because <reason>.` |
| **Human question / need-info** | Answer directly. Offer the alternative if relevant, or ask the clarifying question back. |

No AI attribution in any reply. Bot replies: one or two lines. Human replies: slightly more explanatory on the justification "why."

### Post the reply

```bash
gh api -X POST "/repos/$REPO/pulls/$PR/comments/$COMMENT_ID/replies" \
  -f body="<reply text from table above>"
```

### Tone + resolution policy

**Bot threads** (any login matching the ship registry — `copilot`, `coderabbit`, `aikido`, `qodo`, `greptile`, `ellipsis`, `sourcery`, `cubic`, `korbit`):
- Terse. Reply, then **resolve the thread** via the GraphQL `resolveReviewThread` mutation (the exact call is in `/sonu:ship` Phase 5).

**Human threads** (all other logins):
- Warmer; explain the "why" more fully in justified replies.
- **Reply only.** Never call `resolveReviewThread` on a human's thread — leave resolution to them. Closing someone else's feedback thread is presumptuous and may conflict with their team's review workflow.

### Self-check before moving on

- Did every open bot thread get a reply **and** a resolve?
- Did every human thread get a reply but remain unresolved?
- Is any reply dismissive ("no issue here")? Rewrite it with an actual reason.
- No AI attribution anywhere?
