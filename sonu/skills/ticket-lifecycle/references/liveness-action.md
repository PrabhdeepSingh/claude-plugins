# Liveness Action — flag lost factory passes when nobody is watching

The factory sweep checks liveness on every pass — but a sweep only runs when a session runs. If the only polling machine dies, nothing is left to notice. This optional GitHub Action closes that hole: on a schedule, it applies the **same criteria the sweep uses** and flags presumably-dead passes with `factory:agent-lost` plus one descriptive comment.

**It only ever flags.** It never applies a trigger, never removes one, never comments instructions, and never starts work. Takeover semantics live in the factory poll route; a human who wants a fresh pass re-applies the stage trigger. GitHub tracker only — the other adapters rely on the sweep alone.

## When this file is read

- `/sonu:factory init` on the `github` tracker offers to install it — write the template below to `.github/workflows/factory-liveness.yml` in the managed repo if the user says yes. Skipping costs only unattended detection.
- Re-init on a repo where the workflow is already installed — compare the installed copy against the template below and offer the updated version when they differ, carrying over the repo's tuned values (`STALE_HOURS`, the cron cadence). Installed copies are pinned; this re-offer is how a criteria fix reaches them.
- A user tuning the stale threshold or auditing what the Action can do.

## What it needs

- **Token:** the workflow's own `GITHUB_TOKEN` with the `permissions:` block below — `issues: write` to label and comment, `pull-requests: read` for the PR-activity check. No personal token, no secret to provision. That token can label, so the Action sits inside the same trust boundary as anyone who can apply labels — say so when installing it.
- **Runner:** `ubuntu-latest` (the script uses GNU `date -d`; the `gh` CLI is preinstalled on GitHub-hosted runners).

## The criteria (the sweep's, applied more conservatively)

A ticket is flagged only when **all** hold: open, carrying `factory:building` or `factory:in-review` (never `blocked` — waiting on a human is not death, so blocked tickets are exempt entirely); no `factory-ready-*` trigger present (a trigger means queued, not claimed); a `factory heartbeat` comment exists and is older than the threshold (no heartbeat → skip: a pass from before heartbeats existed should never be flagged on absence of evidence — the one place this is stricter than the sweep, which may fall back to checkpoint ages); the heartbeat's body does not **end with** `stage: built` — anchored at the end, trailing whitespace tolerated, exactly as the script's jq `test` does; a mid-build heartbeat reads `stage: building — step k/N: …` and must stay flaggable (the hand-back writes `stage: built` when it parks a green build for the human's ship decision — built-awaiting-ship is waiting, exactly like `blocked`, and it is the flow's normal success path: flagging it would mark every unshipped green build dead); and no open PR on the ticket's branch has activity newer than the threshold (a ship loop legitimately sits silent while bots review — PR activity is life). Bias alive: a false "lost" costs duplicate work; a missed one only delays pickup.

## Template

```yaml
name: factory-liveness

on:
  schedule:
    - cron: "*/30 * * * *"
  workflow_dispatch: {}

# One scan at a time: overlapping runs can both pass the already-flagged check
# and post duplicate comments. cancel-in-progress stays false on purpose — the
# scan writes (label, then comment), and cancelling between the two writes
# leaves a flag with no explanatory record; a queued run waiting is harmless.
concurrency:
  group: factory-liveness
  cancel-in-progress: false

permissions:
  issues: write
  pull-requests: read

env:
  STALE_HOURS: "2"

jobs:
  flag-lost-passes:
    runs-on: ubuntu-latest
    steps:
      - name: Flag presumably-dead factory passes
        env:
          GH_TOKEN: ${{ github.token }}
          REPO: ${{ github.repository }}
        run: |
          set -euo pipefail
          now=$(date -u +%s)
          stale_s=$(( STALE_HOURS * 3600 ))
          for label in "factory:building" "factory:in-review"; do
            gh issue list --repo "$REPO" --state open --label "$label" \
              --limit 200 --json number --jq '.[].number' |
            while read -r n; do
              labels=$(gh issue view "$n" --repo "$REPO" --json labels --jq '[.labels[].name]')
              # already flagged, still queued (trigger present), or blocked (waiting on a
              # human — exempt from liveness even on a drifted label set) -> never flag
              echo "$labels" | jq -e 'index("factory:agent-lost")' >/dev/null && continue
              echo "$labels" | jq -e 'index("factory:blocked")' >/dev/null && continue
              echo "$labels" | jq -e 'map(select(startswith("factory-ready-"))) | length > 0' >/dev/null && continue
              # Heartbeat record: timestamp + "is this the built hand-back?", as two TSV
              # fields. jq decides the second one, on the real body, because the body must
              # never reach the shell: @tsv escapes a trailing newline to the two literal
              # characters \n, and an end-anchored shell match against that fails — which
              # would flag the very built ticket this check exists to exempt.
              # tail -n 1: --paginate emits one jq result PER PAGE, so a >100-comment
              # ticket would otherwise make $hb multi-line and break the date parse.
              hb=$(gh api "repos/$REPO/issues/$n/comments" --paginate \
                --jq '[.[] | select(.body | startswith("factory heartbeat"))] | last | select(. != null) | [.updated_at, (.body | test("stage: built[[:space:]]*$"))] | @tsv' \
                | tail -n 1)
              # no heartbeat means no evidence, and absence of evidence never flags
              [ -n "$hb" ] || continue
              # stage: built is the implement pass's hand-back — a green build parked for
              # the human's ship decision. Waiting, exactly like blocked -> never flag.
              # Anchored at the line's end (trailing whitespace tolerated) on purpose: a
              # substring match would also exempt "stage: building — step k/N", a live or
              # dead build, which must stay flaggable.
              [ "$(printf '%s\n' "$hb" | cut -f2)" = "true" ] && continue
              last=$(printf '%s\n' "$hb" | cut -f1)
              [ $(( now - $(date -u -d "$last" +%s) )) -gt "$stale_s" ] || continue
              # PR activity on the ticket branch counts as life (ship loops wait silently on bots)
              pr_last=$(gh pr list --repo "$REPO" --state open --limit 100 --json headRefName,updatedAt \
                --jq "[.[] | select(.headRefName | startswith(\"ticket/$n-\"))] | (max_by(.updatedAt) | .updatedAt) // empty")
              if [ -n "$pr_last" ]; then
                [ $(( now - $(date -u -d "$pr_last" +%s) )) -gt "$stale_s" ] || continue
              fi
              gh issue edit "$n" --repo "$REPO" --add-label "factory:agent-lost"
              gh issue comment "$n" --repo "$REPO" --body "factory liveness: pass presumed lost — last heartbeat $last, threshold ${STALE_HOURS}h. A polling session may take this over (factory poll route), or a human re-applies the stage trigger to restart. This flag authorizes nothing by itself."
            done
          done
```

Two deliberate shapes worth keeping when editing. The workflow uses **no third-party actions** — not even a checkout — because everything happens against the API via the preinstalled `gh`; that is the least supply-chain surface a scheduled workflow can have. And the `permissions:` block grants exactly the two scopes the script exercises; widening it should be treated as a finding needing justification.

## Tuning

`STALE_HOURS` is the one knob. Raise it for repos whose builds legitimately run long between checkpoints; the sweep's prose threshold (factory.md Phase 2) should be kept in agreement — the Action's config is authoritative where both exist, which is why the sweep says "unless the repo's liveness Action config says otherwise." The cron cadence only bounds detection latency; it never changes the criteria.

## Provenance and maintenance

Last verified 2026-07:

- `gh` is preinstalled on GitHub-hosted runners and honors `GH_TOKEN`; re-verify against the runner image docs if a run fails with "gh: not found".
- `date -u -d "<ISO8601>" +%s` is GNU coreutils behavior (ubuntu runners); macOS `date` would need `-j -f`. The template pins `ubuntu-latest` for exactly this reason.
- Issue comments API (`repos/{repo}/issues/{n}/comments`) returns `updated_at` per comment; the heartbeat prefix match relies on the adapter's `factory heartbeat` convention (see `github.md`).
- The built-exemption test runs inside jq — `test("stage: built[[:space:]]*$")` — because `@tsv` escapes a trailing newline to a literal `\n`, which would defeat an end-anchored match in the shell. Re-verify the POSIX bracket support with `jq -rn '"x stage: built\n" | test("stage: built[[:space:]]*$")'` (expects `true`).
