---
description: Branch, commit, open PR, gather Claude (code + security) + Copilot reviews, fix/justify/resolve every finding, loop until clean, then merge
allowed-tools: Bash, Read, Edit, Write, Skill
---

# /ship — PR Babysitter

Handles everything from the current working-tree state through a clean, merged PR. Run after implementation is complete and the owner has said "go". Do not stop until the PR is merged (or auto-merging), or a decision only the owner can make is reached.

**Autonomy contract — run start-to-finish without checking back.** Invoking `/ship` (or saying "ship it") IS the authorization for the entire flow through merge. Once started, flow through every phase — including the final merge — without pausing to report or to ask for a go-ahead. In particular:

- **Clean reviews are not a checkpoint.** If code review, security review, and Copilot all come back with nothing actionable, go straight to Phase 7 and merge. Never stop to say "reviews are clean, shall I merge?" — that is not a decision the owner needs to make.
- **Green checks are not a checkpoint.** When the safety checks pass, merge. Do not pause for confirmation.
- **The only valid stops** are: (a) a review finds something that needs a genuine judgment call the owner must make (a real design/product tradeoff, not a routine fix you can apply yourself), (b) a safety check goes red and the fix isn't obvious, or (c) the re-review loop hits its cap without converging. Anything you can fix, justify, or resolve yourself, you do — silently — and keep going.
- Report once, at the end, after the PR is merged. Progress narration mid-flow is fine; handing the turn back mid-flow is not.

Review sources that feed the loop: **Claude** (`/code-review` for correctness + `/security-review` for security, both run on the diff) and **Copilot** (inline review threads). Dependency/secret/CI scanning varies per repo — rely on whatever the repo's own CI provides; do not hunt for a specific external scanner summary comment.

**Copilot identity (verified, repo-independent):** Copilot's *review* author login is `copilot-pull-request-reviewer[bot]`; its *inline comment* author login is `Copilot`. Match either with the jq filter `(.user.login // .author.login) | test("copilot"; "i")`. Copilot's GraphQL bot node id is `BOT_kgDOCnlnWA`.

---

## Phase 0 — Pre-flight

1. **Detect repo context** — this command is repo-agnostic; derive everything from the current checkout. Set these once and reuse them everywhere below:
   ```bash
   REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)        # e.g. owner/name
   OWNER=${REPO%%/*}; NAME=${REPO##*/}
   BASE=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)  # e.g. main / master
   ```
   If `gh repo view` fails (no GitHub remote), stop and tell the owner — this flow needs a GitHub remote.
2. `git status` and `git diff --stat` — understand what changed.
3. If on the default branch (`$BASE`), branch: `git checkout -b <kebab-name-matching-task>`.
4. Existing PR on this branch? `gh pr list --head "$(git branch --show-current)" --json number,url`. If one exists, record its number as `PR` and skip **only the PR-creation step (Phase 1.4)** — you must still stage, commit, and push the current working-tree changes (Phase 1.1–1.3), otherwise reviews run against the stale remote state instead of the just-finished work.

---

## Phase 1 — Commit and open PR (requesting Copilot at creation)

1. Stage relevant files **by name**. Never `git add -A` — exclude `.env*`, secrets, unrelated files.
2. Commit in the repo style (imperative, ≤72-char subject, Co-Authored-By footer).
3. `git push -u origin "$(git branch --show-current)"`.
4. Create the PR **and request Copilot in the same command** (`--reviewer "@copilot"` triggers Copilot review on creation):
   ```bash
   gh pr create --reviewer "@copilot" --title "<imperative title>" --body "$(cat <<'EOF'
   ## Summary
   - <bullet>

   ## Test plan
   - <bullet>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   )"
   ```
5. Record `PR` (number) and the URL. Report both to the owner.

> If automatic Copilot review is already enabled on the repo, `--reviewer "@copilot"` is harmless (idempotent). If the repo has no Copilot review access, the request errors — note it and continue with Claude's reviews only.

> **Verifying the request landed:** do NOT trust `gh pr view --json reviewRequests` — it returns empty for *bot* reviewers even when Copilot is correctly requested. Confirm with the raw REST endpoint instead: `gh api /repos/$REPO/pulls/$PR --jq '.requested_reviewers[].login'` (shows `Copilot`), or the timeline `review_requested` event.

---

## Phase 2 — Gather reviews

Kick off 2A and 2C (both local, fast) and 2B (Copilot, async) together.

### 2A — Claude code review
Invoke `/code-review` at `medium` effort (`high` for diffs >200 lines). Capture findings as a list: `{file, line, description, severity}`.

### 2B — Wait for Copilot's review
Copilot was requested in Phase 1. Wait for it to post a review with a **background until-loop** (do NOT foreground-sleep — it's blocked in this harness). Run via Bash with `run_in_background: true` so one completion notification arrives when the review lands:
```bash
# exits when Copilot posts a review (state COMMENTED or APPROVED), or after ~12 min
for i in $(seq 1 24); do
  state=$(gh pr view $PR --json reviews \
    --jq '[.reviews[] | select(.author.login | test("copilot";"i"))] | last | .state' 2>/dev/null)
  if [ -n "$state" ] && [ "$state" != "null" ]; then echo "COPILOT_REVIEW_READY:$state"; exit 0; fi
  sleep 30
done
echo "COPILOT_TIMEOUT"; exit 0
```
If `COPILOT_TIMEOUT`, surface to the owner and continue with Claude's review only.

Then fetch Copilot's inline comments (note: author login here is `Copilot`):
```bash
gh api "/repos/$REPO/pulls/$PR/comments" --paginate \
  --jq '[.[] | select(.user.login | test("copilot";"i")) | {id:.id, path:.path, line:.line, body:.body}]'
```

### 2C — Claude security review
Invoke `/security-review` on the diff. Capture findings in the same shape as 2A: `{file, line, description, severity}`. Treat them exactly like code-review findings (no external comment, no thread).

---

## Phase 3 — Deduplicate and triage

Merge all sources — Claude code review (2A), Copilot (2B), Claude security review (2C) — into one deduplicated list:

- **Duplicate** (any two sources flag the same file+issue): one entry, address once.
- **Valid** → `FIX`.
- **Already correct / intentional** → `JUSTIFY`.
- **Nitpick / style** → `JUSTIFY`, unless it violates a repo convention (e.g. `CODING.md` / `CONTRIBUTING.md`) — then `FIX`.

Track each item's `source` (`claude` | `copilot`) — only Copilot items get a reply+resolve in Phase 5; Claude findings are just fixed.

---

## Phase 4 — Fix

For each `FIX`: apply the change, commit (`git commit -m "fix: <what>"` — group related fixes). For a finding you judge a false positive, leave a brief `// TODO(security-review): <why this is safe>` rather than contorting the code. After all fixes, `git push`. **Capture the head SHA** for the reply messages: `SHA=$(git rev-parse --short HEAD)`.

---

## Phase 5 — Reply to and resolve every Copilot thread

Applies to **Copilot items only** — Claude code-review and security-review findings have no thread to answer. For **every** Copilot inline comment (both `FIX` and `JUSTIFY`):

### Reply (note the PR number is in the path)
```bash
# Fixed:
gh api -X POST "/repos/$REPO/pulls/$PR/comments/$COMMENT_ID/replies" \
  -f body="Fixed in $SHA — <one line on what changed>."
# Justified:
gh api -X POST "/repos/$REPO/pulls/$PR/comments/$COMMENT_ID/replies" \
  -f body="Keeping as-is — <reason, referencing the convention/constraint>."
```

### Resolve the thread
Get thread ids, matching each thread's first comment `databaseId` to the Copilot `COMMENT_ID`s you replied to:
```bash
gh api graphql -f query='
query($owner:String!,$repo:String!,$pr:Int!){
  repository(owner:$owner,name:$repo){ pullRequest(number:$pr){
    reviewThreads(first:100){ pageInfo{ hasNextPage endCursor } nodes{ id isResolved comments(first:1){ nodes{ databaseId } } } } } } }' \
  -f owner=$OWNER -f repo=$NAME -F pr=$PR
```
`first:100` covers any realistic PR. The query returns `pageInfo` so the cap is **not silent**: if `hasNextPage` is `true`, re-run with `reviewThreads(first:100, after: "<endCursor>")` and keep collecting until it's `false` before resolving — don't assume one page is complete.
Resolve each matching unresolved thread:
```bash
gh api graphql -f query='
mutation($id:ID!){ resolveReviewThread(input:{threadId:$id}){ thread{ isResolved } } }' -f id=$THREAD_ID
```

---

## Phase 6 — Re-review loop

1. **Capture the current latest Copilot review timestamp first** — the 2B loop matches *any* Copilot review, so without a baseline a rerun exits instantly on the existing review and skips the re-review entirely:
   ```bash
   PREV_REVIEW_AT=$(gh pr view $PR --json reviews \
     --jq '[.reviews[] | select(.author.login | test("copilot";"i"))] | last | .submittedAt')
   ```
   Then re-request Copilot:
   ```bash
   gh pr edit $PR --add-reviewer "@copilot"
   ```
   If that errors (token/permission), fall back to the GraphQL re-request:
   ```bash
   PR_NODE=$(gh pr view $PR --json id --jq .id)
   gh api graphql -f query='
   mutation($pr:ID!,$bot:ID!){ requestReviews(input:{pullRequestId:$pr,botIds:[$bot],union:true}){ clientMutationId } }' \
     -f pr=$PR_NODE -f bot=BOT_kgDOCnlnWA
   ```
2. Wait for a review **newer than `$PREV_REVIEW_AT`** (background until-loop). This is the 2B loop with a freshness guard — do not reuse 2B verbatim:
   ```bash
   for i in $(seq 1 24); do
     latest=$(gh pr view $PR --json reviews \
       --jq '[.reviews[] | select(.author.login | test("copilot";"i"))] | last | .submittedAt' 2>/dev/null)
     # If there was no baseline review (initial wait timed out), accept any review;
     # otherwise require one strictly newer than the baseline.
     if [ -n "$latest" ] && [ "$latest" != "null" ] && { [ -z "$PREV_REVIEW_AT" ] || [ "$PREV_REVIEW_AT" = "null" ] || [[ "$latest" > "$PREV_REVIEW_AT" ]]; }; then
       echo "COPILOT_REREVIEW_READY:$latest"; exit 0
     fi
     sleep 30
   done
   echo "COPILOT_REREVIEW_TIMEOUT"; exit 0
   ```
   (ISO-8601 timestamps sort lexicographically, so `>` is a valid recency test.) Re-run `/security-review` on the new diff if the fixes touched security-relevant code.
3. Fetch only **new** Copilot comments — track comment ids already handled and diff against the full list (more robust than timestamps):
   ```bash
   gh api "/repos/$REPO/pulls/$PR/comments" --paginate \
     --jq '[.[] | select(.user.login | test("copilot";"i")) | {id:.id, path:.path, line:.line, body:.body}]'
   ```
   Drop ids you've already replied to/resolved.
4. New actionable comments → back to Phase 3 with only those.
5. Copilot `APPROVED`, or no new actionable comments → Phase 7.

**Loop limit:** after 3 full re-review cycles without convergence, stop, summarize the open items, and hand to the owner. Never loop forever.

---

## Phase 7 — Merge

**You are the merge gate.** The safety checks (everything except deploy-preview checks like Vercel / Netlify / Cloudflare Pages) must all be **passing** before you merge. Never merge while a safety check is pending or failing.

First, figure out which checks are required and whether the branch is protected:
```bash
# Required status checks from branch protection, if any (empty/❌ if branch is unprotected):
gh api "repos/$REPO/branches/$BASE/protection/required_status_checks" --jq '.contexts // .checks' 2>/dev/null
```

- **If the branch is protected with required checks:** prefer `gh pr merge $PR --auto --squash --delete-branch`. With required checks present, `--auto` genuinely gates — it merges only once they pass. You may still poll (below) to report status, but the gating is real.
- **If the branch is unprotected** (the `required_status_checks` call errored or returned empty): `--auto` does NOT gate — it merges immediately. So **you** poll and gate manually.

Poll the **non-deploy-preview** checks only — `gh pr checks --watch` would block on slow deploy previews. Run this as a background until-loop (don't foreground-sleep). Derive the expected safety-check set from what the PR actually reports (excluding deploy previews), and require it to be **non-empty** before breaking — right after PR creation GitHub can return an empty list before Actions register, and `jq all([])` is vacuously `true`, which would otherwise fall through to merge before any check ran:
```bash
# Deploy-preview check names to treat as non-blocking. Extend per repo if needed.
PREVIEW='vercel|netlify|cloudflare|render|preview|deploy'
for i in $(seq 1 30); do
  safety=$(gh pr checks $PR --json name,bucket --jq "[.[] | select(.name | test(\"$PREVIEW\"; \"i\") | not)]")
  echo "$safety" | jq -e \
    '(length > 0) and (map(.bucket) | all(. != "pending"))' >/dev/null 2>&1 && break
  sleep 30
done
echo "$safety" | jq '{failing: [.[]|select(.bucket=="fail").name], pending: [.[]|select(.bucket=="pending").name], passed: [.[]|select(.bucket=="pass").name]}'
gh pr view $PR --json mergeStateStatus,mergeable --jq '{mergeStateStatus, mergeable}'  # want CLEAN / MERGEABLE
```
- **Any safety check failing** → stop, fix it (loop back to Phase 4) or hand to the owner. Never merge red CI.
- **Any safety check still pending** → keep waiting; do not merge yet.
- **All safety checks pass** (deploy preview may still be running) → merge and delete the branch:
  ```bash
  gh pr merge $PR --squash --delete-branch
  ```
- `--delete-branch` also switches your local checkout back to `$BASE`. After it runs, `git checkout $BASE` is a no-op and a separate `git branch -D` will report "not found" — that's expected, not an error.

Final report to the owner:
- PR number + URL
- **Fixed** (brief bullets)
- **Justified** (bullets + the reasoning given to Copilot)
- Merge state: auto-merge enabled / merged / awaiting checks
