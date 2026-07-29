---
name: self-review
description: >-
  Surface the riskiest parts of the current diff so a reviewer knows where to look hardest — one inline pass on small diffs, parallel review lenses with adversarial synthesis on substantial ones. INVOKE PROACTIVELY whenever a change is finished and about to be handed off, reviewed, or shipped. It points attention — never approves, never fixes. Maintaining the learned-rules store is [[memory]].
argument-hint: "[what to review — defaults to the current diff]"
allowed-tools: Skill, Bash, Read, Agent
---

# Self-review — point attention, don't bless the change

A self-review is not a score. It is not an approval. It is a pointer: *here are the spots a reviewer should look hardest at, and here is why.* Self-scoring rubber-stamps the model's own work — the model that wrote the code is the least reliable judge of whether it's right. That bias is structural, so on any substantial diff the *reading* is taken away from the author entirely: independent lenses that never saw this conversation each read the diff cold, and their findings then have to survive a default-reject bar in this session. Author-blindness is defeated by fresh eyes; finding-inflation is defeated by making every finding earn its place.

## When to apply this

- **Conductor hand-back** (`/sonu:build` Phase 3) — before handing back to the user.
- **Ship pre-PR fix loop** (`/sonu:ship` Phase 1.5) — each pass of the loop runs this skill; the final pass's list seeds the `## Risk / reviewer attention` section of the PR body.
- **Standalone** — any time the user says "self-review", "what should I look at", "what's risky here", or similar; or when a change is finished and about to be handed off. When invoked directly as `/sonu:self-review`, review `$ARGUMENTS` — the text typed after the invocation; if that token appears literally or is empty, review the working tree / current branch per the diff-picker below.

## How to apply this

**1. Read the diff.**

Pick the right diff command for the current state:
- **Uncommitted changes** (working tree dirty): `git diff HEAD` — **plus** `git status --porcelain` to catch untracked files. Brand-new files never added to the index do not appear in `git diff HEAD` at all; read each untracked file directly and include it in the review. A change made entirely of new files (a fresh module plus its tests) is the classic case where `git diff HEAD` shows nothing while there is plenty to review.
- **Just committed** (working tree clean, reviewing what was just committed): `git show HEAD` or `git diff HEAD^ HEAD`
- **Staged only**: `git diff --cached`
- **Whole branch, about to open a PR** (clean multi-commit branch): `git diff origin/<base>...HEAD` (three dots = diff from the merge base). Reviewing only the last commit on a multi-commit branch misses risks in the earlier ones.

If the working tree is clean, there are no untracked files, and there is no recent commit or branch to review, say so and stop — don't invent risks on a clean tree.

**2. Gate on size.**

Count the changed **code** lines — documentation is excluded because prose volume says nothing about code risk. Append `--numstat` to the step-1 diff command, drop doc files by extension, and sum; add the line counts of any untracked code files:

```bash
# Example shown for the branch case — use the same filter on whichever
# diff command step 1 picked. Binary files report "-" and sum as zero.
git diff --numstat origin/main...HEAD \
  | grep -vE '\.(md|mdx|txt|rst|adoc)$' \
  | awk '{ n += $1 + $2 } END { print n+0 }'
```

- **Under ~100 changed code lines** → the inline pass (step 4a). Small diffs don't earn a fan-out's cost.
- **At or above ~100** → the lens fan-out (step 4b).
- **Count uncomputable** (weird state, no git) → fan out. Fail toward more review, never less.
- **Judgment override:** a small diff touching a high-risk surface — a migration, an auth path, a contract other code consumes — may be fanned out anyway. The threshold is a floor on cheapness, not a ceiling on caution.

The fan-out also needs a working subagent facility: if the harness has no subagent tool, or [[model-tiering]]'s ladder gives this session no trustworthy executor tier below it (locate your tier per that skill; if you cannot locate it with confidence, treat that as no tier — its uncertainty asymmetry applies here too), run the inline pass regardless of size. The skill degrades to exactly its single-pass behavior; nothing breaks.

**3. Load learned rules — only if the store exists.**

If `~/.sonu/memory/learned-rules.md` exists, select the top scope-matched active entries per [[memory]]'s read protocol and carry them into the review: injected into every lens prompt on the fan-out path, applied as extra checks on the inline path. If the file or directory does not exist, skip this step silently — do not create the store, do not mention its absence. Maintaining the store is [[memory]]'s job, not this skill's.

**4a. Inline pass (small diffs, or no fan-out available).**

Identify the 3–5 riskiest spots yourself. Focus on the things that, if wrong, would be hardest to catch in a review and most expensive to fix after the fact. Common categories (not a checklist — use judgment):

- **Subtle logic** — branches, edge cases, off-by-ones, conditions that are almost-but-not-quite right.
- **Security-relevant surfaces** — auth, permissions, input sanitization, data exposure, token/secret handling.
- **Data integrity / migration risk** — schema changes, destructive writes, non-reversible transformations.
- **Broad blast radius** — a change to a shared utility, a base class, a widely-imported module, or a config that silently affects many call sites. If the diff changes a data shape other components consume and [[blast-radius]] was not run, that is automatically a top-listed risk.
- **Untested edges** — behavior that the new tests don't cover and that could break in production.
- **Silent behavior change** — the code "works" but now does something subtly different from before, in a way callers may depend on — especially a consumer that catches failures and returns a default, where the break produces no error at all.
- **Interface regressions** — *only when the diff touches user-facing interface files* (components, screens, templates, stylesheets, or interface copy): a keyboard or screen-reader path that no longer completes, a layout that breaks at a supported viewport, or copy that now misstates a consequence. These are risks a reviewer cannot see by reading the diff text, which is exactly why they belong on the list. Apply [[interface-review]]'s domains for the judgment.

Then go to step 6.

**4b. Lens fan-out (substantial diffs).**

Dispatch six independent read-only lenses **in parallel, in one turn**, using the harness's subagent tool — the prompt templates live in `references/lenses.md` (read it when dispatching; the six lenses are correctness, security surfaces, data-integrity & migration, blast-radius & consumer-impact, test-adequacy, and silent-behavior-change — one subagent each). A **seventh interface lens** joins the same batch **only when** the diff touches user-facing interface files (components, screens, templates, stylesheets, or interface copy — judge from the diff's file list); on a diff with no such files it is not dispatched at all. Rules the templates encode, which hold even if you compose prompts yourself:

- **Each lens gets the diff command and the repo — never this conversation.** Independence is the entire value; a lens that knows the author's intent inherits the author's blind spots.
- **Each lens reports findings as `Risk: <what> — <why> [file:line]` lines with a confidence tag**, and must reply "Nothing in my lens." rather than invent a finding to seem useful.
- **Lenses are read-only** — no edits, no writes, no state-changing commands.
- **Lens model:** the cheapest trustworthy executor tier below this session on [[model-tiering]]'s ladder. Dispatching read-only lenses is not the delegation that skill's Section 4 forbids — the lenses gather evidence; every accept/reject decision stays in this session.
- Learned rules from step 3, if any, are appended to every lens prompt as additional checks.

**5. Synthesize in-session — never delegated.**

The lenses report; this session judges. That split is load-bearing: reviewing gathered evidence is exactly the judgment [[model-tiering]] keeps in the session.

- **Default-reject.** A finding survives only if it cites a concrete `file:line` and you can articulate the mechanism by which it goes wrong. Pure style, preference, or "could be cleaner" findings are rejected — that is nit-churn, not risk. When you cannot verify a finding against the actual diff, it dies.
- **Dedup with a bump.** Two or more lenses flagging the same file+issue collapse to one entry — and co-flagging by independent readers is itself a signal, so the merged entry ranks higher.
- **Cross-cut.** The same mistake appearing in N places is one theme with N locations, not N findings.
- Rank what survives and keep the top 3–5. If nothing survives, the diff is low-risk — say so plainly ("This diff is low-risk: X, Y, Z") rather than promoting rejected findings to fill a quota.

**6. Write the list.**

For each item: one line on *what* it is and *why* it's risky. Add `file:line` when it helps a reviewer jump straight there. Keep it scannable — no paragraphs.

Format:
```
Risk: <what> — <why it's risky> [file:line]
```

Worked examples — inline, fan-out synthesis, and the low-risk case — live in `references/examples.md`; read it when unsure what good output looks like.

**7. Feed the store — only if it exists.**

A finding that (a) survived synthesis and (b) is **generalizable** — it would recur in other codebases, rather than being an artifact of this repo's specifics — becomes a candidate learned rule: write it to `~/.sonu/memory/learned-rules.md` following [[memory]]'s write protocol (dedup first; a near-duplicate bumps `hits` instead of adding an entry). When unsure whether it generalizes, don't add — absence is safe, bloat is not. Store absent → skip silently.

**8. Explicitly state what this is NOT.**

End the list with a single line:
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Self-check before you call it done

- Did you actually read the diff, or are you working from memory? Did it include untracked files and, for a branch review, every commit since the merge base?
- Was the size gate computed on code lines with docs excluded — and did an uncomputable count fail open to the fan-out, not closed to the cheap path?
- On the fan-out path: did every lens run without conversation context, and did synthesis — every accept/reject — happen in this session, not in a subagent?
- Is every surviving risk concrete — a specific `file:line` and an articulable failure mechanism — not a vague "this could be better"?
- Did rejected findings stay rejected? A list padded with nit-churn to reach five items is a worse pointer than a list of two real risks.
- If the learned-rules store exists: were its scope-matched rules actually carried into the review, and was every candidate you wrote back dedup-checked and genuinely generalizable?
- Did you avoid inventing risks just to fill the list? If it's low-risk, say so.
- Did you end with the explicit non-approval line?
- Is the list scannable in under 30 seconds?

## Reference files

| File | What it answers |
|---|---|
| `references/lenses.md` | The six lens dispatch prompt templates, plus the conditional interface lens — read when dispatching the step-4b fan-out. |
| `references/examples.md` | Worked output examples (inline pass, fan-out synthesis, low-risk case) — read when unsure of the output shape. |
