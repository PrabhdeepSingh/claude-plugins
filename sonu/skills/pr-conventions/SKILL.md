---
name: pr-conventions
description: >-
  Author PR descriptions from the right per-change-type template (the repo's own PULL_REQUEST_TEMPLATE wins), embed issue-tracker links, keep the description current as fixes land, and reply to human and bot review threads. INVOKE when opening or updating a PR or responding to reviewer comments — inside /sonu:ship or standalone. ([[self-review]] seeds the risk section.)
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
# GitHub supports the template in .github/, the repo root, or docs/, in either case spelling —
# on a case-sensitive filesystem (Linux, dev containers) both spellings must be checked explicitly.
TEMPLATE_FOUND=""
for f in \
  ".github/PULL_REQUEST_TEMPLATE.md" ".github/pull_request_template.md" \
  ".github/PULL_REQUEST_TEMPLATE.txt" ".github/pull_request_template.txt" \
  ".github/PULL_REQUEST_TEMPLATE" ".github/pull_request_template" \
  "PULL_REQUEST_TEMPLATE.md" "pull_request_template.md" \
  "PULL_REQUEST_TEMPLATE.txt" "pull_request_template.txt" \
  "PULL_REQUEST_TEMPLATE" "pull_request_template" \
  "docs/PULL_REQUEST_TEMPLATE.md" "docs/pull_request_template.md" \
  "docs/PULL_REQUEST_TEMPLATE.txt" "docs/pull_request_template.txt" \
  "docs/PULL_REQUEST_TEMPLATE" "docs/pull_request_template"; do
  [ -f "$f" ] && { TEMPLATE_FOUND="SINGLE:$f"; break; }   # -f: extension-less names (GitHub allows them) only match files, so the dir scan below is unaffected
done
if [ -z "$TEMPLATE_FOUND" ]; then
  for d in ".github/PULL_REQUEST_TEMPLATE" ".github/pull_request_template" \
            "PULL_REQUEST_TEMPLATE" "pull_request_template" \
            "docs/PULL_REQUEST_TEMPLATE" "docs/pull_request_template"; do
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

# Strip "(#N)" suffixes first — in commit subjects those are PR references added by
# squash/merge (e.g. "add feature (#42)"), NOT issue numbers. Treating one as an issue
# and writing "Closes #42" would auto-close the wrong thing on merge.
LOG_CLEAN=$(echo "$LOG" | sed -E 's/\(#[0-9]+\)//g')

# Pattern 1 — GitHub issue, high confidence: a closing keyword before the number
# ("fixes #123", "closes #123", "resolves #123") in the commits, or issue-N in the branch name.
GH_ISSUE=$(echo "$LOG_CLEAN" | grep -oiE '(fix(e[sd])?|close[sd]?|resolve[sd]?) #[0-9]+' | grep -oE '#[0-9]+' | head -1)
[ -z "$GH_ISSUE" ] && GH_ISSUE=$(echo "$BRANCH" | grep -oE 'issue[-/]?[0-9]+' | grep -oE '[0-9]+' | sed 's/^/#/' | head -1)
# Low confidence fallback: a bare #N with no closing keyword — record it, but format as
# "Related to #N" (never "Closes") so a wrong guess can't auto-close anything.
GH_ISSUE_WEAK=$(echo "$BRANCH $LOG_CLEAN" | grep -oE '#[0-9]+' | head -1)

# Pattern 2 — Shortcut: sc-123 or ch123 in branch or commits. Checked BEFORE the generic
# tracker pattern because an uppercase SC-123 would otherwise be misrouted to JIRA formatting.
# \b prevents matching "ch2024" inside words like "march2024" (works in BSD and GNU grep -E).
SC_KEY=$(echo "$BRANCH $LOG_CLEAN" | grep -oiE '\b(sc-[0-9]+|ch[0-9]+)\b' | head -1)

# Pattern 3 — tracker key (JIRA, Linear, etc.): ABC-123 in branch or commits.
# Exclude common technical tokens that share the shape (UTF-8, ISO-8601, SHA-256, …).
TRACKER_KEY=$(echo "$BRANCH $LOG_CLEAN" | grep -oE '\b[A-Z]{2,10}-[0-9]+\b' \
  | grep -viE '^(utf|iso|sha|rfc|md|aes|rsa|ecdsa|http|tls|ipv)-' | head -1)
```

**Format the link based on what you found (check in order; first hit wins):**

| Found | Formatted link |
|-------|----------------|
| High-confidence GitHub issue (`$GH_ISSUE`) | `Closes #123` — GitHub closes the issue automatically on merge |
| Shortcut `sc-123` / `ch123` (`$SC_KEY`) | `[sc-123](https://app.shortcut.com/<workspace>/story/123)` — fill workspace from `SHORTCUT_WORKSPACE` env var if set |
| JIRA/Linear key `ABC-123` (`$TRACKER_KEY`) | JIRA: `[ABC-123](https://<org>.atlassian.net/browse/ABC-123)` — fill org from `JIRA_URL` env var if set; Linear: `[LIN-123](https://linear.app/<workspace>/issue/LIN-123)` — fill workspace from `LINEAR_WORKSPACE` env var if set |
| Key found, org/workspace unknown | Use `Fixes ABC-123` as plain text — better than a broken link |
| Weak GitHub reference only (`$GH_ISSUE_WEAK`, no closing keyword, no tracker key) | `Related to #123` — never `Closes` on a guess; a bare `#N` may be a PR reference, and auto-closing the wrong item is worse than a soft link. Checked **last**: a real tracker key always outranks a keywordless `#N` guess |
| Nothing found | Omit the issue line entirely — don't invent a reference |

**Where to embed it:** add the formatted issue link as the **last bullet of `## Summary`** in whatever template is being filled (built-in or team), e.g. `- Closes #123` or `- Fixes [ABC-123](url)`. If the team template already has an issue-link placeholder, fill that instead.

---

Fill the matching template from `references/templates.md`. Drop any section that genuinely doesn't apply. Never leave a `<placeholder>` unfilled.

→ `references/templates.md` — the 8 per-change-type templates (feature, bugfix, hotfix, chore, refactor, docs, perf, release), read once the change type is classified and you're actually filling a body.

---

## C — Keep the description current

Re-render the PR body in place after each fix pass and each re-review round. **Never duplicate sections.**

When invoked standalone, derive the PR number first — `PR=$(gh pr view --json number -q .number)` (inside `/sonu:ship`, use the ledger's recorded number). Every fence below is a fresh shell: redeclare `PR` in each, never assume it survived.

1. Fetch the current body: `gh pr view $PR --json body -q .body`.
2. Update **Summary / Changes / Fix** bullets to reflect what the latest commits actually changed — not what was planned, what landed.
3. If the fix surface touched non-trivial logic: run `Skill(sonu:self-review)` on the latest diff and replace the **Risk / reviewer attention** section with the new output.
4. Preserve everything else verbatim — team-template checklists, breaking-change notes, screenshots, migration sections.
5. **Dedup guard**: if `## Risk / reviewer attention` already exists in the body, replace it in-place. Never append a second copy.
6. Write back — **capture the composed text into a variable explicitly, in the same fence that writes it**; passing an unset variable to `--body` silently blanks the live PR description:
   ```bash
   PR=$(gh pr view --json number -q .number)
   # Compose this at column 0 — a heredoc terminator (PREOF) must start the line, unindented.
   BODY=$(cat <<'PREOF'
   <the updated body text from steps 2–5 — replace this entire block>
   PREOF
   )
   gh pr edit "$PR" --body "$BODY"
   ```

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
| **Batched finding (one thread naming N locations)** | One reply for the whole thread, enumerating each location: `Fixed in <SHA> — line 87: <what>, line 162: <what>, line 203: <what>.` Never one reply per location, and never a reply covering fewer locations than the finding names — the resolve applies to the thread, so the reply must account for all of it. |
| **Human question / need-info** | Answer directly. Offer the alternative if relevant, or ask the clarifying question back. |

No AI attribution in any reply. Bot replies: one or two lines. Human replies: slightly more explanatory on the justification "why."

### Post the reply

When invoked standalone (outside `/sonu:ship`, which already knows these values), derive the context and enumerate the open threads first:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
PR=$(gh pr view --json number -q .number)   # the PR for the current branch
# List inline review comments with their ids — $COMMENT_ID below comes from here:
gh api "/repos/$REPO/pulls/$PR/comments" --paginate \
  --jq '[.[] | {id: .id, login: .user.login, path: .path, line: .line, body: .body}]'
```

Then reply to a specific comment (fresh shell — the leading declarations are required, per the shell discipline `/sonu:ship` states):

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
PR=$(gh pr view --json number -q .number)
COMMENT_ID=<id from the listing above — substitute the literal number>
gh api -X POST "/repos/$REPO/pulls/$PR/comments/$COMMENT_ID/replies" \
  -f body="<reply text from table above>"
```

### Classify severity before replying

Not every finding is mandatory, and treating them all as mandatory wastes the fix budget on nits while the structural item waits. Read each incoming finding for its stated severity — `Critical:`/blocking (security, data loss, broken behavior: fix before merge), unprefixed/required (fix before merge), `Nit:`/`Optional:`/`Consider:` (author's discretion — fix or justify), `FYI` (no action owed). A reviewer who marked something optional gets a decision, not an apology; a reviewer who found something structural gets it fixed first, whatever order the comments arrived in.

### Tone + resolution policy

**Bot threads** (inside `/sonu:ship`, any login matching that command's Phase 2 AI-reviewer registry — the canonical home; standalone, where that registry isn't loaded, classify mechanically: a commenter whose API `user.type` is `"Bot"`, or the literal login `Copilot`, is a bot):
- Terse. Reply, then **resolve the thread**. Inside `/sonu:ship`, Phase 5 carries the exact calls. Standalone: fetch thread ids with a GraphQL `reviewThreads` query on the PR (each thread's first comment `databaseId` matches the comment id you replied to), then resolve each with the `resolveReviewThread` mutation — `gh api graphql -f query='mutation($id:ID!){ resolveReviewThread(input:{threadId:$id}){ thread{ isResolved } } }' -f id=<thread id>`.

**Human threads** (all other logins):
- Warmer; explain the "why" more fully in justified replies.
- **Reply only.** Never call `resolveReviewThread` on a human's thread — leave resolution to them. Closing someone else's feedback thread is presumptuous and may conflict with their team's review workflow.

### Self-check before moving on

- Did every open bot thread get a reply **and** a resolve?
- Did every human thread get a reply but remain unresolved?
- Is any reply dismissive ("no issue here")? Rewrite it with an actual reason.
- No AI attribution anywhere?

---

## E — Versions and changelogs

Commits are how *you* track change; a **version** is how your *consumers* track it — the moment anything depends on the code, "latest on main" stops answering "what am I running, and is it safe to upgrade?". Three rules when a change ships to consumers:

- **When unsure whether a change is breaking, assume it is.** A "patch" that changes behavior consumers relied on is a major change wearing a disguise — a surprise major is far cheaper than a broken consumer.
- **A changelog is not `git log`.** It's the curated, consumer-facing answer to "what changed and do I care?" — grouped Added / Changed / Fixed / Deprecated / Removed / Security, phrased by user impact. Write the entry **in the same change that makes the change**, while the impact is fresh; reconstructed at release time, half of it is missing.
- **Derive the version from the tag** where the ecosystem allows, so the artifact, the tag, and the changelog can never disagree — every hand-edited version file is a place for them to drift apart.

## Provenance and maintenance

Volatile facts in this file, last verified 2026-07:

- **PR template locations** (Section A) — re-verify against GitHub's "Creating a pull request template" docs if template discovery ever misses a team's template.
- **Tracker URL formats** (Section B: `atlassian.net/browse/`, `linear.app/<workspace>/issue/`, `app.shortcut.com/<workspace>/story/`) — re-verify by opening a known ticket in each tracker.
- **AI-reviewer registry** — not stored here; the canonical home is `/sonu:ship` Phase 2.

## Reference files

| File | What it answers |
|------|-----------------|
| `references/templates.md` | The 8 per-change-type PR body templates (feature, bugfix, hotfix, chore, refactor, docs, perf, release) |
