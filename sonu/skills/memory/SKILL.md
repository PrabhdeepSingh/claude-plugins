---
name: memory
description: >-
  Own the cross-repo learned-rules store at ~/.sonu/memory/learned-rules.md — the schema, the read protocol other skills use, the dedup-and-capped write protocol, and the /sonu:memory maintenance pass (dedup, decay, evict, graduate rules into skills). INVOKE on /sonu:memory, when asked to see or prune learned rules, or when another skill needs the store's protocols. Performing a review is [[self-review]].
argument-hint: "[show | compact — defaults to compact]"
allowed-tools: Bash, Read, Write, Edit
---

# Memory — lessons that graduate or decay, never pile up

Skills are this plugin's long-term memory: durable, reviewed, versioned. But a lesson learned mid-review — "this class of bug keeps happening" — has no home until someone hand-edits a skill, so it evaporates with the session. This store is the staging area between the two: a single cross-repo file where confirmed, generalizable rules accumulate evidence (`hits`) until they either **graduate** into a real skill through a normal PR or **decay** out. Every mechanism below exists to keep the store small enough to be read in full by a maintainer and cheap enough to be read top-K by a skill — a thousand-entry lessons file that nobody reads is worse than no file at all.

## The store

One file: `~/.sonu/memory/learned-rules.md`. The `~/.sonu/memory/` directory is the extensible container — future kinds of memory get sibling files; do not overload this one.

- **Absent store, reading skill:** skip silently. Readers never create the store and never mention its absence.
- **Absent store, writing:** create the directory and the file with the header below, then append. Only a write creates the store.
- The file's header makes it self-describing, so a maintainer (or a future session) opening it cold knows the rules it lives under:

```markdown
# sonu learned rules
<!-- Schema and protocols: sonu/skills/memory/SKILL.md (the [[memory]] skill).
     Caps: 50 active total, 10 per scope. Graduation: hits >= 3 -> promote into
     a skill via PR, mark graduated. Decay: unused > 90 days -> archived.
     Maintain with /sonu:memory. Do not hand-edit counters casually. -->

## Active

## Archived
```

## Entry schema

One entry per rule, a bullet with indented fields, under `## Active` or `## Archived`:

```markdown
- id: race-in-token-refresh
  rule: Check token-refresh paths for read-then-write races before approving concurrent callers.
  scope: security
  hits: 2
  added: 2031-01-10
  last_used: 2031-03-02
  status: active
```

- `id` — kebab-case slug, unique across the whole file.
- `rule` — ONE imperative line a reviewer can apply directly. If it needs a paragraph, it isn't distilled yet — don't add it.
- `scope` — the sibling skill that would own the rule if it graduated: `code-standards`, `tdd`, `blast-radius`, `safe-migrations`, `infra-standards`, `observability`, `debugging`, `security` (graduates into code-standards' security reference), or `general` when none fits.
- `hits` — how many times this rule was independently re-confirmed. This is the value signal; everything ranks by it.
- `added` / `last_used` — dates (`date +%Y-%m-%d`). `last_used` refreshes on every read-match or duplicate-write.
- `status` — `active` (in play), `graduated` (promoted into a skill; kept one line for the audit trail), `archived` (evicted or decayed; kept under `## Archived`).

## Read protocol (what [[self-review]] and any other reader follows)

1. If the store doesn't exist, skip — silently.
2. Match entries whose `scope` is relevant to the change under review (the surfaces flagged by triage: a migration diff matches `safe-migrations`; everything matches `general`).
3. Take the **top 5 matched active entries**, ranked by `hits` descending, then `last_used` descending. Never load the whole file into a prompt — the cap on what's read is what keeps store growth harmless.
4. Refresh `last_used` on each entry actually carried into the review.

## Write protocol (dedup first, capped always)

A candidate must be **confirmed** (survived whatever verification the writing skill applies) and **generalizable** (would recur in other codebases — not an artifact of one repo's specifics). Unsure → don't write; absence is safe.

1. **Dedup before append.** Grep the store for the candidate's key terms; read the matched entries. Same scope + same substance → do NOT add: increment that entry's `hits`, refresh `last_used`, done. This is the common case and the point — recurrence concentrates evidence on one entry instead of scattering near-duplicates.
2. **Genuinely new** → append under `## Active` with `hits: 1`, both dates today.
3. **Enforce caps on every write:** more than **10 active in that scope** or **50 active total** → move the lowest-value active entries (fewest `hits`, oldest `last_used` as tiebreak) to `## Archived` with `status: archived` until back under both caps. Eviction is a move, not a delete — the archive stays auditable.

## The maintenance pass — `/sonu:memory`

`$ARGUMENTS` — the text typed after the invocation; if that token appears literally or is empty, run `compact`. `show` just prints the active entries grouped by scope with their `hits`, and stops.

`compact` runs four steps, reports what changed, and touches nothing else:

1. **Dedup.** Read all active entries; entries with the same scope and substantially the same rule merge into the strongest phrasing — summed `hits`, earliest `added`, latest `last_used`; the losers move to `## Archived` with a `merged-into: <id>` field appended.
2. **Decay.** Active entries with `last_used` more than **90 days** ago move to `## Archived` (`status: archived`). A rule nobody's work has touched in a quarter has stopped earning its read.
3. **Evict over cap.** Apply the write protocol's caps (10/scope, 50 total) with the same lowest-value-first rule.
4. **Graduate.** Every active entry with `hits >= 3` is a graduation candidate: for each, name the target skill (its `scope` points there) and propose the concrete skill edit — where in that skill's body the rule lands, phrased in the skill's own voice. Graduation itself happens through the normal branch → PR flow in the claude-plugins repo (never a direct edit to an installed copy); once the PR merges, set the entry's `status: graduated` and remove it from `## Active`. If the claude-plugins repo isn't checked out here, list the candidates and the proposed wording for the user to carry over — do not mark anything graduated until the rule actually lives in a skill.

Graduation is the quality-control endpoint of the whole pipeline: a rule either proves itself into the reviewed, versioned skill set, or it decays out. The store is a queue, not an archive of everything ever noticed.

## Self-check before you call it done

- Store reads: top-5 scope-matched only — did any prompt get the whole file? That's the bloat this design exists to prevent.
- Store writes: was dedup actually performed (grep + read), or did a near-duplicate slip in as a new entry?
- Are both caps holding (10 per scope, 50 total) after your write or compact?
- Did every evicted or decayed entry land under `## Archived` rather than being deleted?
- Did anything get marked `graduated` before the rule actually landed in a skill via a merged PR? That's premature — revert the status.
- Is every `rule` line still one imperative sentence a reviewer can apply without context?
