# Adapter — local file store

The dependency-free backend: tickets are Markdown files committed in the repo they describe. No account, no API, no network. The tickets travel with the repo, diff in PRs, and work offline — and because they are just files, the tracker can never be down.

The trade-off to be honest about: there is no server enforcing atomicity, so the claim ordering in section 5 of the spine is what prevents two agents building one ticket. Follow it exactly.

## Layout and file schema

Tickets live in `.sonu/tickets/` at the repo root, one file per ticket, named `NNNN-kebab-slug.md`:

```markdown
---
id: 0001
title: Fix login redirect loop
type: bug
priority: P1
trigger: ready-to-implement
status: open
created: 2031-01-15
---
## Problem

Signing in from an expired session bounces between the login page and the dashboard.

## Scope and non-goals

## Acceptance criteria

## Verification plan

## Discussion

- 2031-01-16 — triage pass — specification added, awaiting human approval.
```

Field rules:

- `id` — the zero-padded four-digit number matching the filename prefix. Never reuse or renumber an id, even after a ticket is closed; a stale reference in a commit message must keep pointing at the right ticket.
- `title` — one line, the same text the slug is derived from.
- `type` — `bug`, `enhancement`, or `documentation`.
- `priority` — `P0` through `P3`, or omitted entirely when the recommendation is rejection.
- `trigger` — `ready-for-spec`, `ready-to-implement`, or `none`. The values deliberately omit the `factory-` prefix that label-based trackers need, because the field name already scopes them. **Only a human sets this to a non-`none` value.**
- `status` — `open`, `in-review`, `done`, or `closed`. This is the one place status is stored rather than derived, because a file has no other state to read (spine section 6).
- `created` — ISO date.

Body sections are the spec. `## Discussion` holds date-stamped bullets, appended newest-last, since a file has no native comment stream.

## The operations

**list queue** — files whose `trigger` matches. Frontmatter is line-oriented, so a grep is enough and needs no YAML parser. Drive it with `find`, never a bare `*.md` glob: in zsh an unmatched glob **aborts the command** with `no matches found`, so an empty ticket store would read as a hard failure instead of an empty queue:

```bash
TRIGGER=ready-to-implement
[ -d .sonu/tickets ] || { echo "STOP: no local ticket store — run /sonu:factory init"; exit 1; }
find .sonu/tickets -maxdepth 1 -name '*.md' | while read -r f; do
  grep -q "^trigger: $TRIGGER$" "$f" || continue
  PRIO=$(grep -m1 '^priority:' "$f" | cut -d' ' -f2)
  printf '%s\t%s\t%s\n' \
    "${PRIO:-unset}" \
    "$(basename "$f")" \
    "$(grep -m1 '^title:' "$f" | cut -d' ' -f2-)"
done | sort -k1,1
```

Four deliberate choices here. The directory check comes first so **"no ticket store" reports differently from "empty queue"** — collapsing those two into silence is how a misconfigured repo reads as "no work to do." The loop pipes `find` output rather than using `-exec grep -l {} +`, whose argument batching needs `ARG_MAX` and hard-fails in restricted or sandboxed environments. There is **no `2>/dev/null` on the `find`**: a permission or path error should be visible, not swallowed into an empty result.

And the missing priority becomes the literal `unset`, which sorts *after* `P0`–`P3` — not an empty field, which would sort **first** and put a ticket recommended for rejection at the top of the dispatch order. A queue whose first row is the one ticket nobody intends to build is worse than an unsorted queue. `unset` is expected on a `ready-for-spec` ticket (nothing has ranked it yet) and is a contradiction on a `ready-to-implement` one — a human authorized building something marked as not-intended work, which the implement route should surface as a blocker rather than silently build.

**list open** — every ticket whose `status:` is not done or closed, trigger or not:

```bash
[ -d .sonu/tickets ] || { echo "STOP: no local ticket store — run /sonu:factory init"; exit 1; }
find .sonu/tickets -maxdepth 1 -name '*.md' | while read -r f; do
  grep -qE '^status: (done|closed)$' "$f" && continue
  printf '%s\t%s\t%s\n' \
    "$(grep -m1 '^priority:' "$f" | cut -d' ' -f2)" "$(basename "$f")" \
    "$(grep -m1 '^title:' "$f" | cut -d' ' -f2-)"
done | sort -k1,1
```

**search** — open *and* closed, for duplicate hunting. Every ticket is a file, so this is a content grep with no state filter:

```bash
TOPIC='login redirect'   # substitute
grep -ril -- "$TOPIC" .sonu/tickets/ 2>/dev/null || echo "(no matches)"
```

**fetch** — read the whole file; it is the ticket, discussion included.

```bash
ID=0001   # substitute
case "$ID" in [0-9][0-9][0-9][0-9]) : ;; *) echo "STOP: id must be four digits"; exit 1 ;; esac
find .sonu/tickets -maxdepth 1 -name "$ID-*.md"
```

The four-digit check is not pedantry: an id of `*` or `../0001` reaching `find -name` matches files the caller never meant to touch.

**claim** — clear the trigger, then verify, then commit. All three steps, in that order:

```bash
ID=0001   # substitute
TRIGGER=ready-to-implement
FILE=$(find .sonu/tickets -maxdepth 1 -name "$ID-*.md" 2>/dev/null | head -1)
[ -n "$FILE" ] || { echo "STOP: no ticket $ID"; exit 1; }
grep -q "^trigger: $TRIGGER$" "$FILE" \
  || { echo "STOP: ticket $ID does not carry $TRIGGER — nothing to claim"; exit 1; }
```

The present-check is the concurrency guard, not a formality: a ticket already at `trigger: none` was claimed by someone else, and treating that as claimable is how the same ticket gets built twice.

Then edit the file's `trigger:` line to `none` with an editing tool (not `sed -i`, whose flag syntax differs between GNU and BSD), re-read to confirm it says `none`, and commit the metadata immediately:

```bash
ID=0001   # substitute
FILE=$(find .sonu/tickets -maxdepth 1 -name "$ID-*.md" 2>/dev/null | head -1)
[ -n "$FILE" ] || { echo "STOP: no ticket $ID"; exit 1; }
grep -q '^trigger: none$' "$FILE" || { echo "STOP: trigger not cleared — do not proceed"; exit 1; }
git add "$FILE" && git commit -m "tickets: claim $ID for implement"
git remote | grep -q . && git push || echo "no remote — local claim only"
```

**update body** — rewrite the body sections below the frontmatter with an editing tool, preserving the reporter's original text under `## Original report`, then commit with `tickets: spec NNNN`.

**Never touch the `trigger:` line while doing it.** That field is the human's authorization, and the spec rewrite is the one moment where an agent has the whole file open and could set it — which would let the pass authorize its own next stage. Verify after every body edit:

```bash
ID=0001   # substitute
case "$ID" in [0-9][0-9][0-9][0-9]) : ;; *) echo "STOP: id must be four digits"; exit 1 ;; esac
FILE=$(find .sonu/tickets -maxdepth 1 -name "$ID-*.md" | head -1)
grep -q '^trigger: none$' "$FILE" \
  || { echo "STOP: trigger changed during a body edit — revert it; only a human sets a trigger"; exit 1; }
```

**comment** — append a bullet under `## Discussion`, date-stamped from the real clock (`date +%Y-%m-%d`), then commit it with a `tickets:` message. Never rewrite or delete an existing bullet; the discussion is an append-only record.

**classify** — edit the `type:` and `priority:` lines. Single-valued fields mean conflicting values cannot coexist. Omit `priority:` entirely for a ticket recommended for rejection rather than inventing a value. Commit with `tickets: classify NNNN`.

**create** — next id is the highest existing prefix plus one, zero-padded to four; the first ticket is `0001`:

```bash
mkdir -p .sonu/tickets
LAST=$(find .sonu/tickets -maxdepth 1 -name '[0-9][0-9][0-9][0-9]-*.md' 2>/dev/null \
  | sed 's|.*/||' | cut -c1-4 | sort -n | tail -1)
[ -n "$LAST" ] || LAST=0000
NEXT=$(printf '%04d' $((10#$LAST + 1)))
echo "next ticket id: $NEXT"
```

The `10#` prefix forces base-10 arithmetic — without it, an id like `0008` is parsed as octal and `0009` fails outright. Write the file with the full frontmatter and the five body headings, `trigger: none`, `status: open`, then commit with `tickets: add $NEXT`.

**close the loop** — a file store has no integration to fire, so the factory sweep does it, and it needs a way to correlate a ticket to a PR. **The branch name is that link**: an implement pass builds on `ticket/NNNN-slug`, and the commit message carries the ticket id. With a GitHub remote:

```bash
ID=0001                        # substitute
SLUG=fix-login-redirect-loop   # substitute
gh pr list --head "ticket/$ID-$SLUG" --state all --json number,state,mergedAt
```

Without a GitHub remote, check whether the branch has landed in the default branch instead:

```bash
ID=0001                        # substitute
SLUG=fix-login-redirect-loop   # substitute
BASE=$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|origin/||')
[ -n "$BASE" ] || { echo "STOP: cannot determine the default branch — set BASE explicitly"; exit 1; }
git branch --merged "$BASE" | grep -qF "ticket/$ID-$SLUG" \
  && echo "merged into $BASE" || echo "not merged yet"
```

**Status transitions**, so the field is never guessed at: `open` on creation and through triage; the sweep sets **`in-review`** once it finds an open PR for the ticket's branch, **`done`** once that PR is merged (or the branch has landed), and `closed` when a human closes the ticket without merging anything. Each flip is one `tickets: ...` commit. Nothing else writes this field — a status nobody owns is the stale bookkeeping the derived-status rule exists to avoid.

## The metadata-commit rule

Every ticket-file edit — claim, spec rewrite, classification, status flip — is committed **immediately, on its own, with a `tickets:` message prefix**, and pushed when a remote exists.

Why each half matters. *Immediately*, because an uncommitted claim is invisible to a second agent, which is the exact race the claim is supposed to prevent — and pushing is what makes it visible to a second *machine*. *On its own*, because ticket bookkeeping mixed into a code commit makes the diff a human reviews noisier for no benefit; during an implement pass the claim commit lands on the default branch before the worktree is even created, so the code diff stays clean.

This is a deliberate carve-out from the rule that workflows never commit: it covers tracker **metadata only, never source code**. Remote trackers get the same durability server-side for free; this is how the file store earns it.

## Provenance and maintenance

Last verified 2026-07:

- `sed -i` is deliberately avoided — GNU requires `-i`, BSD/macOS requires `-i ''`, and the mismatch fails silently in one direction. Use an editing tool for in-place frontmatter edits.
- `$((10#$LAST + 1))` is the portable way to force base 10 in both bash and zsh; verify with `LAST=0009; echo $((10#$LAST + 1))`.
- `grep -l` with no matches exits non-zero and prints nothing, hence the `2>/dev/null` and the `|| true`-style guards at call sites.
