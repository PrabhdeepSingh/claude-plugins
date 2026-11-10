---
name: self-review
description: >-
  Lists the riskiest parts of the current diff so a reviewer knows where to look, using one independent reader on large or high-risk diffs. Use when a change is about to be handed off or shipped. Never approves or fixes. Consumer impact: [[blast-radius]].
argument-hint: "[what to review — defaults to the current diff]"
allowed-tools: Skill, Bash, Read, Agent
---

# Self-review — point attention, do not approve

The project's own instructions win: where its AGENTS.md, CLAUDE.md or established conventions say otherwise, follow them and skip that part of this. In particular, when the project defines its own review step (an external reviewer, a review command, a required second reader), that review is the independent read: do the inline pass here at most, and do not dispatch a reader on top of it.

The output is a short list of the places a reviewer should look hardest and why. It is not a score or an approval, because the author of a change is the least reliable judge of it. On a large or high-risk diff the reading is therefore done by a reader that never saw this conversation, and its findings are accepted only when they hold up against the diff.

It runs at `/sonu:build`'s hand-back, inside `/sonu:ship`'s pre-PR loop, or standalone. Invoked directly as `/sonu:self-review`, review `$ARGUMENTS`; if that token appears literally or is empty, review the working tree or current branch as picked below.

## 1. Read the diff

Pick the diff by the state of the tree.

- **Uncommitted changes:** `git diff HEAD`, plus `git status --porcelain` for untracked files. New files that were never added do not appear in `git diff HEAD`; read each one.
- **Just committed, tree clean:** `git show HEAD`.
- **Staged only:** `git diff --cached`.
- **Whole branch:** `git diff <base>...HEAD`, where the base is `origin/<base>` when a remote exists and the local base branch when it does not. Reviewing only the last commit of a multi-commit branch misses the earlier ones.

If there is nothing to review, say so and stop.

## 2. Size the change

The size of the production-code change decides how much review it is worth. Documentation and test files are left out of the count because their volume says little about code risk; the reader still reads them. Deletions count. `/sonu:ship` uses this same count, so this fence is its one home.

```bash
BASE=main   # substitute the base branch step 1 picked
if git rev-parse --verify -q "origin/$BASE" >/dev/null; then REF="origin/$BASE"
elif git rev-parse --verify -q "$BASE" >/dev/null; then REF="$BASE"   # no remote: the local base branch is the reference
else echo "COUNT UNCOMPUTABLE: no $BASE branch locally or on origin"; exit 1
fi
git diff --numstat "$REF...HEAD" \
  | grep -vE '\.(md|mdx|txt|rst|adoc)$' \
  | grep -vE '(^|/)(test|tests|__tests__|spec)/|\.(test|spec)\.[a-z]+$|_test\.go$|(^|/)test_[^/]*\.py$' \
  | awk '{ n += $1 + $2 } END { print n+0 }'
```

Use the same filter on whichever diff command step 1 picked, and add the line counts of untracked production files. In a repo whose product is its documents (a skills or plugin repo, a docs site), count all changed lines, because there the prose is the behaviour.

How to use the number:

- A small change, up to roughly a hundred lines, gets the inline pass (step 3a). A subagent costs more than it finds there.
- A larger change gets one independent reader (step 3b), because the author's blind spots grow with the diff.
- A small change still gets the reader when it is high-risk: it changes branching logic across a module boundary, asserts something the type system cannot check (thread safety, idempotence, ordering), or cannot be undone (a migration, an auth path, a contract other code consumes).
- A very large change (well past a thousand lines or about a hundred files) cannot be covered by one read. Review the highest-risk files, say in the first line of the output which files were read and that the rest were not, and recommend splitting the change.
- If the count cannot be computed, judge size by reading the diff. A missing remote or an unusual repo state is not a reason to take the more expensive path.

When the harness has no subagent tool, do the inline pass at any size.

## 3a. Inline pass

Identify the three to five riskiest spots: the things that would be hardest to catch in review and most expensive to fix later. Fewer is fine. Categories to consider:

- subtle logic: branches, edge cases, off-by-ones;
- security surfaces: auth, permissions, input handling, data exposure, secrets;
- data integrity: schema changes, destructive or irreversible writes;
- broad blast radius: a shared utility, a config, or a data shape other components consume. Cite [[blast-radius]]'s consumer list when the change produced one, and flag its absence when it should have;
- untested edges;
- a second implementation: a new helper that duplicates an existing function. Search the repo for the helper's verb and noun before accepting it;
- silent behaviour change, especially for a consumer that catches failures and returns a default;
- duplicated judgment: two expressions that decide the same thing separately (a filter and the transform it protects, a cap and the accumulator it bounds) and can drift;
- interface regressions, only when the diff touches user-facing interface files: a keyboard or screen-reader path that no longer completes, a layout that breaks at a supported width, copy that misstates a consequence. The rules are in [[accessibility]], [[layout]], [[typography]], [[colors]], [[ui-polish]] and [[ux-writing]]; [[interface-review]] only coordinates them.

Then go to step 5.

## 3b. Independent reader

Dispatch one read-only reader with the harness's subagent tool. Its prompt is composed from `references/lenses.md`: the shared frame, the code checklists when the diff changes executable code, and each domain checklist whose condition below matches. One reader carrying several checklists reads the diff once; several readers each read it again and add little. Use two readers only when the diff is very large and at least one domain matched: one carries the code checklists, the other the domain checklists.

The reader gets the diff command and the repo, and nothing from this conversation, because a reader that knows the author's intent inherits the author's blind spots. Its prompt is self-contained; do not tell it to load skills. If the harness lets you choose the reader's model, pass a cheaper one explicitly, since an Agent call without `model` runs on the session's model.

**Code checklists** (correctness, test adequacy, silent behaviour change) are carried whenever the diff changes executable code. A non-doc file changed only in metadata (a version string, a lockfile hash) is not code for this purpose.

**Domain checklists** are carried only when the diff contains their domain. A checklist with no domain in the diff produces invented findings.

- **Security.** Carry it when the diff moves or weakens a trust boundary: untrusted input reaching a sink, a control that protects a boundary being added, changed or removed, or a secret crossing one. That sentence is the test, and a change that fits it but matches no example below still qualifies. Examples: auth or authorization, including a guard enforced only on the client; an API, route or endpoint handler; SQL or any constructed query, command or path; deserialization or dynamic evaluation of untrusted data; middleware; crypto, secret, token or credential handling; payments; env vars or config that gate runtime behaviour or name an outbound destination; session, header, CORS, `postMessage` or frame-isolation handling; file I/O, uploads or downloads; subprocess or shell invocation; outbound requests to user-influenced destinations; dependency manifests, lockfiles or a newly included third-party script; untrusted input interpolated into HTML, a URL, a navigation target or a log line; a regex applied to user input; a recursive merge or dynamic property assignment from user-controlled data; model or tool output reaching an executable sink. This is the one home of the security-surface test; `/sonu:ship` cites it.
- **Data integrity.** Carry it when the diff touches migrations, schema, serialization, backfills, bulk or destructive writes, or persisted state that other code or a later release reads back.
- **Blast radius.** Carry it when the diff changes something read or addressed by code outside the diff, by the same test [[blast-radius]] states. Skip purely internal changes and additive optional fields.
- **Interface.** Carry it when the diff touches user-facing interface files.

Leave a domain out only when you can say in one sentence why its domain is absent ("nothing here touches persisted state"). If you cannot, carry it.

**A diff with no executable code.** In a repo whose product is its documents, a skill or command file is the behaviour. Insert the prose frame from `references/lenses.md`, apply the same four domain conditions to what the prose governs, and carry a code checklist only where the prose has its domain: correctness when it states rules or thresholds an executor follows, silent behaviour change when it changes what a component does without saying so. For a typo or wording pass, say the diff is low-risk and do the inline pass.

## 4. Judge the findings yourself

The reader gathers evidence. Every accept or reject happens in this session.

- A finding survives only if it cites a concrete `file:line` and you can state how it goes wrong. Style and preference findings are rejected.
- A finding about code outside the diff survives only if you can name the changed line that enabled it: a guard removed, a caller added, an argument now passed through.
- Close every `Unverified:` line yourself before writing the list, by reading the file or stating why the question does not arise.
- Two checklists flagging the same place become one entry, ranked higher. The same mistake in several places is one theme with several locations.
- Rank by leverage, structural problems first, and keep the top three to five.
- A `Withheld:` line means the reader found more than it reported; read that area yourself.
- If two passes produced substantive findings and you accepted none, say so in the hand-off, because that pattern usually means you are defending your own work.
- If nothing survives, say the diff is low-risk and why. Do not promote rejected findings to fill the list.

## 5. Write the list

One line per item, with `file:line` where it helps:

```
Risk: <what> — <why it's risky> [file:line]
```

When a reader ran, end with a line naming what each domain checklist did, giving the clause that matched or saying none did, then the reader count:

```
Domain lenses: interface (stylesheets + components) · security, data-integrity, blast-radius — no clause matched. Readers: 1
```

`/sonu:ship` reads the `Domain lenses:` prefix and its per-domain shape to set its security verdict, so keep them exactly. On a prose-only diff add a `Code checklists:` line saying which were carried. A reader that failed or returned nothing usable is reported as a risk entry naming the checklists it carried as unread, not as a clean pass.

## 6. Say what this is not

End with:

> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Reference files

| File | What it answers |
|---|---|
| `references/lenses.md` | The reader prompt: shared frame, prose frame, code checklists and the four domain checklists |
| `references/examples.md` | Example outputs for the inline pass, the reader path, a low-risk diff and a failed reader |
