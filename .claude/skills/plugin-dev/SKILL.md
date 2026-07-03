---
name: plugin-dev
description: The maintainer's handbook for the sonu plugin and the claude-plugins repo. INVOKE whenever authoring, editing, reviewing, or releasing any component in this repo — a skill's SKILL.md, a command file (under sonu/commands/ or .claude/commands/), a manifest, or the README — or when answering how the plugin's pieces fit together. Covers the house rules, plugin/skill/command mechanics, the authoring conventions, the trap catalogue mined from this repo's own incidents, and the validation/release procedure pointers. Do NOT load this for ordinary coding work in other repos — it governs work ON the plugin, not work done WITH it.
---

# Plugin development — how to carry this repo forward

This repo has no test suite, no CI, and no compiler: every quality gate is discipline plus the checks in `/validate`. This skill is the discipline written down, so a zero-context contributor (human or model) can change the plugin without re-fighting battles that are already settled. Everything here was extracted from this repo's actual git history — each rule has an incident behind it.

## When to apply this

Any change inside this repo: editing a skill or command, adding a component, bumping a version, touching a manifest or the README, or reviewing someone else's change to any of those. If your working directory is the `claude-plugins` repo, this skill applies.

**When NOT to use it:** using the plugin's skills/commands in some other repo. This file governs the toolbox, not the work done with the tools.

---

## 1. Architecture contract — the load-bearing decisions

These decisions are deliberate. Don't reverse them casually; if one must change, record why in the PR and update this section.

| Decision | Why it holds |
|---|---|
| **One plugin (`sonu`), one marketplace (`prabhdeep-tools`)** | The repo IS the marketplace. `marketplace.json` at the root points at `./sonu`; everything installable lives under `sonu/`. |
| **Commands are thin; skills carry the methodology** | A command is an entry point + sequencing; the how-to lives in exactly one skill. `/sonu:tdd` and `/sonu:design-tree` are page-length wrappers. If a command starts explaining methodology, it's duplicating a skill — stop and delegate. |
| **`/sonu:build` is a conductor, not an implementation** | It sequences design-tree → tdd → self-review and adds only triage and gates. It never re-implements what the skills already say (its own Pitfalls section enforces this). |
| **`/sonu:ship` is the one deliberately fat command** | Its bulk is incident-hardened, copy-exact shell with the failure lore attached. It delegates what it can (PR bodies and replies → pr-conventions, risk lists → self-review). Do not split it; do not "clean up" a snippet whose comment says copy it exactly. |
| **No hooks, no agents, no MCP servers, no scripts/** | Zero-dependency Markdown + JSON keeps the plugin installable anywhere with nothing to break. Adding executable machinery needs a strong reason and a PR discussion, not a drive-by. |
| **Dual manifests (Claude Code + Cursor)** | `.claude-plugin/` and `.cursor-plugin/` carry mirrored manifests at both the repo root (marketplace) and inside `sonu/` (plugin). Cursor reads its own copy; the two `plugin.json` files must stay byte-identical in content. |
| **Installed copies are pinned** | A user's installed plugin does NOT track `main`. Changes reach users only after a version bump + their `/plugin marketplace update`. An unbumped fix on main is invisible to every installed copy — that has already happened once (a ship.md behavior fix landed without a bump). |
| **Maintainer tooling is repo-local, never shipped** | This skill, `/validate`, and `/release` live in `.claude/` (project-level), not under `sonu/` — they only work with this repo checked out, so the audience boundary and the distribution boundary are the same line. Anything useful only to people working ON the plugin goes in `.claude/`; anything useful to people working WITH the plugin ships under `sonu/`. Don't put the next maintainer tool in the plugin. |

## 2. The house rules

These are law for every contribution. They pre-date this file; this is their canonical written home.

1. **No named sources.** Skills never cite books, authors, papers, or studies — not in the description, not in the body. Embed the knowledge as owned instruction. A named source sends a smaller model off to research the citation instead of applying the content (this happened: a named study in content-seo had to be excised).
2. **Description = routing signal only.** A skill's frontmatter `description` says *what it is* and *when to load it* (rich triggers, plus when NOT to and which sibling to use). Methodology, sequencing, and rules live in the body. The description is what the model reads when deciding whether to load the skill — nothing else fits there.
3. **No AI attribution anywhere.** No `Co-Authored-By` trailers, no "Generated with" lines — in this repo's own commits and PRs, and in everything ship produces. Work reads as the owner's own.
4. **Version-sync in the same PR.** Any component change bumps the plugin version and syncs the plugin description across its five homes — they have drifted before. The exact file list and procedure live in exactly one place, `/release` Phase 2; follow that rather than reciting from memory.
5. **Dogfood the flow.** Changes go branch → PR → review → merge (use the plugin's own `/sonu:build` and `/sonu:ship`). Never commit directly to `main`.

## 3. Component mechanics — how the pieces actually work

Definitions a zero-context contributor needs (re-verify against the official Claude Code plugin docs if behavior seems off — see Provenance):

- **Skill** = `sonu/skills/<name>/SKILL.md` with YAML frontmatter carrying `name` and `description`. Skills are **auto-triggered**: the harness matches the task against the description and loads the body when it fits. There is nothing to invoke; the description IS the trigger surface. Registered name is namespaced: `sonu:<name>`.
- **Command** = `sonu/commands/<name>.md`. Invoked explicitly as `/sonu:<name>`. Frontmatter fields used in this repo:
  - `description` — shown in the slash menu; also the model's routing signal.
  - `argument-hint` — placeholder text for the argument, e.g. `"[light|full]"`.
  - `allowed-tools` — tools pre-approved for the command's execution. It reduces permission prompts; it is **not** a hard block on other tools (an audit finding that misread it as a block was refuted — don't repeat that mistake in either direction).
  - `disable-model-invocation: true` — prevents the model from firing the command on its own; the user must type it. Set on `/sonu:ship` deliberately: a command that pushes and merges must never be model-triggered. Keep it there.
- **`$ARGUMENTS`** — the literal text typed after the command. House convention for handling it, used verbatim in every command: *"the text typed after the command; if that token appears literally or is empty, derive the task from context."* (The "appears literally" clause covers harnesses that don't substitute the token.)
- **Skill invocation from a command**: `Skill(sonu:<name>)` — always the namespaced form. Unqualified names are a real bug that shipped once.
- **`[[…]]` links** — an in-house cross-reference convention between sibling skills (e.g. `[[tdd]]`, `[[seo-standards]]`). There is no resolver; it just marks "see that skill." Keep using it; don't invent URLs. Every double-bracket reference must name a real sibling skill — `/validate` checks this.

## 4. Authoring conventions — the house shape

Every skill follows the same skeleton. Match it exactly when adding one:

1. Frontmatter: `name` + trigger-rich, routing-only `description` (rule 2), including when NOT to load it and the sibling to use instead.
2. An opening paragraph earning the skill's existence — the *why*, in the owner's voice.
3. `## How to apply this` / `## When to apply this` — operational entry.
4. Numbered `##` sections, each rule with its *why* and, where code is involved, an `Avoid:` / `Prefer:` example pair.
5. `## Self-check before ...` — a checklist the model runs against its own output.
6. `## Provenance and maintenance` — **only if the skill contains volatile facts** (external-world claims, tool behaviors, thresholds). Date-stamp when they were verified and give one-line re-verification commands or searches. Durable-methodology skills (tdd, design-tree) don't need one.

Commands follow: frontmatter → one-line contract → phased, imperative runbook → pitfalls.

Style rules that are non-negotiable across both:

- **Write for the literal executor.** Assume a smaller model follows the text word-for-word with no context. Ambiguity is a bug: "at most N cycles" once read as "the loop is optional" and shipped a real defect — the fix was the explicit "mandatory whenever…, the cap limits how many, not whether."
- **Every shell fence is self-contained.** Each Bash invocation is a fresh shell: no variable survives between fences. Every snippet declares what it uses (`BOT_RE=…`, `REPO=$(…)`) or marks literal substitution (`PR=<PR number>`). Never rely on "it was set earlier."
- **Evergreen examples.** No dates that rot (use far-future dates in test examples — a real fix in this repo's history), no version numbers that will drift, no "as of this year."
- **One home per fact.** A fact (a registry, a numeric limit, a wording table) lives in exactly one file; everything else points at it. When you find yourself pasting a table between files, stop and cross-reference instead.
- **When adding to a checklist-style skill, carry the rationale.** A rule without its *why* becomes cargo cult within two releases — several dated SEO rules had to be excised precisely because their why was missing or obsolete.

## 5. Trap catalogue — settled battles, do not re-fight

Each of these bit this repo once. The lesson is already encoded at the point of use; this table is the index so you recognize the pattern *before* writing new instances of it.

| Trap | Rule | Where the lesson lives |
|---|---|---|
| zsh's `[`/`test` builtin rejects `\>` string comparison | Use `[[ "$a" > "$b" ]]` for lexicographic compare — works in bash and zsh | ship.md Phase 6 |
| `$(jq -e … && echo yes)` captures jq's `true` output plus `yes` | Discard jq stdout (`>/dev/null`), branch on exit code only | ship.md Phase 7 |
| YAML frontmatter with an unquoted `: ` inside the description breaks parsing | Reword or quote — validate frontmatter after every edit | any SKILL.md (incident: the first SEO-skill PR) |
| "at most N times" read as "optional" | State mandatory-vs-cap explicitly | ship.md Phase 6 preamble |
| Fresh shell per Bash call — unset vars fail *silently* in jq filters and API paths | Self-contained fences, loud guards | ship.md "Shell discipline" |
| `git diff HEAD` omits untracked files | Pair with `git status --porcelain`; read new files directly | self-review skill, build.md |
| Code fences inside markdown list items are indented — an indented heredoc terminator is a zsh parse error | Compose heredocs at column 0; `/validate` dedents before syntax-checking | ship.md Phase 0/1 BODY snippets |
| Plugin description drifts across its five homes | Version-sync rule 4; run `/release` | this file §2, release.md |
| A behavior fix without a version bump reaches no installed copy | Every component change bumps the version | this file §1 (pinning) |

## 6. Validation and release

- **Before opening any PR:** run `/validate` — it mechanically checks manifest sync, frontmatter, shell-fence syntax, named-source and attribution scans, and cross-reference integrity. Fix everything it flags or justify in the PR body.
- **When the change is user-visible:** follow `/release` for the version bump and the five-home sync. That command is the canonical procedure; this file deliberately doesn't restate it.
- **What counts as evidence here:** for shell snippets, a real execution transcript (or `bash -n`/`zsh -n` at minimum); for external claims (bot logins, gh behavior, SEO facts), a fresh check against the live tool or current docs — dated in the Provenance section; for wording changes in ship.md, a dry read asking "how would a literal executor misread this?"

## Self-check before you call it done

- Does the change follow the house shape (§4) and every house rule (§2)?
- Is every new fact in exactly one home, with cross-references elsewhere?
- Did you check the trap catalogue (§5) — especially if you wrote shell, YAML frontmatter, or loop/cap wording?
- Did `/validate` pass?
- If a component changed: version bumped and all five homes synced (`/release`)?
- Is the change going out through a PR (rule 5), with no AI attribution (rule 3)?

## Provenance and maintenance

Last verified 2026-07:

- **Component mechanics (§3)** — frontmatter fields, namespacing, `$ARGUMENTS`, `disable-model-invocation` — re-verify against the official Claude Code plugin/slash-command docs (`https://code.claude.com/docs`) when a new harness version lands.
- **Cursor plugin behavior** (dual manifests, what Cursor actually reads) — re-verify against Cursor's plugin docs; the mirror-the-Claude-manifest convention is this repo's, not an official spec.
- **The incident references (§5)** — stable history; `git log --oneline` reproduces the archaeology.
