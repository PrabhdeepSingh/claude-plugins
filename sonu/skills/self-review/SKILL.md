---
name: self-review
description: Surface the 3–5 riskiest parts of the current diff in plain language so a human reviewer knows exactly where to look hardest. INVOKE THIS PROACTIVELY whenever a change is finished and about to be handed off, reviewed, or shipped — even when the user never says "self-review."
---

# Self-review — point attention, don't bless the change

A self-review is not a score. It is not an approval. It is a pointer: *here are the spots a reviewer should look hardest at, and here is why.* Self-scoring rubber-stamps (the model that wrote the code is the least reliable judge of whether it's right). The value is directing human attention toward the risky corners — exactly what gets skimmed in a real review.

## When to apply this

- **Conductor hand-back** (`/sonu:build` Phase 3) — before handing back to the user.
- **Ship PR creation** (`/sonu:ship` Phase 1) — before writing the PR body, so the riskiest items can seed the `## Risk / reviewer attention` section.
- **Standalone** — any time the user says "self-review", "what should I look at", "what's risky here", or similar; or when a change is finished and about to be handed off.

## How to apply this

**1. Read the diff.**

Pick the right diff command for the current state:
- **Uncommitted changes** (working tree dirty): `git diff HEAD`
- **Just committed** (working tree clean, reviewing what was just committed): `git show HEAD` or `git diff HEAD^ HEAD`
- **Staged only**: `git diff --cached`

If the working tree is clean and there is no recent commit to review, say so and stop — don't invent risks on a clean tree.

**2. Identify the 3–5 riskiest spots.**

Focus on the things that, if wrong, would be hardest to catch in a review and most expensive to fix after the fact. Common categories (not a checklist — use judgment):

- **Subtle logic** — branches, edge cases, off-by-ones, conditions that are almost-but-not-quite right.
- **Security-relevant surfaces** — auth, permissions, input sanitization, data exposure, token/secret handling.
- **Data integrity / migration risk** — schema changes, destructive writes, non-reversible transformations.
- **Broad blast radius** — a change to a shared utility, a base class, a widely-imported module, or a config that silently affects many call sites.
- **Untested edges** — behavior that the new tests don't cover and that could break in production.
- **Silent behavior change** — the code "works" but now does something subtly different from before, in a way callers may depend on.

Aim for 3–5 items. If the diff is genuinely low-risk (small, isolated, well-covered by tests), say so plainly — "This diff is low-risk: X, Y, Z" — rather than inventing risks to fill the list.

**3. Write the list.**

For each item: one line on *what* it is and *why* it's risky. Add `file:line` when it helps a reviewer jump straight there. Keep it scannable — no paragraphs.

Format:
```
Risk: <what> — <why it's risky> [file:line]
```

Example output:
```
Risk: permission check on line 42 uses `user.role` before the role is loaded — could silently pass for unauthenticated users [auth/middleware.ts:42]
Risk: `deleteUserData()` is irreversible and has no dry-run mode — one bad caller will drop real data [users/service.ts:118]
Risk: the retry loop has no backoff — under load this will hammer the upstream API [api/client.ts:67]
```

**4. Explicitly state what this is NOT.**

End the list with a single line:
> *This is a pointer for your review, not an approval. Read the diff yourself.*

## Self-check before you call it done

- Did you actually read the diff, or are you working from memory?
- Is every risk concrete — a specific file, behavior, or condition — not a vague "this could be better"?
- Did you avoid inventing risks just to fill the list? If it's low-risk, say so.
- Did you end with the explicit non-approval line?
- Is the list scannable in under 30 seconds?
