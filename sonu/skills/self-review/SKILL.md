---
name: self-review
description: >-
  Surface the riskiest parts of the current diff so a reviewer knows where to look hardest — one inline pass on small diffs, one cold read on a cheaper model tier with in-session synthesis on substantial ones. INVOKE PROACTIVELY whenever a change is finished and about to be handed off, reviewed, or shipped. It points attention — never approves, never fixes.
argument-hint: "[what to review — defaults to the current diff]"
allowed-tools: Skill, Bash, Read, Agent
---

# Self-review — point attention, don't bless the change

A self-review is not a score. It is not an approval. It is a pointer: *here are the spots a reviewer should look hardest at, and here is why.* Self-scoring rubber-stamps the model's own work — the model that wrote the code is the least reliable judge of whether it's right. That bias is structural, so on any substantial diff the *reading* is taken away from the author entirely: an independent reader on a cheaper model tier, which never saw this conversation, reads the diff cold against every checklist the diff's contents call for, and its findings then have to survive a default-reject bar in this session. Author-blindness is defeated by fresh eyes; finding-inflation is defeated by making every finding earn its place — and the read itself is priced by what the diff needs, not by how many checklists exist.

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

**Compute this count before anything is dispatched** — it decides the path *and* the reader count in step 3b, so a count taken "for the record" after the readers are out has decided nothing. Count the changed **production code** lines: documentation is excluded because prose volume says nothing about code risk, and **test files are excluded** because test lines decide nothing about which domain needs a read — they are still in the diff and the reader still reads them; they just don't inflate the gate (on the diff that motivated this rule, tests and fixtures doubled the count). Deletions still count: a deleted component or export is a consumer risk. Append `--numstat` to the step-1 diff command, drop doc files by extension and test files by path, and sum; add the line counts of any untracked production code files:

```bash
# Example shown for the branch case — use the same filter on whichever
# diff command step 1 picked. Binary files report "-" and sum as zero.
BASE=main   # substitute the base branch step 1 picked — a wrong base must fail LOUDLY,
            # because a failed git diff would otherwise print a silent, computed-looking 0
git rev-parse --verify "origin/$BASE" >/dev/null 2>&1 \
  || { echo "COUNT UNCOMPUTABLE — origin/$BASE not found; take the cold read"; exit 1; }
git diff --numstat "origin/$BASE...HEAD" \
  | grep -vE '\.(md|mdx|txt|rst|adoc)$' \
  | grep -vE '(^|/)(test|tests|__tests__|spec)/|\.(test|spec)\.[a-z]+$|_test\.go$|(^|/)test_[^/]*\.py$' \
  | awk '{ n += $1 + $2 } END { print n+0 }'
```

- **Under ~100 changed production code lines** → the inline pass (step 3a). Small diffs don't earn a subagent's cost.
- **At or above ~100** → the cold read (step 3b): one reader.
- **Above ~500, with at least one domain checklist live** → the cold read with two readers (step 3b says how they split; a diff that size with no domain matched stays at one reader). This is the only place that number lives.
- **Count uncomputable** (weird state, no git) → the cold read. Fail toward more review, never less.
- **Judgment override:** a small diff touching a high-risk surface may get the cold read anyway. Checkable markers of high-risk: it introduces or modifies branching logic, crosses a module or service boundary, asserts a property the type system cannot verify (thread safety, idempotence, ordering, an invariant), or has an irreversible blast radius — a migration, an auth path, a contract other code consumes. The threshold is a floor on cheapness, not a ceiling on caution. And in a repo whose product *is* its documents (a skills or plugin repo, a docs site), count all changed lines — prose and its examples alike — because the doc-exclusion above otherwise makes the cold read unreachable exactly where the diffs are largest.

The cold read also needs a working subagent facility: if the harness has no subagent tool, or [[model-tiering]]'s ladder gives this session no trustworthy executor tier below it (locate your tier per that skill; if you cannot locate it with confidence, treat that as no tier — its uncertainty asymmetry applies here too), run the inline pass regardless of size. The skill degrades to exactly its single-pass behavior; nothing breaks.

**3a. Inline pass (small diffs, or no cold read available).**

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

**3b. Cold read (substantial diffs).**

Dispatch **one** independent read-only reader using the harness's subagent tool, its prompt composed from the checklist blocks in `references/lenses.md` (read it when dispatching): the code checklists when the diff has code, plus every domain checklist whose condition below matches. **One reader with N checklists costs one read of the diff; N readers cost N** — the same arithmetic that merged the three code checklists into one block also puts the domain checklists in the same prompt, because a Sonnet-class reader holding five short checklists is still cheaper and faster than five readers each holding one. The single split: when step 2's count crosses its two-reader threshold (the number lives there, nowhere else), dispatch **two** readers in the same turn — one carrying the code block, one carrying every matched domain block — so a large diff isn't throttled by one reader's output cap. If no domain condition matched, there is nothing to split: one reader, whatever the size — a reader carrying no checklist is never dispatched. **Never more than two readers, whatever the size and however many domains are live.** A "lens" below is a checklist a reader carries, not a subagent of its own. Two conditions decide what goes into the prompt:

- **The code checklists — correctness, test-adequacy, and silent-behavior-change — are carried whenever the diff changes executable code**, meaning changed lines in non-doc files. Any code diff can carry a logic error, an untested path, or a changed default, so this block has no narrower precondition than "there is code here." One carve-out, because it decides the common case: a non-doc file changed **only** in metadata — a version string, a lockfile hash, a copyright year — is not code for this purpose. Nearly every release rides a version bump alongside its real change, and counting those two lines as code would make the prose path below unreachable in exactly the repos that need it. **Read this off the diff, not off step 2's number:** in a docs-product repo step 2 deliberately counts *all* changed lines including prose, and that count answers "is this diff big enough for the cold read," never "is there code here." Confusing the two carries the code checklists against a pure prose diff, which is exactly the waste this gate exists to stop.
- **The domain lenses — security, data-integrity, blast-radius, interface — are carried only when the diff contains their domain**, per the four conditions below. Each is a paragraph in the reader's prompt, so including one costs a paragraph, not a subagent — the gate still matters, because a checklist with no domain in the diff produces invented findings for step 4 to reject, and the reader's attention is spent on it.

**Why gating is safe:** a lens whose domain is absent from the diff is *structurally* empty, not luckily empty — a security checklist cannot find an injection sink in a file that has no sink, and a consumer checklist cannot find a broken consumer of a contract the diff never changed. This is the same reasoning that declines to load [[safe-migrations]] for a stylesheet change.

**The no-code case — a diff that is entirely prose.** You reach the cold read on a prose-only diff through step 2's docs-product rule (a skills repo, plugin repo, or docs site counts all its changed lines). Here the code checklists have no domain either: a reader hunting off-by-ones and unhandled boundaries finds nothing in prose, and test-adequacy is empty in any repo with no test suite. The checklist blocks in `references/lenses.md` are written for code and would send a reader hunting off-by-ones through paragraphs, so **no reader goes out on the code framing alone** — every prose-path prompt carries that file's prose frame, which replaces the code framing and tells the reader it is reading instructions that govern behavior. Ask **what the prose governs** — because in a repo whose product is its documents, a skill or command file *is* the behavior:

Apply **the same four domain conditions below, unchanged** — reading "the diff" as the prose *and the behavior it governs*, because in a repo whose product is its documents a skill or command file **is** the behavior. Prose that alters what another component does is a change "read or addressed by code outside the diff"; prose that alters when a security check runs moves a trust boundary as surely as a middleware edit does. **Do not invent a second, looser test for the prose case** — a domain with two dispatch tests has two answers, and this skill has already had that bug.

The code checklists are carried only where the prose carries at least one of their domains — **correctness** when it states rules, thresholds, or branches an executor follows; **silent-behavior-change** when it changes what a component does without announcing it; **test-adequacy** only when the repo has a suite whose adequacy this prose changes (usually it does not) — and the block keeps only the checklist paragraphs whose domain is present, dropping the rest.

If nothing above fires — a typo pass, a wording cleanup, a README polish — say the diff is low-risk and fall back to the inline pass (3a) rather than dispatching anything at all. A subagent on a doc edit is the failure this gate exists to prevent.

The four domain conditions:

- **Security — dispatch when:** the diff **moves or weakens a trust boundary** — untrusted input reaching a sink, a control that protects a boundary being added/changed/removed, or a secret crossing one. *That sentence is the test.* The list below is examples that speed up the common case; it is **never the whole test**, because an enumeration of attack surfaces is never finished and treating a closed list as the test turns every unlisted class into a silent skip. **A change that fits the principle but matches no example still dispatches.** Examples: auth or authorization, including a route or render guard enforced only on the client; an API, route, or endpoint handler; SQL or any constructed query, command, or path; deserialization or dynamic evaluation of untrusted data; middleware; crypto, secret, token, or credential handling; payments; env vars or config that gates runtime behavior or names an outbound destination; session, header, CORS, `postMessage`, or frame-isolation handling; file I/O, uploads, or downloads; subprocess or shell invocation; outbound requests to user-influenced destinations; dependency manifests, lockfiles, or a newly included third-party script or asset; untrusted input interpolated into HTML, a URL, a navigation target, or a log line (`dangerouslySetInnerHTML`, `innerHTML`, an `href`/`src` built from input, a redirect); a regex applied to user input; a recursive merge or dynamic property assignment from user-controlled data; or model/tool output reaching an executable sink. **This condition is the canonical security-surface test** — `/sonu:ship`'s effort-mode table cites it rather than keeping a second copy, so the two gates can never disagree about the same diff.
- **Data integrity — dispatch when:** the diff touches migrations, schema, serialization or deserialization, backfills, bulk or destructive writes, or **persisted state that other code or a later release reads back** — not scratch state written and read entirely inside the diff.
- **Blast radius — dispatch when:** the diff changes something **read or addressed by code outside the diff** — a return shape or type, response body, serialized payload, DB column, log or telemetry field, event or queue message, config key or env var, parsed CLI/stdout output, or a published identifier (route, tool name, command, exported symbol) **that has consumers outside this change**. Skip purely internal changes and strictly additive optional fields; a newly-added export nothing calls yet is not a contract change. This is the same test [[blast-radius]] itself states — which this skill previously did not honour.
- **Interface — dispatch when:** the diff touches user-facing interface files — components, screens, templates, stylesheets, or interface copy. Judge from the diff's file list.

**You may leave a domain lens out only when you can state, in one sentence, the specific reason its domain is absent** — "nothing here touches persisted state," not "probably fine." If you cannot write that sentence, carry it — it costs a paragraph. Do not treat "unsure" as a feeling to introspect on: it is simply the absence of an articulable reason. This is step 4's default-reject bar pointed the other way.

Rules the templates encode, which hold even if you compose the prompt yourself:

- **The reader gets the diff command and the repo — never this conversation.** Independence is the entire value; a reader that knows the author's intent inherits the author's blind spots.
- **The reader prompt is self-contained.** Criteria live in the checklist blocks. Never tell a subagent to `Skill(…)` load domain skills or to read files from the plugin install — the shared frame only gives the customer repo root, and many harnesses give subagents no Skill tool. The interface block inlines its six-domain checklist for that reason; it does not orchestrate [[interface-review]].
- **The reader reports findings as `Risk (<tag>): <what> — <why> [file:line]` lines with a confidence tag** — the tag names the checklist that caught it and is one of exactly seven: `SECURITY`, `DATA INTEGRITY`, `CONSUMERS`, `INTERFACE`, `CODE/correctness`, `CODE/tests`, `CODE/silent-change` — and must reply "Nothing in my lens." rather than invent a finding to seem useful.
- **The reader reads on a budget and reports under a cap** — both live in the shared frame: bounded reads of a flagged hunk's enclosing block or its one-hop callee, no whole tracked files (untracked files *are* the diff and are read in full), and repo search only in the two places the frame allows — the consumers checklist hunting consumers, the tests checklist looking up a changed function's tests — each capped; at most five findings **per checklist carried**, high or medium confidence only, then one `Withheld (<tag>): N more.` line per checklist that hit the cap. A reader that reads whole files or emits a page of findings is spending the tokens this gate exists to save — and a withheld line is a signal to look at that checklist's domain yourself, never a reason to re-dispatch.
- **Readers are read-only** — no edits, no writes, no state-changing commands.
- **Reader model — set explicitly on the Agent call, every time**, to the cheapest trustworthy executor tier below this session. The mechanic and its why live in [[model-tiering]] Section 5; the literal call shape lives in `references/lenses.md`'s dispatch mechanics. Dispatching a read-only reader is not the delegation that skill's Section 4 forbids — the reader gathers evidence; every accept/reject decision stays in this session.

**4. Synthesize in-session — never delegated.**

The reader reports; this session judges. That split is load-bearing: reviewing gathered evidence is exactly the judgment [[model-tiering]] keeps in the session.

- **Default-reject.** A finding survives only if it cites a concrete `file:line` and you can articulate the mechanism by which it goes wrong. Pure style, preference, or "could be cleaner" findings are rejected — that is nit-churn, not risk. When you cannot verify a finding against the actual diff, it dies.
- **Dedup with a bump.** Two or more checklists flagging the same file+issue collapse to one entry — and a co-flag is itself a signal (two different failure mechanisms point at the same line), so the merged entry ranks higher.
- **Cross-cut.** The same mistake appearing in N places is one theme with N locations, not N findings.
- Rank what survives and keep the top 3–5 — **by leverage, structural problems first**: one structural risk above ten nits is the correct order, because if there is one structural problem and ten nits, the structural problem *is* the review.
- **Watch for doubt theater.** Across two or more passes where the reader surfaced substantive findings, zero findings accepted as real means you are validating your own work, not reviewing it — stop and say so in the hand-off instead of emitting a clean-looking list.
- If nothing survives, the diff is low-risk — say so plainly ("This diff is low-risk: X, Y, Z") rather than promoting rejected findings to fill a quota.

**5. Write the list.**

For each item: one line on *what* it is and *why* it's risky. Add `file:line` when it helps a reviewer jump straight there. Keep it scannable — no paragraphs.

Format:
```
Risk: <what> — <why it's risky> [file:line]
```

**On the cold-read path, end with a dispatch line naming what every gated domain lens did** (a lens here is a checklist the reader carried), followed by the reader count and tier — and, on the prose path only, a second `Code checklists:` line saying which of the three were carried and why the rest were dropped. Give either the clause that matched (so the checklist was carried) or state that none did, e.g. `Domain lenses: interface (stylesheets + components) · security, data-integrity, blast-radius — no clause matched. Readers: 1 (sonnet)`. The `Domain lenses:` prefix and its per-domain shape are read by `/sonu:ship` to set its security verdict — keep them exactly. Report both directions, not just the skips: an over-firing gate quietly eats the reader's attention, an under-firing one quietly eats a finding, and a guard that only makes skips visible catches only the second. A skip nobody can see is indistinguishable from a coverage gap.

Worked examples — inline, cold-read synthesis, and the low-risk case — live in `references/examples.md`; read it when unsure what good output looks like.

**6. Explicitly state what this is NOT.**

End the list with a single line:
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Self-check before you call it done

- Did you actually read the diff, or are you working from memory? Did it include untracked files and, for a branch review, every commit since the merge base?
- Was the size gate computed **before** any dispatch, on production code lines with docs and test paths excluded — and did an uncomputable count fail open to the cold read, not closed to the cheap path?
- On the cold-read path: one reader (two only above the split, never more), each Agent call carrying an explicit `model` at the cheaper tier, no conversation context in the prompt — and did synthesis — every accept/reject — happen in this session, not in a subagent?
- Did the output name, for every gated lens — the four domain lenses always, the code checklists too when the prose path gated them — the clause that matched or that none did, plus the `Readers:` count, so both an over-firing and an under-firing gate are visible?
- Did the reader's reply stay within the per-checklist cap — and did a `Withheld:` line make you read that checklist's domain yourself rather than re-dispatch?
- Is every surviving risk concrete — a specific `file:line` and an articulable failure mechanism — not a vague "this could be better"?
- Did rejected findings stay rejected? A list padded with nit-churn to reach five items is a worse pointer than a list of two real risks.
- Did you avoid inventing risks just to fill the list? If it's low-risk, say so.
- Did you end with the explicit non-approval line?
- Is the list scannable in under 30 seconds?

## Reference files

| File | What it answers |
|---|---|
| `references/lenses.md` | The checklist blocks the reader prompt is composed from — the code block with its three checklists and the four domain blocks — plus the shared frame's read budget and per-checklist output cap, and the dispatch mechanics (prompt order, the one-or-two-reader rule, the mandatory `model` argument) — read when dispatching the step-3b cold read. The conditions themselves live in step 3b, not here. |
| `references/examples.md` | Worked output examples (inline pass, cold-read synthesis, low-risk case) — read when unsure of the output shape. |
