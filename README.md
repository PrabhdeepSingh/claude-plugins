# claude-plugins

Prabhdeep (Sonu) Singh's personal [Claude Code](https://claude.com/claude-code) commands, packaged as a marketplace so they work on any repo and any machine I sign into — and so I can share them.

## Install

### Claude Code

```
/plugin marketplace add PrabhdeepSingh/claude-plugins
/plugin install sonu@prabhdeep-tools
```

Run those once per device. After that, `/sonu:build`, `/sonu:ship`, `/sonu:factory`, `/sonu:tdd`, `/sonu:design-tree`, `/sonu:self-review`, `/sonu:ticket-triage`, `/sonu:classify-tickets`, and `/sonu:bug-finder` are available in every repo on that machine, and the **code-standards**, **tdd**, **debugging**, **security**, **performance**, **intent-interview**, **blast-radius**, **safe-migrations**, **infra-standards**, **observability**, **seo**, **design-tree**, **model-tiering**, **self-review**, **pr-conventions**, **ticket-lifecycle**, **ticket-triage**, **classify-tickets**, and **bug-finder** skills ride along automatically — no command to run, they just shape how code and content get written. To pull updates later:

```
/plugin marketplace update prabhdeep-tools
```

> Commands from a plugin are namespaced as `/<plugin>:<command>` — that's why it's `/sonu:ship`, not `/ship`. The slash menu autocompletes, so typing `ship` finds it.

### Cursor

1. Open **Cursor Settings → Plugins**.
2. Add a custom marketplace pointing at this repo: `PrabhdeepSingh/claude-plugins`.
3. Find the **sonu** plugin and click **Install**.

After that, skills auto-apply in every session and `/sonu:build`, `/sonu:ship`, and `/sonu:factory` appear in Cursor's slash-command menu (direct `/sonu:tdd`-style skill invocation depends on Cursor's skill support — the skills themselves ride along and auto-apply either way). Updates are pulled when you sync the marketplace in Cursor Settings. (One difference: Cursor has no Claude Code-style plan mode, so `/sonu:build`'s design gate runs in-chat there — the command adapts automatically.)

## Commands

`/sonu:build`, `/sonu:ship`, and `/sonu:factory` are commands proper — they sequence phases and hold gates. `/sonu:tdd`, `/sonu:design-tree`, `/sonu:self-review`, `/sonu:interface-review`, `/sonu:ticket-triage`, `/sonu:classify-tickets`, and `/sonu:bug-finder` are the skills themselves invoked directly by name: same syntax, same behavior, but no separate command component (a command and a skill can't share a name — they collide on the harness's one invocation surface).

### `/sonu:build` — decide → build → hand back

The spine of the plugin — a thin conductor that sequences the whole implementation lifecycle so the individual skills work together as a pipeline instead of one-off tools.

What it does:

1. **Triage** the working tree — size (trivial vs. substantial), kind (bug vs. feature), surface (web → SEO bars; schema/data → safe-migrations; IaC/containers/CI → infra-standards; new service/endpoint/job → observability; changed data shape another component consumes → blast-radius). One line.
2. **Design**, in real plan mode — loads `code-standards` (plus the surface-matched bars) as design constraints first, then runs `design-tree` so you interview first and tree the real decision points. The plan must meet design-tree's executor-ready bar — exact paths, conventions settled in place, a verification check per step — before it goes to the gate. **`ExitPlanMode` is the approval gate.** Trivial changes skip this entirely.
3. **Build test-first** — runs `tdd` under `code-standards` and every bar the triage flagged; **runs the suite via Bash** to confirm green. Never takes green on faith.
4. **Self-review + hand back** — lists the 3–5 riskiest things in the diff, then stops: *"Green and ready. Review the diff, then run `/sonu:ship`."* Never commits or merges.

Two human checkpoints: approve the design (ExitPlanMode), then review the diff and choose to run ship. Everything in between is autonomous. (Invoked from `/sonu:factory` on a ticket, the first checkpoint has already happened — you approved the spec — so build skips the plan-mode pause and treats the spec's acceptance criteria as the design constraints. It still trees any fork the spec left open, just in-chat. Same two gates; one of them moved onto the ticket.)

```
/sonu:build                                # build whatever's in context
/sonu:build add cart checkout flow         # build a specific feature
/sonu:build fix the off-by-one in totals   # drive a specific fix
/sonu:build add the export endpoint --orchestrate   # pre-authorize fan-out
/sonu:build tweak the auth refresh --solo           # keep it all in-session
```

Optional `--orchestrate` / `--solo` flags set the [`model-tiering`](#model-tiering--route-the-work-keep-the-judgment) disposition for the run: `--orchestrate` pre-authorizes fan-out (tag every step that clears the delegation bar, break ties toward delegating), `--solo` keeps every step in-session. Neither = the skill's own balanced judgment. The flag is a tie-breaker, not an override — it never delegates design/integration/debugging/security/review work, and can't manufacture an executor tier that isn't there.

### `/sonu:ship` — PR babysitter

Takes a finished change from working tree to a clean, merged PR — autonomously. Once you run it, it doesn't stop to ask "shall I merge?"; it only pauses for a real judgment call.

What it does:

1. **Branch, commit, run the pre-PR fix loop, open a PR** with the right per-change-type description (feature / bugfix / hotfix / chore / refactor / docs / perf / release) — reusing the repo's own `PULL_REQUEST_TEMPLATE` if one exists. The pre-PR loop (Phase 1.5) reviews the branch with `self-review`, fixes what it finds, re-reviews the fix delta, and repeats until a pass comes back dry (cap: 3 passes) — so external reviewers see pre-hardened code instead of generating rounds of findings against bugs you could have caught locally. No AI-attribution trailers — commits and the PR body read as your own.
2. **Gathers every review source**: its own Claude `/code-review` + `/security-review`, plus **every AI reviewer bot enabled on the repo** — detected by who actually posts on the PR (Copilot, CodeRabbit, and the rest of the registry maintained in `ship.md` Phase 2). No config needed; it adapts per-repo. Copilot is requested automatically since it's the one that doesn't auto-fire.
3. **Dedups, fixes, or justifies** every finding. Replies to **bot threads** (with resolve) and **human reviewer threads** (reply only — never auto-resolves a human's comment). Keeps the PR description current as fixes land.
4. **Loops** through re-reviews until clean.
5. **Merges** once the safety checks (everything but deploy previews) pass.

#### Effort modes — right-size the spend

The change's review depth scales to the diff. You can force it:

| Command | Behavior |
|---------|----------|
| `/sonu:ship` | Auto — light touch on trivial diffs, full panel on big or security-relevant ones. |
| `/sonu:ship light` | Minimal Claude review (skips on truly trivial changes); still collects whatever the repo's bots post. |
| `/sonu:ship full` | Deep Claude code + security review, full re-review loop. |

Mode words are parsed forgivingly — `quick`/`fast`/`lite` → `light`, and `thorough`/`deep`/`max` (typos included) → `full`. The mode only scales *Claude's own* reviews; the external bots auto-run on the repo regardless, so they cost the same whether the babysitter waits for them or not.

Ship can also route the mechanical parts of its fix loop to cheaper model tiers: `/sonu:ship --orchestrate` pre-authorizes delegating review fixes that clear the model-tiering bar (doc updates, renames, enumerated test edits) to subagents, `--solo` keeps everything in-session, and with no flag the model-tiering skill decides — including doing nothing at all on a session with no cheaper trustworthy tier below it. Triage, security-touching fixes, thread replies, verification of delegated work, and the merge never delegate. Flags combine with a mode: `/sonu:ship full --orchestrate`.

### `/sonu:factory` — the ticket queue, without a daemon

A software factory runs work through a consistent pipeline instead of ad-hoc terminal sessions: work enters a queue, gets the same checks every time, and keeps moving until it needs a human decision. `/sonu:factory` is that pipeline with **you as the trigger** — no daemon, no hosted service, no background process spending tokens while you sleep. You authorize work by labeling a ticket; the command picks it up on your next invocation.

The tracker is the control plane. A ticket carries the problem, the scope, the acceptance criteria, and the evidence — and three labels record what a human has authorized:

| Trigger | Means |
|---|---|
| `factory-ready-for-spec` | Turn this raw ticket into an implementation-ready spec. |
| `factory-ready-to-implement` | Build this ticket from its approved spec. |
| `factory-ready-to-ship` | Ship the built branch — the `/sonu:ship` review loop through merge. This one is merge authority: apply it only after reviewing the diff, and gate who can apply it like you gate merge rights. |

On GitHub, Jira, and Linear these are label names. On the local file tracker the same authorizations live in the ticket's `trigger:` field (as `ready-for-spec` / `ready-to-implement` / `ready-to-ship` — the field name already supplies the scope).

Only a human ever applies one, each authorizes exactly one pass, and the workflow removes it as its claim before starting. That last part is what makes parallel agents safe: a claim first checks the trigger is actually there, so a second session dispatching the same ticket finds nothing to claim and stops.

**One build engine, two front doors.** `/sonu:factory` doesn't duplicate `/sonu:build` — it feeds it. Describe work in chat and build runs its design gate in plan mode. Route work through a ticket and factory claims it, then invokes that same build with the *pause* already satisfied, because the spec you approved is the approved design — build still trees whatever forks the spec left open, in-chat rather than in plan mode.

```
/sonu:factory init              # pick and configure a tracker (once per repo or globally)
/sonu:factory                   # scan the queue: spec what's spec-ready, ship the top shippable, build the top buildable
/sonu:factory triage 123        # spec one ticket
/sonu:factory implement 123     # build one approved ticket
/sonu:factory ship 123          # ship one built, human-reviewed ticket through review and merge
/sonu:factory 123               # infer the stage from the ticket's own trigger
/sonu:factory classify          # groom the backlog (one type, one priority per ticket)
/sonu:factory bugs              # hunt for one real defect and file it
/sonu:factory poll              # standing loop: watch the queue, work what you authorize
```

An end-to-end trip through the queue:

1. **A ticket exists** — you filed it, a teammate did, or `/sonu:factory bugs` found the defect and filed it.
2. **You authorize a spec pass** — the `factory-ready-for-spec` label, or `trigger: ready-for-spec` on a local ticket file.
3. **`/sonu:factory`** claims it and runs `ticket-triage`: reads the code, reproduces the bug where practical, and rewrites the ticket as a spec with testable acceptance criteria, explicit non-goals, and a verification plan. It never writes code and never authorizes the next stage.
4. **You read the spec** and either answer its questions or authorize the build (`factory-ready-to-implement`, or `trigger: ready-to-implement` locally). This is the real gate — a human deciding the thing is worth building as specified.
5. **`/sonu:factory`** claims it, creates a dedicated worktree, and hands it to `/sonu:build`, which builds test-first under your standards and hands back at a green suite with a risk list.
6. **You review the diff** and authorize the ship — run `/sonu:ship` yourself, or apply `factory-ready-to-ship` and let a pass do it. Either way the flow keeps the tracker's close reference in the PR (`Closes #N` on GitHub, `Fixes ENG-123` on Linear, the issue key on Jira, the ticket id in the commit on local) — GitHub and Linear then close the ticket on merge, and for Jira and local the next `/sonu:factory` sweep marks it done. The ship route only accepts a branch whose ticket proves the build finished (the *built* checkpoint comment, or an existing PR) — never a mid-build branch. **You apply that label once.** A ship spans bot reviews and CI waits that can outlast a single session, so a pass that runs out of turn parks the PR and the next `/sonu:factory` pass picks it up on the same authorization — no re-labelling, no babysitting.

Two human decisions (approve the spec, approve the diff); everything between them is autonomous. Nothing merges without your explicit authorization — typed or labeled.

While all this runs, the ticket itself tells the story: each pass posts checkpoint comments (claimed → plan settled → built, with the risk list), keeps a `factory:*` status label current so the issue list shows every ticket's stage at a glance, and parks any question it can't answer as a `blocked` ticket with the question in a comment — answer it and re-apply the trigger to resume.

#### Poll, parallelism, and trackers — the short version

**Poll mode** (`/sonu:factory poll`) turns the pass into a standing loop in your own session — sweep, ship one, build one, spec what's ready, idle, wake. You still apply every trigger; there is no hosted daemon, and the loop dies with your session. Passes keep an edited-in-place heartbeat so a died session is detectable (`factory:agent-lost`) and its work can be taken over from the spec — blocked tickets waiting on your answer are never flagged.

**Parallel work**: every implement pass builds in its own git worktree on its own `ticket/…` branch, claimed in the main checkout first — so several agents can run at once without stomping, and two sessions can never build the same ticket. Worktrees don't inherit untracked config (`.env` stays behind), so the pass copies what the suite needs.

**Trackers**: five backends — GitHub Issues (`gh`), Jira (MCP/REST), Linear (MCP/GraphQL), a zero-dependency local Markdown store under `.sonu/tickets/`, or `custom` (init interviews you and generates an adapter). Config lives in `.sonu/factory-config.md` (committed; `~/.sonu/` as the global fallback); credentials stay in environment variables, never in the file. Ticket-file edits on the local tracker are committed alone with a `tickets:` prefix — tracker metadata only, never source code. The full operation contract and per-tracker mechanics live in the `ticket-lifecycle` skill's adapters.

### Direct skill invocations

`/sonu:tdd` runs the red-green-refactor loop on a named feature or bug (writes real files, not a plan). `/sonu:design-tree` maps a design as an explicit branching tree — interview, real forks only, rejected branches preserved — into the plan file or in-chat. `/sonu:self-review` lists the 3–5 riskiest spots in the current diff and ends with "a pointer, not an approval"; substantial diffs get independent parallel review lenses. All three are the skills themselves (see the table below) invoked by name; `/sonu:build` and `/sonu:ship` already run them at the right moments.

## Skills

Twenty-six skills fire automatically as Claude works — nothing to invoke, nothing to configure. Each one is plain Markdown: the one-line summary below is the routing signal, and the full methodology (every rule with its why, worked examples, self-checks) lives in `sonu/skills/<name>/SKILL.md` — read it, fork it, edit it to tune the behavior.

| Skill | What it does |
|---|---|
| **accessibility** | Accessibility engineering for product interfaces — keyboard support, focus states, ARIA, forms, screen readers, hit areas, motion and zoom. |
| **blast-radius** | Consumer-impact discipline for contract changes — enumerate every consumer, flag the ones that degrade silently, verify one downstream path end-to-end before shipping. |
| **bug-finder** | Hunt for one real, previously unreported defect and file it as a well-evidenced ticket — proactive discovery, not reactive diagnosis. |
| **classify-tickets** | Backlog hygiene as a sweep — exactly one type and one evidence-based priority per open ticket, and nothing else changes. |
| **code-standards** | Prabhdeep (Sonu) Singh's personal coding standards — the house rules and quality bar for how code gets written. |
| **colors** | Color systems for web interfaces — OKLCH conversion and palette generation, contrast measurement (APCA/WCAG), gamut and P3 fallbacks, theming, one meaning per color. |
| **debugging** | The scientific debugging loop — reproduce first, pull the real event from the repo's observability stack, one hypothesis → one change → one observation, revert failed attempts, escalate instead of thrash. |
| **design-tree** | Make design decisions as an explicit branching tree — genuine alternatives, decisive rationale, rejected branches preserved. |
| **infra-standards** | Infrastructure, container, and CI/CD discipline — IaC, Dockerfiles and compose files, CI pipelines, platform deploy config, env/secrets handling. |
| **intent-interview** | Pre-spec intent extraction — one question at a time, each with a guess attached, until you can predict the user's answers; for when the asked-for artifact itself might be wrong. |
| **interface-review** | Cross-discipline review of a whole screen, flow, or feature — coordinates [[accessibility]], [[layout]], [[ux-writing]], [[typography]], [[colors]], and [[ui-polish]] into one ranked findings table and a single verdict. |
| **layout** | Layout structure for web interfaces — grouping, alignment, negative space, reading order, progressive disclosure, breakpoints, direction-aware (RTL) structure. |
| **model-tiering** | Tag each plan step with the cheapest model tier that can execute it reliably, then delegate tagged steps to subagents at execution — keeping an orchestrator-class session clean for judgment, integration, and review. |
| **observability** | Producing telemetry worth having — metrics, traces, error capture, health-check endpoints, and alerts that page on user-facing symptoms. |
| **performance** | Performance work as measurement discipline — baseline first, one change at a time, beat the noise, and a keep/revert verdict where neutral is a revert. |
| **pr-conventions** | Author PR descriptions from the right per-change-type template (the repo's own PULL_REQUEST_TEMPLATE wins), embed issue-tracker links, keep the description current as fixes land, and reply to human and bot review threads. |
| **safe-migrations** | Zero-downtime schema and data migration discipline — expand → migrate → contract, never destructive in the release that ships the code, backfills as jobs, every step reversible. |
| **security** | Build-time security discipline — threat-model before controls, Always/Ask-First/Never boundary tiers, SSRF, supply chain, LLM/agent security, privacy. |
| **self-review** | Surface the riskiest parts of the current diff so a reviewer knows where to look hardest — one inline pass on small diffs, parallel review lenses with adversarial synthesis on substantial ones. |
| **seo** | SEO for anything served as a web page — the plumbing (templates, routes, redirects, `<head>` metadata, JSON-LD, sitemaps, robots.txt) and the prose (posts, guides, landing copy, docs) so pages rank and get cited by AI answer engines. |
| **tdd** | Test-driven development — the red-green-refactor discipline for code that's correct by design, not by accident. |
| **ticket-lifecycle** | The ticket-as-control-plane rulebook — the single home for the tracker-operations contract, tracker resolution, the type/priority taxonomy, human-only trigger authorization, derived status, and trust boundaries. |
| **ticket-triage** | Turn one raw ticket into an implementation-ready spec — or ask the smallest unblocking question — without writing production code. |
| **typography** | Web typography — typeface choice and pairing, variable fonts and OpenType features, type scales, line-height, letter-spacing, measure, wrapping, truncation, underlines, tabular numbers, iOS input zoom. |
| **ui-polish** | Design-engineering details that make an interface feel polished — border radius, optical alignment, shadows and elevation, animations and micro-interactions, press feedback, icons. |
| **ux-writing** | UX writing and interface copy — voice and tone, button and link labels, error messages, settings labels, empty states, placeholders. |

Every skill is also directly invocable by name (`/sonu:<skill>`) — the same methodology on demand instead of auto-fired. The ones designed as entry points, with argument hints, include `/sonu:tdd`, `/sonu:design-tree`, `/sonu:self-review`, `/sonu:interface-review`, `/sonu:ticket-triage`, `/sonu:classify-tickets`, `/sonu:bug-finder`, `/sonu:performance`, and `/sonu:intent-interview`.

## Developing this plugin

Contributor tooling is **repo-local, not shipped** — it lives in this repo's `.claude/` directory, so it loads automatically for anyone working *on* the plugin (a clone of this repo) and never reaches marketplace users, who'd have no use for it:

- **`plugin-dev` skill** (`.claude/skills/plugin-dev/`) — the maintainer's handbook: the architecture contract (which decisions are load-bearing and why), the five house rules, plugin/skill/command mechanics, the house authoring shape, and the trap catalogue mined from this repo's own incident history. It fires automatically when you edit any component here.
- **`/validate`** (`.claude/commands/validate.md`) — the CI this repo doesn't have: manifests parse and stay in sync, the marketplace description names every shipped component, frontmatter parses, every embedded shell snippet passes `bash -n` **and** `zsh -n`, no named sources, no AI attribution, ship's bot-registry copies identical, cross-references resolve, README inventory complete. Run it before any PR here.
- **`/release`** (`.claude/commands/release.md`) — the five-home version sync: semver decision, the description sync across both `plugin.json` and both `marketplace.json` files plus this README, `/validate`, hand-off to `/sonu:ship`, and the post-merge tag. Exists because both failure modes it prevents have actually happened here.

## Requirements

- The [`gh`](https://cli.github.com/) CLI, authenticated (`gh auth status`).
- A GitHub remote on the repo.
- For Claude reviews: the `/code-review` and `/security-review` commands.
- Any AI reviewer bots are picked up automatically if they're enabled on the repo — nothing to configure here.

For `/sonu:factory`, per tracker:

| Tracker | Needs |
|---|---|
| GitHub Issues | Nothing beyond the `gh` CLI above. |
| Jira | The Atlassian MCP server, or `JIRA_SITE`, `JIRA_EMAIL`, and `JIRA_API_TOKEN` in the environment. |
| Linear | The Linear MCP server, or `LINEAR_API_KEY` in the environment. |
| Local files | Nothing — just git. |
| Custom | Whatever your generated adapter declares. |

Credentials are read from the environment only. They never go in `.sonu/factory-config.md`, an adapter file, or ticket text — all three get committed.

## A note on trust

`/sonu:ship` runs shell commands and **merges PRs on your behalf**. Read the command before you install it (`sonu/commands/ship.md`), and only point it at repos where you're comfortable with that. Plugins run with your local permissions.

`/sonu:factory` adds a second thing worth understanding before you use it: **whoever can apply a trigger label is the trust boundary** — not whoever opened the ticket. Applying `factory-ready-to-implement` authorizes an agent to build that ticket, and applying `factory-ready-to-ship` authorizes it to **merge** that ticket's branch — so only use the queue on repos where everyone able to label is trusted, and gate the ship label as tightly as merge rights. Don't point it at a public repo where outside contributors can label.

Two honest caveats about how that boundary is held. "Only humans apply triggers" is a rule the workflows follow, not a permission the tracker enforces — an agent holding a credential that can *remove* a label can also add one. Where your tracker supports it, give the agent a credential that can't write trigger labels, and keep the default branch protected with required checks — that backstop catches a bad build authorization (it still ends at a PR you have to merge), though not a bad ship authorization, which is exactly why the ship label is merge authority and scoped like it. And ticket bodies, comments, and linked PRs are treated as untrusted data throughout — they inform the work, and instructions embedded in them are ignored however authoritative they sound — but that too is enforced by instruction rather than by a sandbox, which is the argument for reading a spec before you approve it. The only merge path is `/sonu:ship` — run by you directly, or by a factory ship pass you authorized by applying the ship label to a ticket whose build provably finished.

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
    │   ├── ship.md          # /sonu:ship — PR babysitter
    │   └── factory.md       # /sonu:factory — ticket queue: scan, claim, worktree, delegate to build
    └── skills/              # auto-applied; tdd, design-tree, self-review, interface-review, and the ticket skills also invoke directly as /sonu:<name>
        ├── code-standards/
        │   ├── SKILL.md     # how code gets written
        │   └── references/  # data & API examples, security examples
        ├── tdd/
        │   ├── SKILL.md     # test-driven development — red-green-refactor
        │   └── references/  # worked code for every rule — the loop, AAA, seams, thresholds
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
        │   ├── SKILL.md     # technical + editorial SEO for anything served as a web page
        │   └── references/  # before/after examples + the llms.txt aside
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
        │   └── references/  # focus & keyboard, semantics & ARIA, forms, screen readers, hit areas, motion & zoom
        ├── layout/
        │   ├── SKILL.md     # grouping, alignment, reading order, breakpoints, direction-aware structure
        │   └── references/  # grouping & alignment, spacing & adaptivity
        ├── ui-polish/
        │   ├── SKILL.md     # radius, elevation, motion, press feedback, icons
        │   └── references/  # surfaces, animations, icons, performance
        ├── typography/
        │   ├── SKILL.md     # type scale, spacing, wrapping, truncation, text detailing
        │   └── references/  # choosing fonts, variable fonts & OpenType, spacing & sizing, wrapping & punctuation, details, CSS cheat sheet
        ├── colors/
        │   ├── SKILL.md     # OKLCH palettes, contrast measurement, gamut, one meaning per color
        │   └── references/  # conversion, palette generation, contrast, gamut & Tailwind, usage
        ├── ux-writing/
        │   └── SKILL.md     # interface copy — labels, errors, empty states, settings, voice
        ├── design-tree/
        │   └── SKILL.md     # design by branching tree, not linear narrative
        ├── model-tiering/
        │   └── SKILL.md     # grade plan steps for cheaper model tiers; verify their output up top
        ├── self-review/
        │   ├── SKILL.md     # riskiest things in the diff — parallel lenses + adversarial synthesis; pointer, not a score
        │   └── references/  # lens dispatch templates, worked output examples
        ├── pr-conventions/
        │   ├── SKILL.md     # per-type PR templates, living description, reply wording
        │   └── references/  # the 8 per-change-type PR body templates
        ├── ticket-lifecycle/
        │   ├── SKILL.md     # ticket-as-control-plane rulebook — taxonomy, triggers, claim rules, tracker resolution
        │   └── references/  # one adapter per tracker (github, jira, linear, local, custom) + the optional liveness Action
        ├── ticket-triage/
        │   └── SKILL.md     # raw ticket → implementation-ready spec; never implements
        ├── classify-tickets/
        │   └── SKILL.md     # backlog sweep — one type, one evidence-based priority, nothing else
        └── bug-finder/
            └── SKILL.md     # prove one real defect, file one ticket; never fixes it
```

## Troubleshooting

| Symptom | Likely cause → fix |
|---|---|
| A `/sonu:` command doesn't appear in the slash menu | Plugin not installed on this machine, or the session predates the install → re-run the two install commands, then start a new session. |
| A skill didn't fire when it should have | Skills trigger off their `description` matching the task. Name it explicitly ("apply the code-standards skill") — and if that keeps happening for a legitimate task, the description needs richer triggers: file an issue or PR. |
| Behavior doesn't match this README | Installed copies are **version-pinned** — they don't track `main`. Run `/plugin marketplace update prabhdeep-tools`, then compare your installed version against the latest release tag. |
| `/plugin marketplace update` says up-to-date but a fix you saw on `main` is missing | That fix hasn't been released yet (no version bump). Nothing propagates until a release — see `.claude/commands/release.md` in this repo. |
| `/sonu:ship` stalls waiting for bots | The repo may simply have no AI reviewer bots enabled; the wait loop times out (~10 min) and proceeds with whoever posted. That's expected, not a hang. |
| `/sonu:factory` says no tracker is configured | Neither `.sonu/factory-config.md` nor `~/.sonu/factory-config.md` exists → run `/sonu:factory init`, or write the file yourself with a `tracker:` value. It stops rather than guessing on purpose. |
| I applied the ship label, the PR opened, then nothing — later passes say another session is already shipping it | The pass ran out of turn (its waits outlive a headless session). The next `/sonu:factory` pass resumes it once the heartbeat has been quiet past the ship-idle threshold — 5 minutes if the pass parked deliberately, 30 if it just stopped. Re-applying the label forces it immediately: a trigger outranks liveness, so a triggered ticket always routes. |
| Every ship pass makes one more small fix commit and never merges | An older ship re-initialized its resume ledger on each run, so the pre-PR review loop's 3-pass cap never accumulated and every invocation re-reviewed the whole branch. Current behavior: the ledger is adopted when it survives, the cap counts per PR, a resumed run reviews only the delta, and cosmetic findings are justified rather than committed. Run `/plugin marketplace update prabhdeep-tools` if you're seeing the old loop. |
| `/sonu:factory` finds an empty queue | Nothing carries a trigger. Apply `factory-ready-for-spec` (or `factory-ready-to-implement` on an approved spec, or `factory-ready-to-ship` on a reviewed build) to a ticket — a label on GitHub/Jira/Linear, the `trigger:` field on a local ticket file. Only a human authorizes a pass. |

## License

[MIT](LICENSE) © Prabhdeep Singh
