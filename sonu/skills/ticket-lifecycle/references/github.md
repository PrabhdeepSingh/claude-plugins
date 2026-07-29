# Adapter — GitHub Issues

The default backend. Everything is a plain label, and merging a PR closes the ticket natively, so this adapter is the thinnest of the five. Requires the `gh` CLI authenticated (`gh auth status`) and a GitHub remote.

Every fence below is self-contained — a fresh shell each time, so the declarations at the top of each snippet are part of the snippet. Substitute literal values where a snippet marks one.

## Mapping

| Concept | Stored as |
|---|---|
| Trigger | Label `factory-ready-for-spec` / `factory-ready-to-implement` |
| Type | Label `bug` / `enhancement` / `documentation` |
| Priority | Label `P0` / `P1` / `P2` / `P3` |
| Discussion | Native issue comments |
| Close the loop | Native — `Closes #N` in the PR body |

## The seven operations

**list queue** — open issues carrying a trigger. `TRIGGER` is one of the two trigger labels:

```bash
TRIGGER=factory-ready-to-implement
gh issue list --state open --label "$TRIGGER" \
  --json number,title,labels,updatedAt --limit 50
```

GitHub returns these newest-first, not by priority. Priority lives in a label here, so the caller re-sorts on the `P0`–`P3` label to get dispatch order — do not assume the list arrives ranked.

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

**comment**:

```bash
ISSUE=123   # substitute
BODY=$(cat <<'EOF'
Triage pass complete. Specification added above; awaiting human approval.
EOF
)
gh issue comment "$ISSUE" --body "$BODY"
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

## Bootstrap

One time per repo. `--force` keeps the two trigger labels idempotent because this flow owns them; the type and priority labels use `|| true` because GitHub seeds several of these on new repos and their existing colours should not be clobbered:

```bash
gh label create "factory-ready-for-spec" --color 5319E7 --description "Human authorization — run one triage pass" --force
gh label create "factory-ready-to-implement" --color 0E8A16 --description "Human authorization — build from the approved spec" --force
gh label create "bug" --color D73A4A --description "Existing behavior is incorrect" || true
gh label create "enhancement" --color A2EEEF --description "New capability or improved behavior" || true
gh label create "documentation" --color 0075CA --description "Documentation is the primary deliverable" || true
gh label create "P0" --color B60205 --description "Interrupt normal work" || true
gh label create "P1" --color D93F0B --description "Do next" || true
gh label create "P2" --color FBCA04 --description "Schedule normally" || true
gh label create "P3" --color C2E0C6 --description "Low-impact or opportunistic" || true
```

## Provenance and maintenance

Last verified 2026-07:

- `gh label create --force` updates an existing label in place; without it the command exits non-zero when the label exists. Re-verify with `gh label create --help`.
- `gh issue edit` accepts repeated `--add-label` / `--remove-label` flags in one call, and removing an absent label is a no-op. Re-verify with `gh issue edit --help`.
- `gh issue view N --json closedByPullRequestsReferences` is the linked-PR lookup. The `linked:issue` search qualifier is deliberately not used: it is a boolean ("has any linked issue"), so `linked:issue-N` does not filter to issue N and returns unrelated PRs. Re-verify the JSON field with `gh issue view --json 2>&1 | head`.
- `Closes #N` in a PR body auto-closes on merge to the default branch only — a PR merged into a release branch will not close the ticket.
