---
description: Branch, commit, open PR, gather Claude + every enabled AI reviewer (Copilot, CodeRabbit, etc.), fix/justify/resolve every finding, loop until clean, then merge
argument-hint: "[light|full]"
allowed-tools: Bash, Read, Edit, Write, Skill
---

# /ship — PR Babysitter

Handles everything from the current working-tree state through a clean, merged PR. Run after implementation is complete and the owner has said "go". Do not stop until the PR is merged (or auto-merging), or a decision only the owner can make is reached.

**Autonomy contract — run start-to-finish without checking back.** Invoking `/ship` (or saying "ship it") IS the authorization for the entire flow through merge. Once started, flow through every phase — including the final merge — without pausing to report or to ask for a go-ahead. In particular:

- **Clean reviews are not a checkpoint.** If every review source comes back with nothing actionable, go straight to Phase 7 and merge. Never stop to say "reviews are clean, shall I merge?" — that is not a decision the owner needs to make.
- **Green checks are not a checkpoint.** When the safety checks pass, merge. Do not pause for confirmation.
- **The only valid stops** are: (a) a review finds something that needs a genuine judgment call the owner must make (a real design/product tradeoff, not a routine fix you can apply yourself), (b) a safety check goes red and the fix isn't obvious, or (c) the re-review loop hits its cap without converging. Anything you can fix, justify, or resolve yourself, you do — silently — and keep going.
- Report once, at the end, after the PR is merged. Progress narration mid-flow is fine; handing the turn back mid-flow is not.

**No AI attribution.** Do NOT add `Co-Authored-By` trailers, "Generated with Claude Code" lines, or any other AI/tool attribution to commits or the PR body. Commits and PRs read as the owner's own work.

---

## Effort mode — right-size the spend to the change

`$ARGUMENTS` may carry a mode (the text typed after the command; if that token appears literally or is empty, default to `auto`). Parse it forgivingly (it's read by you, not a strict parser) — accept synonyms and misspellings:

- **`light`** ← also `lite`, `quick`, `fast`, `cheap`, `min`, and obvious typos.
- **`full`** ← also `thorough` (and misspellings like `thurough`/`thourough`), `deep`, `max`, `heavy`.
- **no arg → `auto`**: decide from the diff (see below).

The mode scales **only the reviews you pay for** — your own `/code-review` and `/security-review`. It does **not** change the external AI bots: those are configured on the repo/org and auto-trigger when the PR opens, so they cost the same whether you wait for them or not. Always collect whatever they post.

| Mode | Your `/code-review` | Your `/security-review` | Re-review loop |
|------|---------------------|--------------------------|----------------|
| **light** | low effort; skip entirely if the diff is trivial (≤ ~10 lines, only CSS/markup/docs/config/comments, no logic) | skip unless a security-relevant file is touched (auth, api, sql, middleware, crypto, payments/stripe, env, session, headers, file I/O) | 1 cycle max |
| **auto** | trivial diff → treat as **light**; > 200 lines or security-relevant files → treat as **full**; otherwise medium effort | run unless the diff is trivial and non-security | up to 3 cycles |
| **full** | high effort | always | up to 3 cycles |

When `light` skips a review, say so in one line in the final report — don't let a skipped pass read as "clean."

---

## Phase 0 — Pre-flight

1. **Detect repo context** — this command is repo-agnostic; derive everything from the current checkout. Set these once and reuse them everywhere below:
   ```bash
   REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)        # e.g. owner/name
   OWNER=${REPO%%/*}; NAME=${REPO##*/}
   BASE=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)  # e.g. main / master
   ```
   If `gh repo view` fails (no GitHub remote), stop and tell the owner — this flow needs a GitHub remote.
2. `git status` and `git diff --stat` — understand what changed. Use the line count + file types to pick the effort mode (above).
3. If on the default branch (`$BASE`), branch: `git checkout -b <kebab-name-matching-task>`.
4. Existing PR on this branch? `gh pr list --head "$(git branch --show-current)" --json number,url`. If one exists, record its number as `PR` and skip **only the `gh pr create` call (Phase 1.5)** — you must still stage, commit, push, and run self-review (Phase 1.1–1.4). After self-review, refresh the PR description: invoke `Skill(sonu:pr-conventions)` (Section C — *Keep the description current*) to re-render the body in place — updating Summary/Changes and refreshing the Risk section from the new `RISKS` list while preserving the team-template structure. Capture the updated body into `BODY` explicitly before writing it back (passing an unset variable to `--body` will blank the PR description):
   ```bash
   BODY=$(cat <<'PREOF'
   <updated body text from Skill(sonu:pr-conventions) Section C — replace this block>
   PREOF
   )
   gh pr edit $PR --body "$BODY"
   ```
   Otherwise the PR body reflects stale risk info and reviewers see a new diff with no context.

---

## Phase 1 — Commit and open PR (requesting Copilot at creation)

1. Stage relevant files **by name**. Never `git add -A` — exclude `.env*`, secrets, unrelated files.
2. Commit in the repo style (imperative, ≤72-char subject). **No AI attribution / no `Co-Authored-By` trailer** (see the contract above).
3. `git push -u origin "$(git branch --show-current)"`.
4. **Run `Skill(sonu:self-review)` on the committed diff** (`git show HEAD` / `git diff HEAD^ HEAD` — the working tree is clean at this point, so `git diff HEAD` would return nothing). This surfaces the 3–5 riskiest spots in the change so they can be embedded in the PR body for traceability and shown to the owner. Capture the list — call it `RISKS`.
5. **Invoke `Skill(sonu:pr-conventions)`** to compose the PR body — the skill scans for a team `PULL_REQUEST_TEMPLATE` first (wins over built-ins if found), classifies the change type from the branch name / commit prefix / diff, fills the matching per-type template, and embeds the `RISKS` list from step 4 in the risk section. Do not put any AI-attribution line in the body. **Capture the composed text into `BODY` explicitly** (never pass `--body "$BODY"` with an unset variable — that opens the PR with a blank description):
   ```bash
   BODY=$(cat <<'PREOF'
   <body text composed by Skill(sonu:pr-conventions) — replace this entire block>
   PREOF
   )
   gh pr create --reviewer "@copilot" --title "<imperative title>" --body "$BODY"
   ```
6. Record `PR` (number) and the URL. Report both to the owner.

> If automatic Copilot review is already enabled on the repo, `--reviewer "@copilot"` is harmless (idempotent). If the repo has no Copilot access, the request errors — note it and continue with whatever else reviews.

> **Verifying the request landed:** do NOT trust `gh pr view --json reviewRequests` — it returns empty for *bot* reviewers even when Copilot is correctly requested. Confirm with the raw REST endpoint instead: `gh api /repos/$REPO/pulls/$PR --jq '.requested_reviewers[].login'` (shows `Copilot`), or the timeline `review_requested` event.

---

## Phase 2 — Gather reviews

Three sources feed the loop: **your own Claude reviews** (2A, 2B — run per the effort mode), and **every AI reviewer bot enabled on the repo** (2C). Kick the local reviews (2A/2B) and the bot wait (2C) off together — 2C is the async one.

**Why a participation scan, not a config-file scan:** AI review bots (CodeRabbit, Aikido, Qodo, Greptile, Ellipsis, Sourcery, Cubic, Korbit, …) are usually enabled at the org/app level, NOT via a repo config file — so a file scan misses them. The reliable, repo-agnostic signal is *who actually posts on the PR*. Opening the PR auto-triggers every enabled bot; Copilot was requested in Phase 1. So: wait a bounded window, then harvest whichever bots actually showed up and match them against the registry below. This "just knows" — it adapts to each repo (your work repos surface CodeRabbit + Aikido + Copilot; a personal repo surfaces only Copilot) with zero configuration and nothing to keep in sync.

### Known AI-reviewer login registry (match case-insensitively)
| Tool | Author login(s) | Notes |
|------|-----------------|-------|
| GitHub Copilot | `copilot-pull-request-reviewer[bot]` (review), `Copilot` (inline comments) | Requested, not auto. Always a non-blocking COMMENT review. |
| CodeRabbit | `coderabbitai[bot]` | Auto on open. `@coderabbitai review` to retrigger; `@coderabbitai resolve`. |
| Aikido | `aikido-autofix[bot]` | Security scanner; auto via CI. |
| Qodo Merge | `qodo-merge-pro[bot]` | Auto on open. `/review`. |
| Greptile | `greptile-apps[bot]` | Auto on ready. `@greptileai`. |
| Ellipsis | `ellipsis-dev[bot]` | Auto on ready. `@ellipsis-dev`. |
| Sourcery | `sourcery-ai[bot]` | Auto on ready. `@sourcery-ai review`. |
| Cubic | `cubic-dev-ai[bot]` | Auto on open. `@cubic-dev-ai`. |
| Korbit | `korbit-ai[bot]` | Auto on open. |

Treat an author as an AI reviewer if its lowercased login matches the registry (substring match on `copilot`, `coderabbit`, `aikido`, `qodo`, `greptile`, `ellipsis`, `sourcery`, `cubic`, `korbit`). Any *other* `[bot]` that posts an actual PR **review** (not just a status comment) is a probable reviewer too — include it and note it.

### 2A — Claude code review
Per the effort mode, invoke `/code-review` (low/medium/high) — or skip in `light` on a trivial diff. Capture findings: `{file, line, description, severity}`.

### 2B — Claude security review
Per the effort mode, invoke `/security-review` on the diff — or skip when the mode says to. Capture findings in the same shape. Treat them like code-review findings (no external comment, no thread).

### 2C — Wait for the AI reviewer bots
Opening the PR triggered every enabled bot; Copilot was requested. Wait for them with a **background until-loop** (do NOT foreground-sleep — it's blocked in this harness). Run via Bash with `run_in_background: true`. You don't know the full guest list in advance, so wait until activity settles: break once Copilot has reviewed AND the set of participating bots has been stable for two consecutive polls (no newcomers), or after ~10 min.
```bash
BOT_RE='copilot|coderabbit|aikido|qodo|greptile|ellipsis|sourcery|cubic|korbit'
prev=""; stable=0
for i in $(seq 1 20); do
  # union of bot logins seen across reviews + inline comments + issue comments
  bots=$(gh api "/repos/$REPO/pulls/$PR/reviews" --paginate --jq '.[].user.login' 2>/dev/null; \
         gh api "/repos/$REPO/pulls/$PR/comments" --paginate --jq '.[].user.login' 2>/dev/null; \
         gh api "/repos/$REPO/issues/$PR/comments" --paginate --jq '.[].user.login' 2>/dev/null) 
  bots=$(echo "$bots" | tr 'A-Z' 'a-z' | grep -E "$BOT_RE" | sort -u | tr '\n' ',')
  copilot_done=$(gh pr view $PR --json reviews --jq '[.reviews[] | select(.author.login|test("copilot";"i"))] | length' 2>/dev/null)
  if [ "$bots" = "$prev" ] && [ -n "$bots" ]; then stable=$((stable+1)); else stable=0; fi
  prev="$bots"
  # settle: Copilot in, and bot set unchanged for 2 polls
  if [ "${copilot_done:-0}" -ge 1 ] && [ "$stable" -ge 2 ]; then echo "BOTS_SETTLED:$bots"; exit 0; fi
  sleep 30
done
echo "BOTS_TIMEOUT:$prev"; exit 0
```
If it times out with some bots still absent, surface that and continue with whoever did post (the Phase 6 loop catches stragglers). Then fetch every bot's inline comments (this is the registry-matched harvest):
```bash
gh api "/repos/$REPO/pulls/$PR/comments" --paginate \
  --jq "[.[] | select(.user.login | ascii_downcase | test(\"$BOT_RE\")) | {id:.id, login:.user.login, path:.path, line:.line, body:.body}]"
```
Also grab each bot's review-level summary (some put findings in the review body, not inline):
```bash
gh api "/repos/$REPO/pulls/$PR/reviews" --paginate \
  --jq "[.[] | select(.user.login | ascii_downcase | test(\"$BOT_RE\")) | {login:.user.login, state:.state, body:.body}]"
```

Also harvest **human inline review comments** for Phase 5 reply handling. Re-declare `BOT_RE` in this command — each Bash invocation is its own shell, so the variable from the wait-loop above is not automatically in scope:
```bash
BOT_RE='copilot|coderabbit|aikido|qodo|greptile|ellipsis|sourcery|cubic|korbit'
gh api "/repos/$REPO/pulls/$PR/comments" --paginate \
  --jq "[.[] | select(.user.login | ascii_downcase | test(\"$BOT_RE\") | not) | select(.user.type != \"Bot\") | {id:.id, login:.user.login, path:.path, line:.line, body:.body}]"
```

---

## Phase 3 — Deduplicate and triage

Merge all sources — your Claude code review (2A), your Claude security review (2B), AI bot findings from 2C, and **human inline comments** from 2C — into one deduplicated list:

- **Duplicate** (any two sources flag the same file+issue): one entry, address once. (Bots overlap a lot — expect heavy dedup.)
- **Valid** → `FIX`.
- **Already correct / intentional** → `JUSTIFY`.
- **Nitpick / style** → `JUSTIFY`, unless it violates a repo convention (e.g. `CODING.md` / `CONTRIBUTING.md`) — then `FIX`.

Track each item's `source` (`claude`, `bot`, or `human`) and, for bot and human findings, the `login` + `comment_id` — bots get reply+resolve in Phase 5; humans get reply-only (no resolve); your own Claude findings are just fixed.

---

## Phase 4 — Fix

For each `FIX`: apply the change, commit (`git commit -m "fix: <what>"` — group related fixes; **no `Co-Authored-By` trailer**). For a finding you judge a false positive, leave a brief `// TODO(review): <why this is safe>` rather than contorting the code. After all fixes, `git push`. **Capture the head SHA** for the reply messages: `SHA=$(git rev-parse --short HEAD)`.

Then refresh the PR description: invoke `Skill(sonu:pr-conventions)` (Section C — *Keep the description current*) to update Summary/Changes bullets to reflect the fixes and re-render the Risk section if the fix surface changed. This applies on the first fix pass and within every cycle of the Phase 6 loop.

---

## Phase 5 — Reply to every review thread; resolve bot threads

Applies to **all inline comments** — bot threads and human reviewer threads. Your own Claude code-review and security-review findings have no thread to answer. Reply wording comes from `Skill(sonu:pr-conventions)` Section D. Mechanics differ by source:

- **Bot threads** (Copilot, CodeRabbit, Greptile, …): reply + resolve the thread (see Resolve section below).
- **Human threads**: reply only — never resolve a human's thread; leave that to them.

For **every** inline comment (both `FIX` and `JUSTIFY`):

### Reply (the PR number is in the path)
```bash
# Fixed:
gh api -X POST "/repos/$REPO/pulls/$PR/comments/$COMMENT_ID/replies" \
  -f body="Fixed in $SHA — <one line on what changed>."
# Justified:
gh api -X POST "/repos/$REPO/pulls/$PR/comments/$COMMENT_ID/replies" \
  -f body="Keeping as-is — <reason, referencing the convention/constraint>."
```

### Resolve the thread (bot threads only — never resolve a human's thread)
Get thread ids, matching each thread's first comment `databaseId` to the **bot** `COMMENT_ID`s you replied to. Skip any `COMMENT_ID` whose `source` is `human`:
```bash
gh api graphql -f query='
query($owner:String!,$repo:String!,$pr:Int!){
  repository(owner:$owner,name:$repo){ pullRequest(number:$pr){
    reviewThreads(first:100){ pageInfo{ hasNextPage endCursor } nodes{ id isResolved comments(first:1){ nodes{ databaseId author{ login } } } } } } } }' \
  -f owner=$OWNER -f repo=$NAME -F pr=$PR
```
`first:100` covers any realistic PR. The query returns `pageInfo` so the cap is **not silent**: if `hasNextPage` is `true`, re-run with `reviewThreads(first:100, after: "<endCursor>")` and keep collecting until it's `false` before resolving — don't assume one page is complete.
Resolve each matching unresolved thread:
```bash
gh api graphql -f query='
mutation($id:ID!){ resolveReviewThread(input:{threadId:$id}){ thread{ isResolved } } }' -f id=$THREAD_ID
```
(Some bots also honor their own resolve command — e.g. `@coderabbitai resolve` — but `resolveReviewThread` works uniformly, so prefer it.)

---

## Phase 6 — Re-review loop

(`light` mode: at most 1 cycle. `auto`/`full`: up to 3.)

1. **Capture each bot's current latest review timestamp first** — a rerun must wait for activity *newer* than what's already there, or it exits instantly on the existing reviews:
   ```bash
   PREV_AT=$(gh pr view $PR --json reviews \
     --jq "[.reviews[] | select(.author.login | ascii_downcase | test(\"$BOT_RE\"))] | (map(.submittedAt) | max) // \"\"")
   ```
   Then re-trigger the bots:
   - **Copilot:** `gh pr edit $PR --add-reviewer "@copilot"` (fallback if it errors: GraphQL `requestReviews` with `botIds:["BOT_kgDOCnlnWA"]` — Copilot's node id — and `union:true`).
   - **Other bots** generally re-review automatically on a new push (your Phase 4 `git push`). For any that don't, drop their re-review mention as an issue comment (`@coderabbitai review`, `@sourcery-ai review`, `@greptileai`, `@ellipsis-dev`, `/review` for Qodo).
2. Wait for activity **newer than `$PREV_AT`** using the explicit loop below. Run it as a background until-loop (do NOT foreground-sleep).

   **CRITICAL: run this loop separately from the Phase 7 CI poll. Never combine them into one loop.** Two conditions (re-review timestamp AND CI buckets) cannot be safely merged — the variable-capture patterns are incompatible and a combined loop will stall. Run this re-review loop first, collect new findings, then run the CI poll loop in Phase 7.

   ISO-8601 sorts lexicographically, so a string `>` is a valid recency test — but **do not write `[ "$MAX_AT" \> "$PREV_AT" ]`**: this harness runs under `zsh`, whose `[`/`test` builtin rejects `\>` with `condition expected: >`. Use `[[ ... > ... ]]` instead (works in both bash and zsh):
   ```bash
   BOT_RE='copilot|coderabbit|aikido|qodo|greptile|ellipsis|sourcery|cubic|korbit'
   for i in $(seq 1 20); do
     MAX_AT=$(gh pr view $PR --json reviews \
       --jq "[.reviews[] | select(.author.login | ascii_downcase | test(\"$BOT_RE\"))] | (map(.submittedAt) | max) // \"\"" 2>/dev/null)
     if [ -n "$MAX_AT" ] && [[ "$MAX_AT" > "$PREV_AT" ]]; then echo "NEW_REVIEW:$MAX_AT"; exit 0; fi
     sleep 30
   done
   echo "REREVIEW_TIMEOUT"
   ```
   Re-run `/security-review` on the new diff if the fixes touched security-relevant code (and the mode runs it).
3. Fetch only **new** comments — both bot and human inline — that you haven't already handled. Re-declare `BOT_RE` (each Bash call is a separate shell):
   ```bash
   BOT_RE='copilot|coderabbit|aikido|qodo|greptile|ellipsis|sourcery|cubic|korbit'
   # New bot comments:
   gh api "/repos/$REPO/pulls/$PR/comments" --paginate \
     --jq "[.[] | select(.user.login | ascii_downcase | test(\"$BOT_RE\")) | {id:.id, login:.user.login, path:.path, line:.line, body:.body}]"
   # New human comments:
   gh api "/repos/$REPO/pulls/$PR/comments" --paginate \
     --jq "[.[] | select(.user.login | ascii_downcase | test(\"$BOT_RE\") | not) | select(.user.type != \"Bot\") | {id:.id, login:.user.login, path:.path, line:.line, body:.body}]"
   ```
   Drop ids you've already replied to/resolved (from prior loop cycles).
4. New actionable comments → back to Phase 3 with only those.
5. All bots approved or quiet (no new actionable comments) → Phase 7.

**Loop limit:** after the mode's cycle cap without convergence, stop, summarize the open items, and hand to the owner. Never loop forever.

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

Poll the **non-deploy-preview** checks only — `gh pr checks --watch` would block on slow deploy previews. Run this as a background until-loop (don't foreground-sleep). Require the safety-check set to be **non-empty** before breaking — right after PR creation GitHub can return an empty list before Actions register, and `jq all([])` is vacuously `true`, which would otherwise fall through to merge before any check ran.

**jq boolean pattern warning:** Do NOT write `done=$(jq -e '...' && echo "yes" || echo "no")`. `jq -e` always prints `true`/`false` to stdout before `&&` runs, so `$()` captures `"true\nyes"` — a multi-line string that never equals `"yes"` and the loop never breaks. The pattern below pipes through `>/dev/null 2>&1` to discard jq's output and uses only its exit code to drive `break` — copy it exactly, don't adapt it:
```bash
PREVIEW='vercel|netlify|cloudflare|render|preview|deploy'
for i in $(seq 1 30); do
  safety=$(gh pr checks $PR --json name,bucket --jq "[.[] | select(.name | test(\"$PREVIEW\"; \"i\") | not)]")
  echo "$safety" | jq -e '(length > 0) and (map(.bucket) | all(. != "pending"))' >/dev/null 2>&1 && break
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
- Effort mode used, and any review deliberately skipped (so a skip never reads as "clean")
- AI reviewers that participated (e.g. Copilot, CodeRabbit) + any expected-but-absent
- **Risk / reviewer attention** — the 3–5 items from the self-review (same list as in the PR body)
- **Fixed** (brief bullets)
- **Justified** (bullets + the reasoning given to the bots)
- **Human threads replied to** — N comments answered; none auto-resolved (resolution left to the reviewer)
- Merge state: auto-merge enabled / merged / awaiting checks
