# Adapter — GitHub Issues

The default backend. Everything is a plain label, and merging a PR closes the ticket natively, so this adapter is the thinnest of the five. Requires the `gh` CLI authenticated (`gh auth status`) and a GitHub remote.

Every fence below is self-contained — a fresh shell each time, so the declarations at the top of each snippet are part of the snippet. Substitute literal values where a snippet marks one.

## Mapping

| Concept | Stored as |
|---|---|
| Trigger | Label `factory-ready-for-spec` / `factory-ready-to-implement` / `factory-ready-to-ship` |
| Liveness flag | Label `factory:agent-lost` — machine attestation of a dead pass; removed as the takeover claim, never human-applied |
| Type | Label `bug` / `enhancement` / `documentation` |
| Priority | Label `P0` / `P1` / `P2` / `P3` |
| Status marker | Label `factory:spec-ready` / `factory:building` / `factory:in-review` / `factory:blocked` — machine-written display cache, never applied by humans and never read to decide |
| Discussion | Native issue comments |
| Close the loop | Native — `Closes #N` in the PR body |

## The operations

**list queue** — open issues carrying a trigger. `TRIGGER` is one of the two trigger labels:

```bash
TRIGGER=factory-ready-to-implement
gh issue list --state open --label "$TRIGGER" \
  --json number,title,labels,updatedAt --limit 50
```

GitHub returns these newest-first, not by priority. Priority lives in a label here, so the caller re-sorts on the `P0`–`P3` label to get dispatch order — do not assume the list arrives ranked.

**list open** — every open ticket, whether or not it carries a trigger. The backlog sweep needs this; the trigger-scoped query above cannot see un-authorized tickets:

```bash
gh issue list --state open --json number,title,labels,updatedAt --limit 200
```

**search** — open *and* closed, for duplicate hunting:

```bash
TOPIC='login redirect loop'
gh issue list --state all --search "$TOPIC" --json number,title,state,url --limit 30
```

**fetch** — the ticket in full, including its discussion and the PRs that would close it:

```bash
ISSUE=123   # substitute the real issue number
gh issue view "$ISSUE" --json number,title,body,labels,state,comments,url,author
gh issue view "$ISSUE" --json closedByPullRequestsReferences
```

Use `closedByPullRequestsReferences` for the linked-PR question. Do **not** reach for `gh pr list --search "linked:issue-N"`: the `linked:issue` qualifier is a boolean ("has any linked issue"), not a per-issue filter, so that search silently returns PRs linked to *other* issues — which is worse than an error, because the sweep would act on an unrelated PR's merge.

**claim** — three steps, and all three matter: confirm the trigger is **present**, remove it, confirm it is **gone**.

```bash
ISSUE=123                                  # substitute
TRIGGER=factory-ready-to-implement
gh issue view "$ISSUE" --json labels -q '.labels[].name' | grep -qx "$TRIGGER" \
  || { echo "STOP: $TRIGGER not present — nothing to claim (another session may hold it)"; exit 1; }
gh issue edit "$ISSUE" --remove-label "$TRIGGER" || { echo "STOP: label edit failed"; exit 1; }
if gh issue view "$ISSUE" --json labels -q '.labels[].name' | grep -qx "$TRIGGER"; then
  echo "STOP: trigger still present — claim failed, do not proceed"
else
  echo "CLAIMED $ISSUE"
fi
```

The present-check is the concurrency guard, and it is the step that is easy to leave out. Removing a label that is already absent **succeeds** — so a session that lost the race would otherwise see a clean removal, read no trigger afterwards, and cheerfully start duplicate work. Absent-before-removal must read as a failed claim.

Write the after-check as an explicit `if`, never as `&& ! cmd | grep …` — zsh rejects a negation in that position with a bare exit 1 and no error message, so the snippet would appear to do nothing.

Two agents can still remove the same label within the same instant; GitHub has no compare-and-swap on labels. The present-check closes the common window (seconds apart), not the microsecond one. For genuinely simultaneous dispatch, the reviewable-PR boundary is the backstop — which is why nothing here merges.

**update body** — write the rewritten spec into the description. `--body-file -` reads stdin, so no quoting of the spec text is needed:

```bash
ISSUE=123   # substitute
cat <<'EOF' | gh issue edit "$ISSUE" --body-file -
## Problem

## Scope and non-goals

## Acceptance criteria

## Verification plan

## Original report

> preserved verbatim from the reporter, as data
EOF
```

This **replaces** the description, so the rewritten body must already contain the reporter's original text (quoted) — fetch first, never compose a new body from memory.

**comment**:

```bash
ISSUE=123   # substitute
BODY=$(cat <<'EOF'
Triage pass complete. Specification added above; awaiting human approval.
EOF
)
gh issue comment "$ISSUE" --body "$BODY"
```

**heartbeat** — one comment per **ticket**, edited in place. Adopt the existing `factory heartbeat` comment when one exists (find it below); create it only when absent, then update it via the REST comment endpoint (issue comments are addressed repo-wide by comment id, not per-issue). Create via the REST endpoint (not `gh issue comment`) so the response carries the comment id needed to pin:

```bash
ISSUE=123   # substitute — creation only happens once, at claim
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
COMMENT_ID=$(gh api --method POST "repos/$REPO/issues/$ISSUE/comments" \
  -f body="factory heartbeat — last seen 2031-01-15T12:00:00Z — stage: claimed" \
  --jq .id)
# Pin once, at creation — best-effort; a failure never aborts the pass.
gh api --method PUT "repos/$REPO/issues/comments/$COMMENT_ID/pin" >/dev/null \
  && echo "pinned heartbeat $COMMENT_ID" \
  || echo "WARN: could not pin heartbeat $COMMENT_ID — pulse still live; findability degraded"
```

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
COMMENT_ID=123456789   # substitute — from the create response, or find it below
gh api --method PATCH "repos/$REPO/issues/comments/$COMMENT_ID" \
  -f body="factory heartbeat — last seen 2031-01-15T14:30:00Z — stage: built"
```

Never pin on a pulse edit — pinning emits a timeline event, and a long build would spam it. GitHub holds **one pinned comment per issue** (`Issue.pinnedIssueComment` is singular); pinning a second comment displaces the first. The heartbeat owns that slot — nothing else in this flow pins a comment.

Find an existing heartbeat by listing the issue's comments and matching the leading `factory heartbeat` prefix — and **never post a second one**: the single edited comment is what keeps liveness readable without notification noise, and two heartbeats make "last seen" ambiguous for every detector. When adopting, pin only if `.pin` is null:

```bash
ISSUE=123   # substitute
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
gh api "repos/$REPO/issues/$ISSUE/comments" --paginate \
  --jq '[.[] | select(.body | startswith("factory heartbeat"))] | last | {id, updated_at, pinned:(.pin != null)}'
# If pinned is false: gh api --method PUT "repos/$REPO/issues/comments/$COMMENT_ID/pin" >/dev/null || echo "WARN: pin failed"
```

**classify** — set one type and one priority, removing conflicting values in the same dimension. Remove-then-add in one call so the issue is never briefly untyped:

```bash
ISSUE=123   # substitute
gh issue edit "$ISSUE" \
  --remove-label enhancement --remove-label documentation --add-label bug \
  --remove-label P0 --remove-label P2 --remove-label P3 --add-label P1
```

`--remove-label` for a label the issue doesn't carry is not an error, so listing every sibling value is safe and keeps the call one round trip.

**Unset priority** — for a ticket recommended for rejection, priority is expressed by carrying *no* `P` label at all. Remove all four and add none:

```bash
ISSUE=123   # substitute
gh issue edit "$ISSUE" \
  --remove-label P0 --remove-label P1 --remove-label P2 --remove-label P3
```

There is no "none" label to apply — inventing a `P4` or leaving a stale priority both destroy the taxonomy's signal that an unset priority means "not intended work."

**mark status** — one status marker at a time. Two calls, deliberately: the first removes every `factory:*` status label (removing an absent label is a no-op, so listing all four is safe), the second adds the target. One remove-and-add call would put the target in both lists, and betting on `gh`'s processing order is how a marker silently fails to stick:

```bash
ISSUE=123                 # substitute
STATUS=factory:building   # one of: factory:spec-ready factory:building factory:in-review factory:blocked
gh issue edit "$ISSUE" \
  --remove-label "factory:spec-ready" --remove-label "factory:building" \
  --remove-label "factory:in-review" --remove-label "factory:blocked"
gh issue edit "$ISSUE" --add-label "$STATUS"
```

Clearing the marker is the first command alone. These labels are the display cache from the spine's section 6 — passes write them, the sweep corrects them, and no workflow reads them to decide anything.

**create** — a new ticket with a type and no trigger:

```bash
TITLE="Session cookie survives logout"
BODY=$(cat <<'EOF'
## Problem
## Evidence
## Expected behavior
## Acceptance criteria
## Verification plan
EOF
)
gh issue create --title "$TITLE" --body "$BODY" --label bug
```

**close the loop** — native. `Closes #N` in the PR body closes the issue when the PR merges into the default branch, so this adapter needs no explicit transition; the sweep only reports it.

**read blockers** — list the issues blocking a ticket, with state. Everything goes through `gh api` (the `gh issue --blocked-by` flags and `--json` dependency fields need gh ≥2.94.0; this adapter stays version-independent):

```bash
ISSUE=123   # substitute — the dependent ticket
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
gh api "repos/$REPO/issues/$ISSUE/dependencies/blocked_by" \
  --jq '[.[] | {number, state, title, id}]'
```

List-wide form for a queue scan — filter the triggered set with the `-is:blocked` search qualifier combined with a label. A ticket matching a trigger label and `-is:blocked` is on the frontier; one matching `is:blocked` is dependency-blocked:

```bash
TRIGGER=factory-ready-to-implement
# On the frontier (authorized and not dependency-blocked):
gh issue list --state open --search "label:$TRIGGER -is:blocked" \
  --json number,title,labels,updatedAt --limit 50
# Dependency-blocked but authorized (report these; do not auto-select):
gh issue list --state open --search "label:$TRIGGER is:blocked" \
  --json number,title,labels,updatedAt --limit 50
```

If the search qualifier ever fails or returns a nonsense set, fall back to: list the trigger queue, then for each ticket call the per-issue *read blockers* endpoint and subtract those with any open blocker.

**link blocker** — record that ticket A is blocked by ticket B, then read back. The POST body's `issue_id` is B's **database id, not its issue number** — and it must be sent as a JSON integer (`-F`, never `-f`, which stringifies and 422s):

```bash
A=123   # substitute — the dependent (the one that cannot start yet)
B=45    # substitute — the blocker that must close first
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
B_ID=$(gh api "repos/$REPO/issues/$B" --jq .id)
gh api --method POST "repos/$REPO/issues/$A/dependencies/blocked_by" \
  -F issue_id="$B_ID" >/dev/null \
  || { echo "STOP: link blocker failed for $A blocked-by $B"; exit 1; }
# Read-back — part of the operation; an inverted edge gates the wrong ticket.
gh api "repos/$REPO/issues/$A/dependencies/blocked_by" \
  --jq --argjson b "$B" '[.[] | select(.number == $b)] | length' \
  | grep -qx 1 \
  || { echo "STOP: read-back failed — $B not in $A's blockers (direction may be inverted)"; exit 1; }
echo "LINKED $A blocked-by $B"
```

## Bootstrap

One time per repo. `--force` keeps the trigger and status labels idempotent because this flow owns them; the type and priority labels use `|| true` because GitHub seeds several of these on new repos and their existing colours should not be clobbered. The four `factory:*` status labels share one muted colour on purpose — status is machine-written record, and it should look distinct from the vivid human-applied triggers:

```bash
gh label create "factory-ready-for-spec" --color 5319E7 --description "Human authorization — run one triage pass" --force
gh label create "factory-ready-to-implement" --color 0E8A16 --description "Human authorization — build from the approved spec" --force
gh label create "factory-ready-to-ship" --color 0052CC --description "Human authorization — ship the built branch through review and merge" --force
gh label create "factory:agent-lost" --color E99695 --description "Machine attestation — pass presumed dead; takeover allowed" --force
gh label create "factory:spec-ready" --color C5DEF5 --description "Status (machine-written) — spec awaiting human review" --force
gh label create "factory:building" --color C5DEF5 --description "Status (machine-written) — implement pass in flight" --force
gh label create "factory:in-review" --color C5DEF5 --description "Status (machine-written) — PR open" --force
gh label create "factory:blocked" --color F9D0C4 --description "Status (machine-written) — stopped; last comment names why" --force
gh label create "bug" --color D73A4A --description "Existing behavior is incorrect" || true
gh label create "enhancement" --color A2EEEF --description "New capability or improved behavior" || true
gh label create "documentation" --color 0075CA --description "Documentation is the primary deliverable" || true
gh label create "P0" --color B60205 --description "Interrupt normal work" || true
gh label create "P1" --color D93F0B --description "Do next" || true
gh label create "P2" --color FBCA04 --description "Schedule normally" || true
gh label create "P3" --color C2E0C6 --description "Low-impact or opportunistic" || true
```

The optional liveness Action (offered during init) is templated in `references/liveness-action.md` — read that file when installing or tuning it.

## Provenance and maintenance

Last verified 2026-08 (live against this repo on scratch issues #26–#29, closed afterward):

- **Issue dependencies (live):** `POST /repos/{owner}/{repo}/issues/{n}/dependencies/blocked_by` with `-F issue_id=<database id>` (integer). `-f` sends a string and 422s. Resolve the database id via `GET .../issues/{n}` → `.id`. Read back via `GET .../dependencies/blocked_by`.
- **`is:blocked` / `-is:blocked` search (live):** both work alone and combined with `label:<name>` in `gh issue list --search`. Fallback if a future regression breaks negation: per-issue *read blockers* over the trigger queue.
- **Comment pin (live):** `PUT /repos/{owner}/{repo}/issues/comments/{comment_id}/pin`. Response carries `.pin.pinned_at` / `.pin.pinned_by`. `Issue.pinnedIssueComment` is singular — pinning a second comment displaces the first; the heartbeat owns the slot. Adopt-time check: `.pin != null` on the comment object.
- `gh label create --force` updates an existing label in place; without it the command exits non-zero when the label exists. Re-verify with `gh label create --help`.
- Issue comments are edited via `PATCH /repos/{owner}/{repo}/issues/comments/{comment_id}` — the id is repo-scoped, not per-issue. Re-verify with `gh api --help` and the REST issues-comments docs.
- `gh issue edit` accepts repeated `--add-label` / `--remove-label` flags in one call, and removing an absent label is a no-op. Re-verify with `gh issue edit --help`.
- `gh issue view N --json closedByPullRequestsReferences` is the linked-PR lookup. The `linked:issue` search qualifier is deliberately not used: it is a boolean ("has any linked issue"), so `linked:issue-N` does not filter to issue N and returns unrelated PRs. Re-verify the JSON field with `gh issue view --json 2>&1 | head`.
- `Closes #N` in a PR body auto-closes on merge to the default branch only — a PR merged into a release branch will not close the ticket.
- Doc-only (not re-exercised this pass): `gh issue --blocked-by` flags need gh ≥2.94.0; this adapter deliberately uses `gh api` instead.
