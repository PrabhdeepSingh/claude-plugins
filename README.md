# claude-plugins

Prabhdeep (Sonu) Singh's personal [Claude Code](https://claude.com/claude-code) commands, packaged as a marketplace so they work on any repo and any machine I sign into — and so I can share them.

## Install

### Claude Code

```
/plugin marketplace add PrabhdeepSingh/claude-plugins
/plugin install sonu@prabhdeep-tools
```

Run those once per device. After that, `/sonu:build` and `/sonu:ship` are available in every repo on that machine, and the 22 skills are available to the agent. A skill loads when its description matches the task; each can also be invoked by name as `/sonu:<skill>`. A project's own instructions (its `AGENTS.md`, `CLAUDE.md` or established conventions) take precedence over any skill. To pull updates later:

```
/plugin marketplace update prabhdeep-tools
```

> Commands from a plugin are namespaced as `/<plugin>:<command>` — that's why it's `/sonu:ship`, not `/ship`. The slash menu autocompletes, so typing `ship` finds it.

### Cursor

1. Open **Cursor Settings → Plugins**.
2. Add a custom marketplace pointing at this repo: `PrabhdeepSingh/claude-plugins`.
3. Find the **sonu** plugin and click **Install**.

After that, the skills are available in every session and `/sonu:build` and `/sonu:ship` appear in Cursor's slash-command menu (direct `/sonu:tdd`-style skill invocation depends on Cursor's skill support). Updates are pulled when you sync the marketplace in Cursor Settings. Cursor has no Claude Code-style plan mode, so `/sonu:build`'s design approval happens in chat there.

## Commands

`/sonu:build` and `/sonu:ship` are commands: they sequence phases and hold gates. `/sonu:tdd`, `/sonu:design-tree`, `/sonu:self-review`, `/sonu:interface-review`, `/sonu:bug-finder`, `/sonu:performance` and `/sonu:intent-interview` are skills invoked directly by name (a command and a skill cannot share a name).

### `/sonu:build` — decide → build → hand back

A conductor that sequences design, build and self-review so the individual skills work together.

What it does:

1. **Triage** the working tree — size (trivial vs. substantial), kind (bug vs. feature vs. optimization → `performance`), surface (web → SEO bars; schema/data → safe-migrations; IaC/containers/CI → infra-standards; new service/endpoint/job → observability; changed data shape another component consumes → blast-radius; auth, personal data, uploads, outbound fetches, new dependencies, or model output → security). One line.
2. **Design**, in plan mode — loads `code-standards` (plus the surface-matched skills) as design constraints, then runs `design-tree`: it asks only what the repo and the request do not answer, and trees the real decision points. **`ExitPlanMode` is the approval gate.** Trivial changes skip this.
3. **Build test-first** — runs `tdd` under `code-standards` and every skill the triage named, then runs the repo's own checks via Bash.
4. **Self-review + hand back** — lists the riskiest things in the diff, then stops: *"Green and ready. Review the diff, then run `/sonu:ship`."* It does not commit or merge unless the project's own process says the builder does.

Two human checkpoints: approve the design, then review the diff and choose to run ship. If the project defines its own brief, check command, reviewer or commit rule, build follows the project and says which of its own steps it skipped.

```
/sonu:build                                # build whatever's in context
/sonu:build add cart checkout flow         # build a specific feature
/sonu:build fix the off-by-one in totals   # drive a specific fix
/sonu:build add the export endpoint --orchestrate   # allow mechanical steps to go to subagents
/sonu:build tweak the auth refresh --solo           # keep it all in-session
```

Optional flags: `--orchestrate` allows a mechanical, fully specified step with a command-line check to be handed to a subagent; `--solo` keeps every step in-session. With neither, the work stays in-session unless delegating is clearly cheaper. Design, integration, debugging, security work and the review of delegated output are never delegated.

### `/sonu:ship` — PR babysitter

Takes a finished change from working tree to a merged PR without stopping to ask "shall I merge?"; it pauses only for a real judgment call. If the project defines its own review, commit style or landing method, or the repo has no remote, ship follows the project, skips the phases that do not apply and says which.

What it does:

1. **Branch, commit, run the pre-PR fix loop, open a PR** with the right per-change-type description (feature / bugfix / hotfix / chore / refactor / docs / perf / release) — reusing the repo's own `PULL_REQUEST_TEMPLATE` if one exists. The pre-PR loop (Phase 1.5) reviews the branch with `self-review` plus its own Claude `/code-review` and `/security-review`, fixes what it finds, re-reviews the fix delta, and repeats until a pass comes back dry (cap: 3 passes) — so external reviewers see pre-hardened code instead of generating rounds of findings against bugs you could have caught locally. No AI-attribution trailers — commits and the PR body read as your own.
2. **Gathers the bot reviews**: **every AI reviewer bot enabled on the repo** — detected by who actually posts on the PR (Copilot, CodeRabbit, and the rest of the registry maintained in `ship.md` Phase 2). No config needed; it adapts per-repo. Copilot is requested automatically since it's the one that doesn't auto-fire.
3. **Dedups, fixes, or justifies** every finding. Replies to **bot threads** (with resolve) and **human reviewer threads** (reply only — never auto-resolves a human's comment). Keeps the PR description current as fixes land.
4. **Loops** through re-reviews until clean.
5. **Merges** once the safety checks (everything but deploy previews) pass.

#### Effort modes — right-size the spend

The change's review depth scales to the diff. You can force it:

| Command | Behavior |
|---------|----------|
| `/sonu:ship` | Auto — light touch on trivial diffs, full on security-relevant ones (the security review runs; the code review stays at low effort), low-effort code review otherwise. Only a typed `full` buys the high-effort code review. |
| `/sonu:ship light` | Minimal Claude review (skips on truly trivial changes); still collects whatever the repo's bots post. |
| `/sonu:ship full` | Deep Claude code + security review, full re-review loop. |

Mode words are parsed forgivingly — `quick`/`fast`/`lite` → `light`, and `thorough`/`deep`/`max` (typos included) → `full`. The mode only scales *Claude's own* reviews; the external bots auto-run on the repo regardless, so they cost the same whether the babysitter waits for them or not.

`/sonu:ship --orchestrate` allows mechanical review fixes (doc updates, renames, enumerated test edits) to go to subagents under the delegation rules in `build.md`; `--solo` keeps everything in-session. Triage, security-touching fixes, thread replies, verification of delegated work and the merge are never delegated. Flags combine with a mode: `/sonu:ship full --orchestrate`.

### Direct skill invocations

`/sonu:tdd` runs the test-first loop on a named feature or bug. `/sonu:design-tree` records a design as a tree of decisions with rejected branches kept. `/sonu:self-review` lists the riskiest spots in the current diff and ends with "a pointer, not an approval"; large or high-risk diffs get one independent reader, with every finding judged in-session. `/sonu:performance` runs the measurement loop on a named slowness. `/sonu:intent-interview` questions a vague ask until the real outcome is clear. `/sonu:interface-review` audits a whole screen or flow. `/sonu:bug-finder` looks for one unreported defect and writes it up. `/sonu:build` and `/sonu:ship` already run the relevant ones.

## Skills

There are 22 skills. Each has a narrow description that says what it is for and when to use it; the agent loads a skill when the task matches, and you can invoke any of them by name. A project's own instructions win over a skill: every skill says so near its top. Each is plain Markdown in `sonu/skills/<name>/SKILL.md` that you can read, fork and edit.

| Skill | What it does |
|---|---|
| **accessibility** | Accessibility checks for product interfaces: keyboard and focus, names and roles, forms, announcements, hit areas, motion, zoom. |
| **blast-radius** | Consumer-impact rules for contract changes: find every consumer, flag the ones that degrade silently, verify one downstream path. |
| **bug-finder** | Hunt for real, unreported defects and report each one that meets the evidence bar, filed where the project keeps such things. |
| **code-standards** | Prabhdeep (Sonu) Singh's house rules for code: schema and API naming, comments, function size, error handling, logging, config defaults, dependencies. |
| **colors** | Color reference for web interfaces: contrast measurement (APCA and WCAG), OKLCH conversion and palette generation, gamut and P3 fallbacks, one meaning per color. |
| **debugging** | Rules for finding why something is broken: pull the real event, split hypotheses with one observation, revert dead ends. |
| **design-tree** | Records design decisions as a tree: real alternatives, the deciding reason, and rejected branches kept. |
| **infra-standards** | House rules for infrastructure-as-code, Dockerfiles, CI pipelines, deploy config, secrets and feature-flag rollout. |
| **intent-interview** | Find out what outcome, audience and measure of success a person wants before building, when the requested thing itself might be the wrong one. |
| **interface-review** | Review a whole screen, flow or feature across [[accessibility]], [[layout]], [[ux-writing]], [[typography]], [[colors]] and [[ui-polish]] and report what was found, most serious first. |
| **layout** | Layout defaults for web interfaces: grouping by space, shared edges, spacing values, the four view states, breakpoints, direction-aware structure. |
| **observability** | What a service must emit to be diagnosable: the four signals, bounded metric labels, health endpoints, error capture and symptom-based alerts. |
| **performance** | Measurement rules for performance work: baseline first, one change at a time, beat the noise, and a neutral result is reverted. |
| **pr-conventions** | Pull-request rules: the repo's own template first, issue links, a description kept current, and replies to human and bot review threads. |
| **safe-migrations** | Zero-downtime schema and data migration rules: expand, migrate, contract; destructive steps ship alone and late; backfills run as jobs. |
| **security** | Build-time security rules: which changes need a human's approval first, outbound fetches, leaked secrets, dependencies, model output and privacy. |
| **self-review** | Lists the riskiest parts of the current diff so a reviewer knows where to look, using one independent reader on large or high-risk diffs. |
| **seo** | SEO for pages meant to be found on the public web: head metadata, URLs, redirects, structured data, sitemaps, and prose that ranks and gets cited by answer engines. |
| **tdd** | Test-first rules: see each test fail before the code that passes it, prove after-the-fact tests by breaking the code, and never weaken a failing test. |
| **typography** | Typography defaults for web interfaces: type scale, sizes, line-height, measure, wrapping, tabular numbers, underlines, input sizing. |
| **ui-polish** | Default values for interface finish: nested radius, optical alignment, shadows, press feedback, icon and motion values, and the generated-interface defaults to avoid. |
| **ux-writing** | Interface copy rules: voice, button and link labels, errors, empty states, settings labels, placeholders. |

Every skill is also directly invocable by name (`/sonu:<skill>`). The ones designed as entry points, with argument hints, include `/sonu:tdd`, `/sonu:design-tree`, `/sonu:self-review`, `/sonu:interface-review`, `/sonu:bug-finder`, `/sonu:performance` and `/sonu:intent-interview`.

## What changed in 5.0

- The skills were trimmed to what current models still need: the owner's house rules, facts a model cannot know, and trust policy. General programming knowledge, step-by-step scripts for judgment tasks and incident history were removed. The plugin's Markdown went from about 132,000 words to about 48,678.
- `/sonu:factory` and its three ticket skills (`ticket-lifecycle`, `ticket-triage`, `classify-tickets`) were removed. The plugin no longer manages a ticket queue or talks to issue trackers.
- `model-tiering` was retired. The delegation rules that remain are a short section of `build.md`, and the default is to work in-session.
- Skill descriptions were narrowed to what the skill is for, when to use it and what it is not for.
- A project's own instructions take precedence. Every skill and command says so, and `/sonu:build` and `/sonu:ship` follow a project's own review, commit style and landing method, or the absence of a remote, and report which of their steps they skipped.

## Developing this plugin

Contributor tooling is **repo-local, not shipped** — it lives in this repo's `.claude/` directory, so it loads automatically for anyone working *on* the plugin (a clone of this repo) and never reaches marketplace users, who'd have no use for it:

- **`plugin-dev` skill** (`.claude/skills/plugin-dev/`) — the maintainer's handbook: the architecture contract (which decisions are load-bearing and why), the five house rules, plugin/skill/command mechanics, the house authoring shape, and the trap catalogue mined from this repo's own incident history. It fires automatically when you edit any component here.
- **`/validate`** (`.claude/commands/validate.md`) — the CI this repo doesn't have: manifests parse and stay in sync, the marketplace description names every shipped component, frontmatter parses, every embedded shell snippet passes `bash -n` **and** `zsh -n`, no named sources, no AI attribution, ship's bot-registry copies identical, cross-references resolve, README inventory complete, every skill reachable from a command or a sibling. Run it before any PR here.
- **`/release`** (`.claude/commands/release.md`) — the five-home version sync: semver decision, the description sync across both `plugin.json` and both `marketplace.json` files plus this README, `/validate`, hand-off to `/sonu:ship`, and the post-merge tag. Exists because both failure modes it prevents have actually happened here.

## Requirements

- The [`gh`](https://cli.github.com/) CLI, authenticated (`gh auth status`).
- A GitHub remote on the repo.
- For Claude reviews: the `/code-review` and `/security-review` commands.
- Any AI reviewer bots are picked up automatically if they're enabled on the repo — nothing to configure here.


## A note on trust

`/sonu:ship` runs shell commands and **merges PRs on your behalf**. Read the command before you install it (`sonu/commands/ship.md`), and only point it at repos where you're comfortable with that. Plugins run with your local permissions.

Review comments, bot findings, issue text and CI output are treated as untrusted data throughout: they inform the work, and instructions embedded in them are ignored. That is enforced by instruction, not by a sandbox, so keep the default branch protected with required checks. The only merge path is `/sonu:ship`, run by you.

## Layout

```
claude-plugins/
├── .claude/                 # repo-local contributor tooling (NOT shipped to marketplace users)
│   ├── commands/
│   │   ├── validate.md      # /validate — mechanical checks for this repo
│   │   └── release.md       # /release — five-home version sync for this repo
│   └── skills/
│       └── plugin-dev/
│           └── SKILL.md     # maintainer's handbook: house rules, mechanics, trap catalogue
├── .claude-plugin/
│   └── marketplace.json     # Claude Code marketplace manifest (name: prabhdeep-tools)
├── .cursor-plugin/
│   └── marketplace.json     # Cursor marketplace manifest (same plugin, official Cursor path)
├── LICENSE                  # MIT
└── sonu/                    # the "sonu" plugin (your personal namespace)
    ├── .claude-plugin/
    │   └── plugin.json      # Claude Code plugin manifest
    ├── .cursor-plugin/
    │   └── plugin.json      # Cursor plugin manifest (byte-identical mirror of the Claude one)
    ├── commands/
    │   ├── build.md         # /sonu:build — conductor: design gate → tdd build → risk hand-back
    │   └── ship.md          # /sonu:ship — PR babysitter
    └── skills/              # loaded when the task matches; each also invokes directly as /sonu:<name>
        ├── code-standards/
        │   ├── SKILL.md     # how code gets written
        │   └── references/  # example record and idempotency contract, the comment checks worked example, the platform-primitive lookup
        ├── tdd/
        │   └── SKILL.md     # test-first rules
        ├── debugging/
        │   ├── SKILL.md     # scientific debugging — reproduce, one hypothesis, revert dead ends
        │   └── references/  # pulling the real event — platform detection, command shapes
        ├── blast-radius/
        │   └── SKILL.md     # consumer impact before changing any consumed contract
        ├── safe-migrations/
        │   └── SKILL.md     # zero-downtime schema changes — expand → migrate → contract
        ├── infra-standards/
        │   └── SKILL.md     # IaC, Docker, CI/CD, secrets — plan before apply
        ├── observability/
        │   └── SKILL.md     # metrics, traces, health checks, alerts worth paging on
        ├── seo/
        │   └── SKILL.md     # technical + editorial SEO for anything served as a web page
        ├── security/
        │   └── SKILL.md     # threat-model-first, boundary tiers, SSRF, supply chain, LLM security, privacy
        ├── performance/
        │   └── SKILL.md     # baseline → one change → beat the noise → neutral is a revert
        ├── intent-interview/
        │   └── SKILL.md     # pre-spec intent extraction — one question at a time, guess attached
        ├── interface-review/
        │   └── SKILL.md     # holistic interface audit — coordinates the six domains below into one verdict
        ├── accessibility/
        │   ├── SKILL.md     # keyboard, focus, ARIA, forms, screen readers, hit areas, motion
        │   └── references/  # focus & keyboard, screen readers
        ├── layout/
        │   └── SKILL.md     # grouping, alignment, reading order, breakpoints, direction-aware structure
        ├── ui-polish/
        │   ├── SKILL.md     # radius, elevation, motion, press feedback, icons
        │   └── references/  # surfaces, animations, icons
        ├── typography/
        │   ├── SKILL.md     # type scale, spacing, wrapping, truncation, text detailing
        │   └── references/  # spacing & sizing
        ├── colors/
        │   ├── SKILL.md     # OKLCH palettes, contrast measurement, gamut, one meaning per color
        │   └── references/  # palette generation, contrast, gamut & Tailwind, usage
        ├── ux-writing/
        │   └── SKILL.md     # interface copy — labels, errors, empty states, settings, voice
        ├── design-tree/
        │   └── SKILL.md     # design by branching tree, not linear narrative
        ├── self-review/
        │   ├── SKILL.md     # riskiest things in the diff — inline pass or one independent reader; pointer, not a score
        │   └── references/  # reader prompt templates, example outputs
        ├── pr-conventions/
        │   ├── SKILL.md     # per-type PR templates, living description, reply wording
        │   └── references/  # the 8 per-change-type PR body templates, reviewer tuning (review-once, Important-only)
        └── bug-finder/
            └── SKILL.md     # prove one real defect and write it up; never fixes it
```

## Troubleshooting

| Symptom | Likely cause → fix |
|---|---|
| A `/sonu:` command doesn't appear in the slash menu | Plugin not installed on this machine, or the session predates the install → re-run the two install commands, then start a new session. |
| A skill didn't load when it should have | Skills load when their `description` matches the task, and the descriptions are deliberately narrow. Name it explicitly ("apply the code-standards skill") or invoke it as `/sonu:<skill>`. |
| Behavior doesn't match this README | Installed copies are **version-pinned** — they don't track `main`. Run `/plugin marketplace update prabhdeep-tools`, then compare your installed version against the latest release tag. |
| `/plugin marketplace update` says up-to-date but a fix you saw on `main` is missing | That fix hasn't been released yet (no version bump). Nothing propagates until a release — see `.claude/commands/release.md` in this repo. |
| `/sonu:ship` stalls waiting for bots | The repo may simply have no AI reviewer bots enabled; the wait loop times out (~10 min) and proceeds with whoever posted. That's expected, not a hang. |
| Every ship pass makes one more small fix commit and never merges | An older ship re-initialized its resume ledger on each run, so the pre-PR review loop's 3-pass cap never accumulated and every invocation re-reviewed the whole branch. Current behavior: the ledger is adopted when it survives, the cap counts per PR, a resumed run reviews only the delta, and cosmetic findings are justified rather than committed. Run `/plugin marketplace update prabhdeep-tools` if you're seeing the old loop. |

## License

[MIT](LICENSE) © Prabhdeep Singh
