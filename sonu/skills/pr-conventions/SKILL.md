---
name: pr-conventions
description: >-
  Pull-request rules: use the repo's own PR template first, link the issue, keep the description current as fixes land, and reply to human and bot review threads. Use when opening or updating a PR or answering review comments. Not for projects without pull requests. Risk section: [[self-review]].
---

# PR conventions

The project's own instructions win: where its AGENTS.md, CLAUDE.md or established conventions say otherwise, follow them and skip that part of this. A project that does not use pull requests skips this skill.

A PR body is the reviewer's briefing. It matches the change type, stays accurate as fixes land, and every review thread gets a reply. `/sonu:ship` uses section A and B when opening a PR, section C after each fix push, and section D when answering comments; each also works standalone.

---

## A — Use the team's template first

The repo's own PR template wins over the built-in ones. Single-file locations take precedence over template directories, and both case spellings are checked because some filesystems are case-sensitive.

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
| Single file | Use it as the base and fill its sections. If it has no risk section, append `## Risk / reviewer attention` at the end. If it has no issue-link placeholder, add the link as the last Summary bullet. Keep its checklists exactly. |
| Template directory | Classify the change type (section B) and pick the file whose name matches best, or the most generic one. Then fill it the same way. |
| Nothing | Use the built-in template for the change type (section B). |

---

## B — Change type, linked issue, built-in templates

Classify the change type from the first signal that applies: the branch prefix (`feat/`, `fix/`, `hotfix/`, `chore/`, `refactor/`, `docs/`, `perf/`, `release/`), then the commit prefixes ahead of the base, then the diff (new files with new tests is a feature; only docs is docs; only dependency or config bumps is a chore). The default is feature.

### Find the linked issue

The issue reference belongs in the PR body so the tracker links back.

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

Format the link by the first row that applies:

| Found | Formatted link |
|-------|----------------|
| High-confidence GitHub issue (`$GH_ISSUE`) | `Closes #123` — GitHub closes the issue automatically on merge |
| Shortcut `sc-123` / `ch123` (`$SC_KEY`) | `[sc-123](https://app.shortcut.com/<workspace>/story/123)` — fill workspace from `SHORTCUT_WORKSPACE` env var if set |
| JIRA/Linear key `ABC-123` (`$TRACKER_KEY`) | JIRA: `[ABC-123](https://<org>.atlassian.net/browse/ABC-123)` — fill org from `JIRA_URL` env var if set; Linear: `[LIN-123](https://linear.app/<workspace>/issue/LIN-123)` — fill workspace from `LINEAR_WORKSPACE` env var if set |
| Key found, org/workspace unknown | Use `Fixes ABC-123` as plain text — better than a broken link |
| Weak GitHub reference only (`$GH_ISSUE_WEAK`, no closing keyword, no tracker key) | `Related to #123` — never `Closes` on a guess; a bare `#N` may be a PR reference, and auto-closing the wrong item is worse than a soft link. Checked **last**: a real tracker key always outranks a keywordless `#N` guess |
| Nothing found | Omit the issue line entirely — don't invent a reference |

Add the link as the last bullet of `## Summary`, or fill the team template's own placeholder.

Then fill the matching template from `references/templates.md`. Drop a section that does not apply and leave no `<placeholder>` unfilled.

---

## C — Keep the description current

Re-render the PR body in place after each fix pass and each re-review round. Do not duplicate sections.

1. Fetch the current body: `gh pr view $PR --json body -q .body`. Standalone, derive `PR` with `gh pr view --json number -q .number`.
2. Update the Summary, Changes and Fix bullets to what the latest commits changed.
3. If the fixes touched non-trivial logic, run `Skill(sonu:self-review)` on the latest diff and replace the `## Risk / reviewer attention` section in place.
4. Keep everything else as it was: team-template checklists, breaking-change notes, screenshots.
5. Write it back. Capture the text into a variable in the same fence that writes it, because passing an unset variable to `--body` blanks the live description:

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

## D — Reply to review comments

Every open inline thread gets a reply.

| Scenario | Wording |
|----------|---------|
| Fixed | `Fixed in <SHA> — <one line on what changed>.` |
| Keeping as-is | `Keeping as-is — <reason: the convention, constraint or trade-off>.` |
| False positive | `I think this is a false positive — <why>; left a \`// TODO(review): <note>\` in the code marking it safe.` |
| Partially addressed | `Addressed <X> in <SHA>; deferring <Y> because <reason>.` |
| One thread naming several locations | One reply listing each location: `Fixed in <SHA> — line 87: <what>, line 162: <what>.` The resolve applies to the whole thread, so the reply accounts for all of it. |
| Re-raised or duplicate finding (already in the ship ledger's `justified:`, or raised by a second bot) | `Same as <the first thread's URL> — <the recorded reply, verbatim>.` Do not argue it again. |
| Human question | Answer directly, or ask the clarifying question back. |

No AI attribution in any reply. Replies to bots are one or two lines, and one sentence when justifying a Minor finding. Replies to humans explain the reason a little more fully.

### Post the reply

Standalone, derive the context and list the open comments first. Inside `/sonu:ship` these values are already known.

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
PR=$(gh pr view --json number -q .number)   # the PR for the current branch
# List inline review comments with their ids — $COMMENT_ID below comes from here:
gh api "/repos/$REPO/pulls/$PR/comments" --paginate \
  --jq '[.[] | {id: .id, login: .user.login, path: .path, line: .line, body: .body}]'
```

Then reply to a specific comment. Each fence is a fresh shell, so the declarations are repeated:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
PR=$(gh pr view --json number -q .number)
COMMENT_ID=<id from the listing above — substitute the literal number>
gh api -X POST "/repos/$REPO/pulls/$PR/comments/$COMMENT_ID/replies" \
  -f body="<reply text from table above>"
```

### Read the severity the reviewer stated

Reviewers that label findings do it on the comment's first line. CodeRabbit writes `_<category>_ | _<severity>_ | _<effort>_` with severity Critical, Major or Minor. Others use Critical/High/Medium/Low or a `[P1]`/`[P2]`/`[P3]` prefix: read High and `[P1]` as Critical, Medium and `[P2]` as Major, Low and `[P3]` as Minor. Copilot labels nothing. Carry the reviewer's own word into triage instead of inventing a second scale.

The label sets the default. Critical and Major are `FIX`. Minor is `FIX` when it names a concrete failing input and `JUSTIFY` otherwise. Anything naming security, data loss or a stated repo convention is `FIX` whatever its label. An unlabeled finding is triaged on `/sonu:ship` Phase 3's bullets alone. Do not look for `Nit:`, `Optional:` or `Consider:` prefixes; current bots do not use them.

### Bot threads and human threads

A commenter is a bot when its login matches `/sonu:ship`'s Phase 2 registry (its one home). Standalone, a commenter whose API `user.type` is `"Bot"`, or the literal login `Copilot`, is a bot.

- **Bot threads:** reply, then resolve the thread. Inside `/sonu:ship`, Phase 5 has the calls. Standalone, fetch thread ids with a GraphQL `reviewThreads` query on the PR (each thread's first comment `databaseId` matches the comment id you replied to), then resolve each: `gh api graphql -f query='mutation($id:ID!){ resolveReviewThread(input:{threadId:$id}){ thread{ isResolved } } }' -f id=<thread id>`.
- **Human threads:** reply only. Never resolve a human's thread; closing someone else's feedback is theirs to do and may conflict with their team's workflow.

---

## E — Versions and changelogs

When a change ships to consumers:

- If unsure whether a change is breaking, treat it as breaking. A surprise major version costs less than a broken consumer.
- A changelog is not `git log`. It is grouped Added, Changed, Fixed, Deprecated, Removed, Security, phrased by user impact, and written in the same change that makes the change.
- Derive the version from the tag where the ecosystem allows, so the artifact, the tag and the changelog cannot disagree.

## F — Reviewer settings

`/sonu:ship` controls its review loop from inside and depends on no repo-side reviewer configuration. `references/reviewer-tuning.md` is optional reading for a repo owner who wants to tune the reviewers themselves; read it only when asked to set up a repo's reviewers.

## Maintenance

Last checked 2026-09: the reviewer severity headers in section D (re-check by reading the first line of a few recent bot comments), the PR template locations in section A, and the tracker URL formats in section B.

## Reference files

| File | What it answers |
|------|-----------------|
| `references/templates.md` | The built-in PR body templates by change type |
| `references/reviewer-tuning.md` | Optional per-reviewer settings (section F) |
