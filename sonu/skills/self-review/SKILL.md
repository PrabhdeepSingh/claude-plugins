---
name: self-review
description: >-
  Surface the riskiest parts of the current diff so a reviewer knows where to look hardest — one inline pass on small diffs, parallel review lenses with adversarial synthesis on substantial ones. INVOKE PROACTIVELY whenever a change is finished and about to be handed off, reviewed, or shipped. It points attention — never approves, never fixes.
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
BASE=main   # substitute the base branch step 1 picked — a wrong base must fail LOUDLY,
            # because a failed git diff would otherwise print a silent, computed-looking 0
git rev-parse --verify "origin/$BASE" >/dev/null 2>&1 \
  || { echo "COUNT UNCOMPUTABLE — origin/$BASE not found; fan out"; exit 1; }
git diff --numstat "origin/$BASE...HEAD" \
  | grep -vE '\.(md|mdx|txt|rst|adoc)$' \
  | awk '{ n += $1 + $2 } END { print n+0 }'
```

- **Under ~100 changed code lines** → the inline pass (step 3a). Small diffs don't earn a fan-out's cost.
- **At or above ~100** → the lens fan-out (step 3b).
- **Count uncomputable** (weird state, no git) → fan out. Fail toward more review, never less.
- **Judgment override:** a small diff touching a high-risk surface may be fanned out anyway. Checkable markers of high-risk: it introduces or modifies branching logic, crosses a module or service boundary, asserts a property the type system cannot verify (thread safety, idempotence, ordering, an invariant), or has an irreversible blast radius — a migration, an auth path, a contract other code consumes. The threshold is a floor on cheapness, not a ceiling on caution. And in a repo whose product *is* its documents (a skills or plugin repo, a docs site), count all changed lines — the doc-exclusion above otherwise makes the fan-out unreachable exactly where the diffs are largest.

The fan-out also needs a working subagent facility: if the harness has no subagent tool, or [[model-tiering]]'s ladder gives this session no trustworthy executor tier below it (locate your tier per that skill; if you cannot locate it with confidence, treat that as no tier — its uncertainty asymmetry applies here too), run the inline pass regardless of size. The skill degrades to exactly its single-pass behavior; nothing breaks.

**3a. Inline pass (small diffs, or no fan-out available).**

Identify the 3–5 riskiest spots yourself. Focus on the things that, if wrong, would be hardest to catch in a review and most expensive to fix after the fact. Common categories (not a checklist — use judgment):

- **Subtle logic** — branches, edge cases, off-by-ones, conditions that are almost-but-not-quite right.
- **Security-relevant surfaces** — auth, permissions, input sanitization, data exposure, token/secret handling.
- **Data integrity / migration risk** — schema changes, destructive writes, non-reversible transformations.
- **Broad blast radius** — a change to a shared utility, a base class, a widely-imported module, or a config that silently affects many call sites. If the diff changes a data shape other components consume, consumer impact is automatically a top-listed risk — cite [[blast-radius]]'s consumer enumeration when the change ran it, and flag its absence when it didn't.
- **Untested edges** — behavior that the new tests don't cover and that could break in production.
- **Silent behavior change** — the code "works" but now does something subtly different from before, in a way callers may depend on — especially a consumer that catches failures and returns a default, where the break produces no error at all.
- **Duplicated judgment** — two expressions that independently decide the same thing (a filter and the transform it protects, a cap and the accumulator it bounds) and can drift apart; the failure ships as records that pass one and fail the other, with no error anywhere.
- **Interface regressions** — *only when the diff touches user-facing interface files* (components, screens, templates, stylesheets, or interface copy): a keyboard or screen-reader path that no longer completes, a layout that breaks at a supported viewport, or copy that now misstates a consequence. These are risks a reviewer cannot see by reading the diff text, which is exactly why they belong on the list. Apply the owning domain skills for the judgment ([[accessibility]], [[layout]], [[typography]], [[colors]], [[ui-polish]], [[ux-writing]]) — [[interface-review]] is orchestration only and carries no domain rules.

Then go to step 5.

**3b. Lens fan-out (substantial diffs).**

Dispatch independent read-only lenses **in parallel, in one turn**, using the harness's subagent tool — one subagent per dispatched lens, prompt templates in `references/lenses.md` (read it when dispatching). Two tiers decide who goes out:

- **The code lenses — correctness, test-adequacy, silent-behavior-change — dispatch whenever the diff changes executable code**, meaning changed lines in non-doc files. Any code diff can carry a logic error, an untested path, or a changed default, so these three have no narrower precondition than "there is code here." One carve-out, because it decides the common case: a non-doc file changed **only** in metadata — a version string, a lockfile hash, a copyright year — is not code for this purpose. Nearly every release rides a version bump alongside its real change, and counting those two lines as code would make the prose path below unreachable in exactly the repos that need it. **Read this off the diff, not off step 2's number:** in a docs-product repo step 2 deliberately counts *all* changed lines including prose, and that count answers "is this diff big enough to fan out," never "is there code here." Confusing the two dispatches all three lenses against a pure prose diff, which is exactly the waste this gate exists to stop.
- **The domain lenses — security, data-integrity, blast-radius, interface — dispatch only when the diff contains their domain**, per the four conditions below.

**Why gating is safe:** a lens whose domain is absent from the diff is *structurally* empty, not luckily empty — a security lens cannot find an injection sink in a file that has no sink, and a consumer lens cannot find a broken consumer of a contract the diff never changed. This is the same reasoning that declines to load [[safe-migrations]] for a stylesheet change.

**The no-code case — a diff that is entirely prose.** You reach this fan-out on a prose-only diff through step 2's docs-product rule (a skills repo, plugin repo, or docs site counts all its changed lines). Here the code lenses have no domain either: a lens hunting off-by-ones and unhandled boundaries finds nothing in prose, and test-adequacy is empty in any repo with no test suite. The lens prompts in `references/lenses.md` are written for code and would send a reader hunting off-by-ones through paragraphs, so **no lens goes out on its code prompt alone** — every prose-path dispatch prepends that file's prose frame, which replaces the code framing and tells the lens it is reading instructions that govern behavior. Ask **what the prose governs** — because in a repo whose product is its documents, a skill or command file *is* the behavior:

Apply **the same four domain conditions below, unchanged** — reading "the diff" as the prose *and the behavior it governs*, because in a repo whose product is its documents a skill or command file **is** the behavior. Prose that alters what another component does is a change "read or addressed by code outside the diff"; prose that alters when a security check runs moves a trust boundary as surely as a middleware edit does. **Do not invent a second, looser test for the prose case** — a domain with two dispatch tests has two answers, and this skill has already had that bug.

The three code lenses come back only where the prose carries their domain: **correctness** when it states rules, thresholds, or branches an executor follows; **silent-behavior-change** when it changes what a component does without announcing it; **test-adequacy** only when the repo has a suite whose adequacy this prose changes — usually it does not.

If nothing above fires — a typo pass, a wording cleanup, a README polish — say the diff is low-risk and fall back to the inline pass (3a) rather than dispatching anything at all. Seven agents on a doc edit is the failure this gate exists to prevent.

The four domain conditions:

- **Security — dispatch when:** the diff **moves or weakens a trust boundary** — untrusted input reaching a sink, a control that protects a boundary being added/changed/removed, or a secret crossing one. *That sentence is the test.* The list below is examples that speed up the common case; it is **never the whole test**, because an enumeration of attack surfaces is never finished and treating a closed list as the test turns every unlisted class into a silent skip. **A change that fits the principle but matches no example still dispatches.** Examples: auth or authorization, including a route or render guard enforced only on the client; an API, route, or endpoint handler; SQL or any constructed query, command, or path; deserialization or dynamic evaluation of untrusted data; middleware; crypto, secret, token, or credential handling; payments; env vars or config that gates runtime behavior or names an outbound destination; session, header, CORS, `postMessage`, or frame-isolation handling; file I/O, uploads, or downloads; subprocess or shell invocation; outbound requests to user-influenced destinations; dependency manifests, lockfiles, or a newly included third-party script or asset; untrusted input interpolated into HTML, a URL, a navigation target, or a log line (`dangerouslySetInnerHTML`, `innerHTML`, an `href`/`src` built from input, a redirect); a regex applied to user input; a recursive merge or dynamic property assignment from user-controlled data; or model/tool output reaching an executable sink. **This condition is the canonical security-surface test** — `/sonu:ship`'s effort-mode table cites it rather than keeping a second copy, so the two gates can never disagree about the same diff.
- **Data integrity — dispatch when:** the diff touches migrations, schema, serialization or deserialization, backfills, bulk or destructive writes, or **persisted state that other code or a later release reads back** — not scratch state written and read entirely inside the diff.
- **Blast radius — dispatch when:** the diff changes something **read or addressed by code outside the diff** — a return shape or type, response body, serialized payload, DB column, log or telemetry field, event or queue message, config key or env var, parsed CLI/stdout output, or a published identifier (route, tool name, command, exported symbol) **that has consumers outside this change**. Skip purely internal changes and strictly additive optional fields; a newly-added export nothing calls yet is not a contract change. This is the same test [[blast-radius]] itself states — which this skill previously did not honour.
- **Interface — dispatch when:** the diff touches user-facing interface files — components, screens, templates, stylesheets, or interface copy. Judge from the diff's file list.

**You may skip a domain lens only when you can state, in one sentence, the specific reason its domain is absent** — "nothing here touches persisted state," not "probably fine." If you cannot write that sentence, dispatch. Do not treat "unsure" as a feeling to introspect on: it is simply the absence of an articulable reason, and everything above about saving cost is never one. This is step 4's default-reject bar pointed the other way, and it exists because a gate written to save money will be read by someone who wants to save money.

Rules the templates encode, which hold even if you compose prompts yourself:

- **Each lens gets the diff command and the repo — never this conversation.** Independence is the entire value; a lens that knows the author's intent inherits the author's blind spots.
- **Each lens prompt is self-contained.** Criteria live in the lens block. Never tell a subagent to `Skill(…)` load domain skills or to read files from the plugin install — the shared frame only gives the customer repo root, and many harnesses give subagents no Skill tool. The interface lens inlines its six-domain checklist for that reason; it does not orchestrate [[interface-review]].
- **Each lens reports findings as `Risk: <what> — <why> [file:line]` lines with a confidence tag**, and must reply "Nothing in my lens." rather than invent a finding to seem useful.
- **Lenses are read-only** — no edits, no writes, no state-changing commands.
- **Lens model:** the cheapest trustworthy executor tier below this session on [[model-tiering]]'s ladder. Dispatching read-only lenses is not the delegation that skill's Section 4 forbids — the lenses gather evidence; every accept/reject decision stays in this session.

**4. Synthesize in-session — never delegated.**

The lenses report; this session judges. That split is load-bearing: reviewing gathered evidence is exactly the judgment [[model-tiering]] keeps in the session.

- **Default-reject.** A finding survives only if it cites a concrete `file:line` and you can articulate the mechanism by which it goes wrong. Pure style, preference, or "could be cleaner" findings are rejected — that is nit-churn, not risk. When you cannot verify a finding against the actual diff, it dies.
- **Dedup with a bump.** Two or more lenses flagging the same file+issue collapse to one entry — and co-flagging by independent readers is itself a signal, so the merged entry ranks higher.
- **Cross-cut.** The same mistake appearing in N places is one theme with N locations, not N findings.
- Rank what survives and keep the top 3–5 — **by leverage, structural problems first**: one structural risk above ten nits is the correct order, because if there is one structural problem and ten nits, the structural problem *is* the review.
- **Watch for doubt theater.** Across two or more passes where lenses surfaced substantive findings, zero findings accepted as real means you are validating your own work, not reviewing it — stop and say so in the hand-off instead of emitting a clean-looking list.
- If nothing survives, the diff is low-risk — say so plainly ("This diff is low-risk: X, Y, Z") rather than promoting rejected findings to fill a quota.

**5. Write the list.**

For each item: one line on *what* it is and *why* it's risky. Add `file:line` when it helps a reviewer jump straight there. Keep it scannable — no paragraphs.

Format:
```
Risk: <what> — <why it's risky> [file:line]
```

**On the fan-out path, end with one line naming what every gated lens did** — each of the four domain lenses, plus the three code lenses whenever the prose path gated them too. Give either the clause that matched (so it was dispatched) or state that none did, e.g. `Domain lenses: interface (stylesheets + components) · security, data-integrity, blast-radius — no clause matched.` Report both directions, not just the skips: an over-firing gate quietly eats the saving, an under-firing one quietly eats a finding, and a guard that only makes skips visible catches only the second. A skip nobody can see is indistinguishable from a coverage gap.

Worked examples — inline, fan-out synthesis, and the low-risk case — live in `references/examples.md`; read it when unsure what good output looks like.

**6. Explicitly state what this is NOT.**

End the list with a single line:
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Self-check before you call it done

- Did you actually read the diff, or are you working from memory? Did it include untracked files and, for a branch review, every commit since the merge base?
- Was the size gate computed on code lines with docs excluded — and did an uncomputable count fail open to the fan-out, not closed to the cheap path?
- On the fan-out path: did every lens run without conversation context, and did synthesis — every accept/reject — happen in this session, not in a subagent?
- Did the output name, for every gated lens — the four domain lenses always, the code lenses too when the prose path gated them — the clause that matched or that none did, so both an over-firing and an under-firing gate are visible?
- Is every surviving risk concrete — a specific `file:line` and an articulable failure mechanism — not a vague "this could be better"?
- Did rejected findings stay rejected? A list padded with nit-churn to reach five items is a worse pointer than a list of two real risks.
- Did you avoid inventing risks just to fill the list? If it's low-risk, say so.
- Did you end with the explicit non-approval line?
- Is the list scannable in under 30 seconds?

## Reference files

| File | What it answers |
|---|---|
| `references/lenses.md` | The dispatch prompt templates for both tiers — the three code lenses and the four domain lenses — read when dispatching the step-3b fan-out. The conditions themselves live in step 3b, not here. |
| `references/examples.md` | Worked output examples (inline pass, fan-out synthesis, low-risk case) — read when unsure of the output shape. |
