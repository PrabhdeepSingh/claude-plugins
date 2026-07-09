# claude-plugins

Prabhdeep (Sonu) Singh's personal [Claude Code](https://claude.com/claude-code) commands, packaged as a marketplace so they work on any repo and any machine I sign into — and so I can share them.

## Install

### Claude Code

```
/plugin marketplace add PrabhdeepSingh/claude-plugins
/plugin install sonu@prabhdeep-tools
```

Run those once per device. After that, `/sonu:build`, `/sonu:ship`, `/sonu:tdd`, `/sonu:design-tree`, and `/sonu:self-review` are available in every repo on that machine, and the **code-standards**, **tdd**, **debugging**, **blast-radius**, **safe-migrations**, **infra-standards**, **observability**, **seo-standards**, **content-seo**, **design-tree**, **self-review**, and **pr-conventions** skills ride along automatically — no command to run, they just shape how code and content get written. To pull updates later:

```
/plugin marketplace update prabhdeep-tools
```

> Commands from a plugin are namespaced as `/<plugin>:<command>` — that's why it's `/sonu:ship`, not `/ship`. The slash menu autocompletes, so typing `ship` finds it.

### Cursor

1. Open **Cursor Settings → Plugins**.
2. Add a custom marketplace pointing at this repo: `PrabhdeepSingh/claude-plugins`.
3. Find the **sonu** plugin and click **Install**.

After that, skills auto-apply in every session and the `/sonu:*` commands appear in Cursor's slash-command menu. Updates are pulled when you sync the marketplace in Cursor Settings. (One difference: Cursor has no Claude Code-style plan mode, so `/sonu:build`'s design gate runs in-chat there — the command adapts automatically.)

## Commands

### `/sonu:build` — decide → build → hand back

The spine of the plugin — a thin conductor that sequences the whole implementation lifecycle so the individual skills work together as a pipeline instead of one-off tools.

What it does:

1. **Triage** the working tree — size (trivial vs. substantial), kind (bug vs. feature), surface (web → SEO bars; schema/data → safe-migrations; IaC/containers/CI → infra-standards; new service/endpoint/job → observability; changed data shape another component consumes → blast-radius). One line.
2. **Design**, in real plan mode — loads `code-standards` (plus the surface-matched bars) as design constraints first, then runs `design-tree` so you interview first and tree the real decision points. The plan must meet design-tree's executor-ready bar — exact paths, conventions settled in place, a verification check per step — before it goes to the gate. **`ExitPlanMode` is the approval gate.** Trivial changes skip this entirely.
3. **Build test-first** — runs `tdd` under `code-standards` and every bar the triage flagged; **runs the suite via Bash** to confirm green. Never takes green on faith.
4. **Self-review + hand back** — lists the 3–5 riskiest things in the diff, then stops: *"Green and ready. Review the diff, then run `/sonu:ship`."* Never commits or merges.

Two human checkpoints: approve the design (ExitPlanMode), then review the diff and choose to run ship. Everything in between is autonomous.

```
/sonu:build                                # build whatever's in context
/sonu:build add cart checkout flow         # build a specific feature
/sonu:build fix the off-by-one in totals   # drive a specific fix
```

### `/sonu:ship` — PR babysitter

Takes a finished change from working tree to a clean, merged PR — autonomously. Once you run it, it doesn't stop to ask "shall I merge?"; it only pauses for a real judgment call.

What it does:

1. **Branch, commit, open a PR** with the right per-change-type description (feature / bugfix / hotfix / chore / refactor / docs / perf / release) — reusing the repo's own `PULL_REQUEST_TEMPLATE` if one exists. No AI-attribution trailers — commits and the PR body read as your own.
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

### `/sonu:tdd` — drive a change test-first

Runs the red-green-refactor loop on a named feature, bug, or behavior. Invokes the `tdd` skill directly.

```
/sonu:tdd                          # apply test-first methodology to the current change
/sonu:tdd cart checkout flow       # drive a specific feature test-first
/sonu:tdd fix the off-by-one bug   # reproduce with a failing test, then fix
```

The `tdd` skill (below) auto-applies the same methodology whenever code is written or changed, without needing an explicit invocation.

### `/sonu:design-tree` — design tree mapper

Maps any design problem as an explicit branching tree instead of one linear narrative. Pair it with `/sonu:ship` for complete PR lifecycle coverage: tree the design first, then ship the implementation.

What it does:

1. **Interviews you** to reach shared understanding — intent, constraints, success criteria, non-goals — before mapping a single decision point. This is the highest-leverage step, and it's always first.
2. **Finds the real forks** — only decision points where the design could genuinely go in ≥2 consequential ways. House standards load first as pre-decided constraints, so a fork they already settle is stated and cited, not treed.
3. **Records every branch**: chosen option with the decisive reason, rejected options with the reason each lost.
4. **Preserves the rejected branches** so decisions don't get silently relitigated and you have a real fork to backtrack to if a downstream choice invalidates an earlier one.
5. **Folds into the plan file** when in plan mode (as a `## Design Tree` section), or prints in-chat when called standalone.

```
/sonu:design-tree                  # tree the current design or active plan
/sonu:design-tree auth system      # tree a specific topic
```

The `design-tree` skill (below) auto-applies the same methodology in plan mode without needing an explicit invocation.

### `/sonu:self-review` — where should a reviewer look?

Runs the `self-review` skill on demand: the 3–5 riskiest spots in the current diff (untracked files and multi-commit branches included), in plain language, ending with an explicit "this is a pointer, not an approval." `/sonu:build` and `/sonu:ship` already run it automatically at the right moments — this command is for everywhere else.

## Skills

### `code-standards` — code the way I do

A skill, not a command — there's nothing to invoke. Once the plugin is installed, Claude consults it automatically before writing, generating, or refactoring code in any repo, so AI-written code lands in my style instead of generic boilerplate.

It opens with **working discipline** — how to approach the task, not just the output: think before coding and surface assumptions, build the minimum that solves the problem, look for an existing implementation (codebase, stdlib, dependencies) before writing a fresh one, make surgical changes that trace to the request (reverting dead ends completely), and turn vague asks into verifiable goals — claiming only outcomes you actually observed. Then it encodes the foundation across fourteen areas:

- **Naming** — intention-revealing names, no `data`/`temp`/`Manager` junk-drawer names.
- **Schema & API conventions** — `snake_case` fields, UUID ids, `created_date`/`last_modified_date` timestamps in UTC, first/last name stored separately.
- **Readability** — flat guard-clause control flow over nested `if`s, comments that explain *why*.
- **Modularity** — small single-purpose functions, separation of concerns.
- **Data access** — select only the columns you need, paginate by default, no unbounded loads, no N+1.
- **Presentation/logic/data separation** — no inline styles, no magic numbers or strings.
- **Error handling** — fail loudly, never swallow errors.
- **Logging** — through one shared logger (never raw `console.log`), structured, filterable, no secrets.
- **Input validation & injection** — validate untrusted input at the boundary, parameterize every query.
- **Information leaks** — generic API errors with detail logged internally, no auth/account enumeration.
- **State** — immutability by default, tight scope.
- **Tooling diagnostics** — fix the cause, never bare-suppress (`as any`, `@ts-ignore`, `eslint-disable` need a narrow scope and a stated reason).
- **API design** — responses built from explicit field allowlists (never a serialized entity — no leaked hashes/tokens/internal flags), honest status codes, one error shape, pagination from day one, staged deprecation for breaking changes.
- **Configuration & flags** — absence must be safe: a missing or malformed env var/flag never silently enables behavior; feature gates default off, protective controls fail fast; `true` defaults are explicit commented decisions; resolved flags logged at startup.

Every rule explains *why* it's there, ships with good/bad examples so it actually sticks, and ends with a self-check the model runs against its own diff. When it's editing an existing codebase, matching that codebase's conventions wins over the guide.

Edit `sonu/skills/code-standards/SKILL.md` to make it yours — it's plain Markdown.

### `tdd` — test-driven development, baked in

A skill, not a command — there's nothing to invoke. Once the plugin is installed, Claude follows the red-green-refactor discipline automatically whenever it writes, changes, or tests code in any repo — even when "TDD" or "tests" aren't mentioned. It also fires on the explicit `/sonu:tdd` command.

It encodes a strict test-first methodology with honest carve-outs (spikes are thrown away and rebuilt test-first; code never lands without tests) across twelve areas:

- **Red-green-refactor** — failing test first, minimum code to green, refactor under protection. Small steps, run tests constantly.
- **Test-first discipline** — the one carve-out: exploratory spikes to learn a shape, discarded entirely before building the real thing test-first.
- **Behavior not implementation** — assert observable outcomes, never private state or internal call counts, so refactors don't break tests.
- **Arrange-Act-Assert** — one behavior per test, three clean phases, one reason to fail.
- **Spec-sentence naming** — test names document what broke and under what condition; the suite reads as a specification.
- **Test qualities** — fast (milliseconds), isolated (no shared mutable state, no ordering), deterministic (injected clock/seed, no real I/O in unit tests), self-validating.
- **Test doubles** — mock only at architectural seams (database, network, clock); real domain objects throughout the core; no mock returning a mock.
- **The testing pyramid** — many unit tests, fewer integration, fewest end-to-end; push behavior down to the unit level.
- **Coverage as byproduct** — use it to find gaps, not to hit a number; a test with no meaningful assertion is negative value.
- **What to test** — behavior, boundaries, edge cases, error paths; thresholds (limits, timeouts, caps) at values a test can actually trip, asserting both sides; skip trivial pass-throughs and generated code.
- **The bug-fix reflex** — reproduce the bug with a failing test before fixing it, every time.
- **The test is innocent** — a failing test means the code is wrong, not the test; no updating expectations to match broken output, no skips, no broadened assertions, no sleeps.

Every rule explains the *why*, ships with Avoid/Prefer code examples, and the red-green-refactor section shows the full three-step sequence end-to-end. Tests are held to the same bar as production code via `code-standards`.

Edit `sonu/skills/tdd/SKILL.md` to make it yours — it's plain Markdown.

### `debugging` — hypothesis testing, not patch roulette

A skill, not a command — it fires automatically whenever Claude is diagnosing anything broken: an error message, a stack trace, a failing or flaky test, a crash, unexpected output, a regression, a production incident. When the bug is a production report, it **pulls the real event instead of debugging the paraphrase** — discovering from the repo whether the project uses Sentry, Datadog, Azure Application Insights, CloudWatch, or plain logs, fetching the exact exception, breadcrumbs, frequency, and first-seen release (via MCP server, API, or CLI — or asking for access rather than improvising), and treating what it finds as production data that never leaks into code, commits, or PRs. It encodes the scientific debugging loop: **reproduce first** (no reproduction, no fix), **read the actual error** (verbatim, first-error-in-the-log, no pattern-matched diagnoses), **locate the origin, not the surface** (trace the bad state back to where it was made wrong — never patch where it exploded), **one hypothesis → one change → one observation** (never two variables at once), **instrument and bisect** instead of guessing, **revert dead ends completely** (no fossils of failed attempts under the final fix), **prove the fix** (the reproduction passes AND you can say why in one sentence, pinned with a regression test via `tdd`), and **know when to stop** (three dead hypotheses → reframe; escalate with a structured summary of what's ruled out).

Edit `sonu/skills/debugging/SKILL.md` to make it yours — it's plain Markdown.

### `blast-radius` — who reads the thing you're changing?

A skill, not a command — it fires automatically whenever a change alters the shape, format, or semantics of anything other code consumes: a function's return value, an API/tool response body, a serialized payload, a DB column read elsewhere, a log or telemetry field, an event message, a config value, or parsed CLI output. Wrapping counts; renaming counts; "the data is still there, just enveloped" counts.

It encodes consumer-impact discipline in six steps: **name the seam** (an unnamed contract can't be searched for), **enumerate consumers mechanically** — by grepping for symbols, field names, and parse sites, never from memory, and always including the out-of-band readers (loggers, telemetry, ETL, dashboards) that call-graph intuition structurally misses — **classify each consumer by how it fails** (unaffected / breaks loudly / *degrades silently* — the killer class, hunted via `catch → default`, `|| []`, `?? null` downstream of the seam), **disposition every affected consumer explicitly** (update it, version the contract expand-→-migrate-→-contract style, or accept the break in writing), **verify one downstream path end-to-end** by observing real output — a logged row, an emitted event — not just the producer's green tests, and **record the downstream impact** where the reviewer will see it.

The motivating failure mode: a locally-correct, fully-tested change to a producer's output format that silently nulls out a downstream parser's data for weeks, because the parser's fallback made breakage look like missing data. This skill makes "who reads this?" a mandatory step instead of an instinct.

Edit `sonu/skills/blast-radius/SKILL.md` to make it yours — it's plain Markdown.

### `safe-migrations` — the schema change and the safe path to it are different artifacts

A skill, not a command — it fires automatically whenever Claude writes or edits a database migration, an `ALTER TABLE`, a backfill, or any change where code and schema move together. It encodes zero-downtime discipline: every migration stays compatible one release in each direction (rolling deploys mean old code meets new schema), breaking changes decompose into **expand → migrate → contract** across separate releases, destructive operations ship alone one release late, backfills run as batched/resumable/idempotent jobs (never inside the deploy), every step has a tested down path or an explicit `IRREVERSIBLE` marker, lock-aware DDL forms, and rehearsal against production-shaped data.

### `infra-standards` — infrastructure is code: reviewed, planned, least-privileged, boring

A skill, not a command — it fires automatically on any infra surface: Terraform/Bicep/CloudFormation, Dockerfiles, CI pipelines (GitHub Actions, Azure Pipelines), Vercel config, env/secrets handling. It encodes: no clickops (console changes are drift; codify or they don't exist), read the plan before apply (every `destroy`/`replace` line explained), secrets only from secret stores (never in source, tfvars, images, or pipeline logs), Dockerfile baselines (pinned bases, multi-stage, non-root, cache-aware layers, pinned deploy tags), CI discipline (build once and promote, scoped tokens, pinned actions — and **never green a pipeline by weakening it**), least privilege everywhere, and idempotency as the IaC contract.

### `observability` — instrument for the question you'll ask at 2am

A skill, not a command — it fires automatically when creating a service, endpoint, or job, or adding metrics/tracing/error capture/health checks/alerts. It encodes the producing side of what `debugging` consumes: the four questions every operation must answer from telemetry alone (traffic, errors, latency, saturation), the metric cardinality trap (no unbounded labels), trace-context propagation, liveness-vs-readiness health endpoints that don't cause restart storms, error capture tagged with the release (what makes first-seen bisection possible), and alert quality — page on user-facing symptoms only, every alert actionable and owned, delete what gets ignored. Instrumentation ships with the feature, not after the incident.

### `seo-standards` — technical SEO, baked in

The HTML/template/plumbing side of SEO. Like `code-standards`, there's nothing to invoke — Claude reaches for it automatically whenever it touches anything served as a web page or that affects how one is crawled: page/component templates (HTML, JSX/TSX, Vue, Svelte, Astro), route and URL definitions, redirects, `<head>` metadata (title, meta description, canonical, robots, hreflang, Open Graph), schema.org JSON-LD, sitemaps, and robots.txt.

It covers heading structure (one `<h1>` per URL, headings for hierarchy not nav), title/meta length, canonical tags, URL strategy, redirect rules (301/302/410), structured data, render-blocking JS/CSS, and indexation controls — every rule optimizing for the crawler and the human reader at once. The point: correct SEO is far cheaper to bake in at build time than to retrofit after launch.

Edit `sonu/skills/seo-standards/SKILL.md` to tune it.

### `content-seo` — write it so humans rank it and AI cites it

The editorial counterpart to `seo-standards`: that one governs the plumbing, this one governs the *writing*. It fires automatically whenever Claude writes or edits prose meant to be published — blog posts, articles, guides, tutorials, landing/marketing copy, press releases, changelog and "what's new" entries, FAQs, docs, and Markdown content (`content/**/*.md`, `*.mdx`) plus its frontmatter.

It encodes modern on-page SEO: start from a single search intent, structure a machine can extract, real depth and E-E-A-T over keyword stuffing, internal linking, image alt text, URL slugs, and featured-snippet / AI-citation formatting — so content doesn't just rank in the traditional top 10 but earns citations in AI answer engines (Google AI Overviews, ChatGPT, Claude, Perplexity, Gemini).

Edit `sonu/skills/content-seo/SKILL.md` to tune it.

### `pr-conventions` — right template, living description, honest replies

A skill, not a command — there's nothing to invoke. Once the plugin is installed, Claude uses it automatically inside `/sonu:ship` to author PR descriptions, keep them current, and reply to review threads. Also callable standalone when you're crafting a PR body or responding to comments outside the ship flow.

What it does:

1. **Discovers the team's own template first** — scans for `.github/PULL_REQUEST_TEMPLATE.md` (and the multi-template directory variant) before reaching for any built-in. The team standard wins; built-ins are the fallback.
2. **Picks the right built-in template** from eight types: feature, bugfix, hotfix, chore, refactor, docs, perf, release — detected from the branch name, conventional-commit prefix on the commits, or the diff. Each template includes the sections that matter for that change type and nothing else.
3. **Keeps the description current** as fixes land and re-reviews cycle through — refreshing Summary/Changes bullets and the Risk section in-place so re-reviewers see the actual state, not the opening snapshot.
4. **Supplies reply templates** for every review-thread scenario (fixed / justified / false-positive / partial / question). Bot threads get replied to and resolved; human threads get replied to and left open — presumptuously closing a person's feedback thread is not this skill's call.

Edit `sonu/skills/pr-conventions/SKILL.md` to tune the templates or add new change types.

### `self-review` — point attention at the riskiest parts

A skill, not a command — there's nothing to invoke. Once the plugin is installed, Claude runs it automatically at two moments: before handing back from `/sonu:build` (so you know where to look before you run `/sonu:ship`), and before creating the PR in `/sonu:ship` (so the riskiest items are embedded in the PR body for traceability and surfaced in the final report).

What it produces: a plain-language list of the **3–5 spots in the diff** that a reviewer should look hardest at — subtle logic, security-relevant surfaces, data integrity risk, broad blast radius, untested edges, silent behavior changes. One line per item with `file:line` where helpful.

What it explicitly is **not**: a score, a grade, a gate, or an approval. Self-scoring rubber-stamps the model's own work; the value is directing *your* eyes to the corners that will otherwise get skimmed. The list ends with a plain statement to that effect.

The same reasoning applies when you ask for a self-review manually — "what should I look at?", "what's risky here?", "self-review this." If the diff is genuinely low-risk, it says so rather than inventing items to fill the list.

### `design-tree` — decide by branching, not by marching

A skill, not a command — there's nothing to invoke in plan mode, it fires automatically when you're designing or planning an implementation approach. The explicit counterpart is `/sonu:design-tree` (above), which you can call manually at any time.

It encodes a design methodology built around one core idea: design is traversing a branching tree, not marching a line. What that looks like in practice:

- **Interview first.** Before branching anything, ask 2–4 targeted questions to confirm intent, constraints, success criteria, and non-goals. Designing the right problem saves more context and tokens than anything else.
- **Load the standards as constraints.** `code-standards` always, plus `safe-migrations`/`seo-standards`/`content-seo`/`infra-standards`/`observability` when the surface matches — a fork a standard already settles is stated and cited, never treed. The plan-vs-standards conflict gets resolved at design time, not discovered by the executor mid-build.
- **Find the real forks.** Only decision points where the design could genuinely go in ≥2 consequential ways — not trivia, not forced choices.
- **Enumerate genuine alternatives** at every fork. No strawmen invented to be knocked down.
- **Record the chosen branch** with a decisive reason (a real constraint, trade-off, or irreversibility) — not a vague preference.
- **Keep the rejected branches** with the reason each lost. Stops silent relitigation; preserves real forks to return to.
- **Backtrack deliberately** to a recorded fork when a downstream decision invalidates an earlier choice, rather than patching forward.

The tree is written as a compact nested-bullet notation (`✓ chosen — reason`, `✗ rejected — why`) that's scannable in seconds. In plan mode it becomes a `## Design Tree` section in the plan file, with a `Constraints:` line naming the standards the design was drawn under — and the plan itself must be **executor-ready**: exact file paths, conventions settled in place, a verification check per step, no judgment call left for a smaller model in a fresh session to guess at.

Edit `sonu/skills/design-tree/SKILL.md` to make it yours — it's plain Markdown.

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

## A note on trust

`/sonu:ship` runs shell commands and **merges PRs on your behalf**. Read the command before you install it (`sonu/commands/ship.md`), and only point it at repos where you're comfortable with that. Plugins run with your local permissions.

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
    │   ├── design-tree.md   # /sonu:design-tree
    │   ├── tdd.md           # /sonu:tdd
    │   └── self-review.md   # /sonu:self-review
    └── skills/              # auto-applied skills (nothing to invoke)
        ├── code-standards/
        │   └── SKILL.md     # how code gets written
        ├── tdd/
        │   └── SKILL.md     # test-driven development — red-green-refactor
        ├── debugging/
        │   └── SKILL.md     # scientific debugging — reproduce, one hypothesis, revert dead ends
        ├── blast-radius/
        │   └── SKILL.md     # consumer impact before changing any consumed contract
        ├── safe-migrations/
        │   └── SKILL.md     # zero-downtime schema changes — expand → migrate → contract
        ├── infra-standards/
        │   └── SKILL.md     # IaC, Docker, CI/CD, secrets — plan before apply
        ├── observability/
        │   └── SKILL.md     # metrics, traces, health checks, alerts worth paging on
        ├── seo-standards/
        │   └── SKILL.md     # technical SEO for web pages
        ├── content-seo/
        │   └── SKILL.md     # editorial SEO for published prose
        ├── design-tree/
        │   └── SKILL.md     # design by branching tree, not linear narrative
        ├── self-review/
        │   └── SKILL.md     # 3–5 riskiest things in the diff — pointer, not a score
        └── pr-conventions/
            └── SKILL.md     # per-type PR templates, living description, reply wording
```

## Troubleshooting

| Symptom | Likely cause → fix |
|---|---|
| A `/sonu:` command doesn't appear in the slash menu | Plugin not installed on this machine, or the session predates the install → re-run the two install commands, then start a new session. |
| A skill didn't fire when it should have | Skills trigger off their `description` matching the task. Name it explicitly ("apply the code-standards skill") — and if that keeps happening for a legitimate task, the description needs richer triggers: file an issue or PR. |
| Behavior doesn't match this README | Installed copies are **version-pinned** — they don't track `main`. Run `/plugin marketplace update prabhdeep-tools`, then compare your installed version against the latest release tag. |
| `/plugin marketplace update` says up-to-date but a fix you saw on `main` is missing | That fix hasn't been released yet (no version bump). Nothing propagates until a release — see `.claude/commands/release.md` in this repo. |
| `/sonu:ship` stalls waiting for bots | The repo may simply have no AI reviewer bots enabled; the wait loop times out (~10 min) and proceeds with whoever posted. That's expected, not a hang. |

## License

[MIT](LICENSE) © Prabhdeep Singh
