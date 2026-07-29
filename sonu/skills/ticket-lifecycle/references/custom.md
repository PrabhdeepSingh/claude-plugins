# Adapter — bring your own tracker

Any tracker that is not one of the four shipped backends works through a user-authored adapter. This file is both the template for that adapter and the interview that generates it.

The extension point is the operations contract in `SKILL.md` section 2: a workflow only ever names an operation, so a tracker is fully supported the moment a document answers every one of them. A user adapter is the same kind of document as a shipped one — the only difference is who wrote it.

## Where the adapter lives

Config sets `tracker: custom` and an `adapter:` path, resolved repo-then-global exactly like the config itself:

1. the path in `adapter:` (relative to the repo root), default `.sonu/tracker-adapter.md`;
2. `~/.sonu/tracker-adapter.md`.

Commit the repo-local one so the team shares it. Keep the global one for a personal tracker used across many repos.

## The interview — generating an adapter during init

When the user's tool is not one of the shipped four, ask these seven questions, then write the adapter from the answers. Ask them in one batch; they are short, and a round trip per question wastes the user's time.

1. **What is the tool**, and what does it call a ticket (issue, work item, card, story)?
2. **How does an agent reach it** — a CLI (which command, is it authenticated already), an MCP server (which tools), or an HTTP API (which base URL)?
3. **Which credentials**, and where do they come from? Environment variable names only — never a pasted secret, and never a value written into the config or the adapter.
4. **What marks the two triggers** — a label, a status/state, a field value, a tag?
5. **How do type and priority map** onto native fields or labels, including what to do when a value has no native equivalent?
6. **How is a ticket marked done**, and does merging a PR do it automatically (magic words, an integration) or does the sweep have to do it explicitly?
7. **How is a ticket identified** in a branch name and commit message, so the work traces back to it?

Then write the adapter, echo the path, and tell the user to review it before the first pass — a generated adapter is a draft, and question 5's edge cases are where a wrong guess hides.

## Template

Copy this shape. Keep the operation names; a workflow looks them up by name.

```markdown
# Adapter — <tool name>

## Access
How the agent reaches the tool, which credentials by environment-variable name,
and the explicit stop condition when they are missing.

## Mapping
| Concept | Stored as |
|---|---|
| Trigger | ... |
| Type | ... |
| Priority | ... |
| Discussion | ... |
| Close the loop | ... |

## The operations
**list queue** — open tickets carrying a given trigger.
**list open** — every open ticket, trigger or not.
**search** — matching tickets across open *and* closed work.
**fetch** — one ticket in full, discussion included.
**claim** — confirm the trigger is present, clear it, confirm it is gone.
**update body** — replace the description, preserving the reporter's text.
**comment** — append to the discussion.
**classify** — set one type and one priority.
**create** — open a new ticket, no trigger.
**close the loop** — mark done once the PR merges.

## Bootstrap
Anything that must exist before the first pass (labels, states, fields), or
"nothing to create" when the tool creates values implicitly.

## Provenance and maintenance
Date-stamped notes on the commands and API shapes, with one-line re-verification
steps, since these are the facts most likely to rot.
```

## Rules a custom adapter must satisfy

These are not style preferences — each one is a failure mode the shipped adapters already avoid:

- **Every operation in the contract, each with a concrete mechanism.** A resolved adapter missing any operation is a **hard stop that names the gap**. Workflows must never improvise tracker mechanics: an improvised claim builds a ticket twice, and an improvised close strands finished work in flight forever.
- **Claim must be verifiable.** After clearing the trigger, re-read the ticket and confirm it is actually gone. An unverified claim is not a claim, and it is the only thing standing between two concurrent agents and the same ticket.
- **Claim must be able to fail loudly.** If the mechanism cannot report failure, the adapter is not safe for parallel work; say so in the adapter, and the pass stops rather than proceeding on an unconfirmed claim.
- **Credentials by environment variable only**, with an explicit stop when they are absent. Never a literal secret in the adapter, the config, or ticket text — those files get committed.
- **No trigger is ever applied by the adapter.** Only humans authorize; an adapter that offers an apply-trigger operation invites a workflow to self-authorize.
- **Priority must express "unset"**, because unset is the taxonomy's signal for a ticket recommended for rejection. If the tool has no such state, say what stands in for it and note the ambiguity.
- **Every value must already exist in the tool.** An adapter never creates fields, statuses, or types on the fly; a missing configured value is a configuration error to report, not a gap to fill silently.

## Promoting an adapter

An adapter that has proven itself is a candidate to ship: it is already the same document shape as the four in this directory, so promoting it means moving the file into a plugin release and pointing the spine's reference table at it — a copy and two edits, not a redesign. That path runs through a normal branch and PR like any other component change.

## Provenance and maintenance

Last verified 2026-07:

- The operations contract lives in `SKILL.md` section 2 — if an operation is added or renamed there, this template and every adapter need the same edit. That single-home rule is why the contract is not restated in full here.
