---
description: Prepare a release of the sonu plugin — decide the semver bump, sync the version and plugin description across all five homes, run /validate, then hand to /sonu:ship. Only meaningful inside the claude-plugins repo; in any other repo, say so and stop.
argument-hint: "[major|minor|patch to force the bump, tag for post-merge tagging, or empty to infer]"
allowed-tools: Bash, Read, Edit, Skill
---

# /release — the five-home sync, done right every time

**Why this exists:** installed copies of the plugin are pinned — a change on `main` reaches nobody until the version is bumped and users run `/plugin marketplace update`. And the plugin's description lives in **five files** that must move together. Both halves of that have failed before (two releases of marketplace drift; one behavior fix that never got a bump). This command is the procedure.

If `$ARGUMENTS` is `tag`, skip straight to Phase 5 (the PR already merged). Otherwise run Phases 1–4 in order.

**Guard first.** This command releases the `claude-plugins` repo itself:
```bash
[ -f .claude-plugin/marketplace.json ] && [ -d sonu/skills ] \
  && echo "OK: claude-plugins repo" || echo "STOP: not the claude-plugins repo"
```

## Phase 1 — What changed, and what bump does it earn

```bash
git log $(git describe --tags --abbrev=0 2>/dev/null || echo origin/main)..HEAD --oneline
git diff origin/main --stat 2>/dev/null | tail -5
CURRENT=$(python3 -c "import json; print(json.load(open('sonu/.claude-plugin/plugin.json'))['version'])")
echo "current version: $CURRENT"
```

Pick the bump (from `$ARGUMENTS` if given — the text typed after the command; if that token appears literally or is empty, infer from the diff):

| Change | Bump |
|---|---|
| New skill or command added, or behavior of an existing one meaningfully changed | **minor** |
| Fix/clarification to existing component, docs, manifests | **patch** |
| Removed or renamed a component, or changed how users invoke things | **major** |

**Any user-visible change gets a bump.** "It's just a wording fix in ship.md" is a behavior change for the model executing it — that exact reasoning skipped a bump once and left every installed copy running the buggy text.

## Phase 2 — The five homes, in order

Update each; the version appears in homes 1–2, the plugin description in homes 1–5 (sync the description text **only if the component set or a component's pitch changed**):

| # | File | What to update |
|---|---|---|
| 1 | `sonu/.claude-plugin/plugin.json` | `version` (always), `description` + `keywords` (if components changed) |
| 2 | `sonu/.cursor-plugin/plugin.json` | Mirror #1 **byte-for-byte** — the pair must stay identical |
| 3 | `.claude-plugin/marketplace.json` | plugin `description` + `keywords` (if components changed) |
| 4 | `.cursor-plugin/marketplace.json` | Same as #3 |
| 5 | `README.md` | Component sections, install-section command roster, the Layout diagram |

Description conventions: every command and skill is *named* in the marketplace description (that's what `/validate` check 2 enforces); keep it scannable, not a mega-paragraph.

## Phase 3 — Validate

Run `/validate` (the full battery: manifest sync, frontmatter, shell fences, house-rule scans, cross-references, README inventory). Fix every FAIL before proceeding. Do not hand-wave a FAIL to "fix after the release" — the release IS the sync point.

## Phase 4 — Hand back for shipping

This command never commits or merges (house rule 5 — everything goes through a PR), and `/sonu:ship` is deliberately user-only (`disable-model-invocation` — a model must never trigger a merge on its own). So **stop the turn here** with:

> **Release prepared and validated.** Review the five-home diff, then run `/sonu:ship` to open and merge the PR. Come back with `/release tag` once it's merged.

## Phase 5 — Tag (run after the user's PR has merged)

```bash
# Tag the release so versions are traceable without archaeology. BASE is derived,
# not hardcoded (matches ship.md/validate.md's convention) — and every step is
# chained with && so a failed checkout/pull can't fall through into tagging the
# wrong commit.
BASE=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
git checkout "$BASE" && git pull && \
NEW=$(python3 -c "import json; print(json.load(open('sonu/.claude-plugin/plugin.json'))['version'])") && \
git tag "v$NEW" && git push origin "v$NEW"
```

Then remind the user: installed copies pick this up on their next `/plugin marketplace update prabhdeep-tools` — nothing propagates automatically.

## When NOT to use this

- Change isn't user-visible at all (pure repo housekeeping like `.gitignore`)? Still ship via PR, but no bump needed — say so explicitly in the PR body.
- Not sure the change is even right yet? That's `/sonu:build` → review → then come back here.
