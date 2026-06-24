# claude-plugins

Prabhdeep (Sonu) Singh's personal [Claude Code](https://claude.com/claude-code) commands, packaged as a marketplace so they work on any repo and any machine I sign into — and so I can share them.

## Install

### Claude Code

```
/plugin marketplace add PrabhdeepSingh/claude-plugins
/plugin install sonu@prabhdeep-tools
```

Run those once per device. After that, `/sonu:build` and `/sonu:ship` are available in every repo on that machine, and the **code-standards**, **tdd**, **seo-standards**, **content-seo**, **design-tree**, and **self-review** skills ride along automatically — no command to run, they just shape how code and content get written. To pull updates later:

```
/plugin marketplace update prabhdeep-tools
```

> Commands from a plugin are namespaced as `/<plugin>:<command>` — that's why it's `/sonu:ship`, not `/ship`. The slash menu autocompletes, so typing `ship` finds it.

### Cursor

1. Open **Cursor Settings → Plugins**.
2. Add a custom marketplace pointing at this repo: `PrabhdeepSingh/claude-plugins`.
3. Find the **sonu** plugin and click **Install**.

After that, skills auto-apply in every session and `/sonu:build`, `/sonu:ship`, `/sonu:tdd`, and `/sonu:design-tree` appear in Cursor's slash-command menu. Updates are pulled when you sync the marketplace in Cursor Settings.

## Commands

### `/sonu:build` — decide → build → hand back

The spine of the plugin — a thin conductor that sequences the whole implementation lifecycle so the individual skills work together as a pipeline instead of one-off tools.

What it does:

1. **Triage** the working tree — size (trivial vs. substantial), kind (bug vs. feature), surface (web → SEO bars). One line.
2. **Design**, in real plan mode — runs `design-tree` so you interview first and tree the real decision points. **`ExitPlanMode` is the approval gate.** Trivial changes skip this entirely.
3. **Build test-first** — runs `tdd` under `code-standards` (and SEO bars when relevant); **runs the suite via Bash** to confirm green. Never takes green on faith.
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

1. **Branch, commit, open a PR** (no AI-attribution trailers — commits read as your own).
2. **Gathers every review source**: its own Claude `/code-review` + `/security-review`, plus **every AI reviewer bot enabled on the repo** — detected by who actually posts on the PR (Copilot, CodeRabbit, Aikido, Qodo, Greptile, Ellipsis, Sourcery, Cubic, Korbit, …). No config needed; it adapts per-repo. Copilot is requested automatically since it's the one that doesn't auto-fire.
3. **Dedups, fixes, or justifies** every finding, then **replies to and resolves** each bot's review threads.
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
2. **Finds the real forks** — only decision points where the design could genuinely go in ≥2 consequential ways.
3. **Records every branch**: chosen option with the decisive reason, rejected options with the reason each lost.
4. **Preserves the rejected branches** so decisions don't get silently relitigated and you have a real fork to backtrack to if a downstream choice invalidates an earlier one.
5. **Folds into the plan file** when in plan mode (as a `## Design Tree` section), or prints in-chat when called standalone.

```
/sonu:design-tree                  # tree the current design or active plan
/sonu:design-tree auth system      # tree a specific topic
```

The `design-tree` skill (below) auto-applies the same methodology in plan mode without needing an explicit invocation.

## Skills

### `code-standards` — code the way I do

A skill, not a command — there's nothing to invoke. Once the plugin is installed, Claude consults it automatically before writing, generating, or refactoring code in any repo, so AI-written code lands in my style instead of generic boilerplate.

It opens with **working discipline** — how to approach the task, not just the output: think before coding and surface assumptions, build the minimum that solves the problem, make surgical changes that trace to the request, and turn vague asks into verifiable goals. Then it encodes the foundation across eleven areas:

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

Every rule explains *why* it's there, ships with good/bad examples so it actually sticks, and ends with a self-check the model runs against its own diff. When it's editing an existing codebase, matching that codebase's conventions wins over the guide.

Edit `sonu/skills/code-standards/SKILL.md` to make it yours — it's plain Markdown.

### `tdd` — test-driven development, baked in

A skill, not a command — there's nothing to invoke. Once the plugin is installed, Claude follows the red-green-refactor discipline automatically whenever it writes, changes, or tests code in any repo — even when "TDD" or "tests" aren't mentioned. It also fires on the explicit `/sonu:tdd` command.

It encodes a strict test-first methodology with honest carve-outs (spikes are thrown away and rebuilt test-first; code never lands without tests) across eleven areas:

- **Red-green-refactor** — failing test first, minimum code to green, refactor under protection. Small steps, run tests constantly.
- **Test-first discipline** — the one carve-out: exploratory spikes to learn a shape, discarded entirely before building the real thing test-first.
- **Behavior not implementation** — assert observable outcomes, never private state or internal call counts, so refactors don't break tests.
- **Arrange-Act-Assert** — one behavior per test, three clean phases, one reason to fail.
- **Spec-sentence naming** — test names document what broke and under what condition; the suite reads as a specification.
- **Test qualities** — fast (milliseconds), isolated (no shared mutable state, no ordering), deterministic (injected clock/seed, no real I/O in unit tests), self-validating.
- **Test doubles** — mock only at architectural seams (database, network, clock); real domain objects throughout the core; no mock returning a mock.
- **The testing pyramid** — many unit tests, fewer integration, fewest end-to-end; push behavior down to the unit level.
- **Coverage as byproduct** — use it to find gaps, not to hit a number; a test with no meaningful assertion is negative value.
- **What to test** — behavior, boundaries, edge cases, error paths; skip trivial pass-throughs and generated code.
- **The bug-fix reflex** — reproduce the bug with a failing test before fixing it, every time.

Every rule explains the *why*, ships with Avoid/Prefer code examples, and the red-green-refactor section shows the full three-step sequence end-to-end. Tests are held to the same bar as production code via `code-standards`.

Edit `sonu/skills/tdd/SKILL.md` to make it yours — it's plain Markdown.

### `seo-standards` — technical SEO, baked in

The HTML/template/plumbing side of SEO. Like `code-standards`, there's nothing to invoke — Claude reaches for it automatically whenever it touches anything served as a web page or that affects how one is crawled: page/component templates (HTML, JSX/TSX, Vue, Svelte, Astro), route and URL definitions, redirects, `<head>` metadata (title, meta description, canonical, robots, hreflang, Open Graph), schema.org JSON-LD, sitemaps, and robots.txt.

It covers heading structure (one `<h1>` per URL, headings for hierarchy not nav), title/meta length, canonical tags, URL strategy, redirect rules (301/302/410), structured data, render-blocking JS/CSS, and indexation controls — every rule optimizing for the crawler and the human reader at once. The point: correct SEO is far cheaper to bake in at build time than to retrofit after launch.

Edit `sonu/skills/seo-standards/SKILL.md` to tune it.

### `content-seo` — write it so humans rank it and AI cites it

The editorial counterpart to `seo-standards`: that one governs the plumbing, this one governs the *writing*. It fires automatically whenever Claude writes or edits prose meant to be published — blog posts, articles, guides, tutorials, landing/marketing copy, press releases, changelog and "what's new" entries, FAQs, docs, and Markdown content (`content/**/*.md`, `*.mdx`) plus its frontmatter.

It encodes modern on-page SEO: start from a single search intent, structure a machine can extract, real depth and E-E-A-T over keyword stuffing, internal linking, image alt text, URL slugs, and featured-snippet / AI-citation formatting — so content doesn't just rank in the traditional top 10 but earns citations in AI answer engines (Google AI Overviews, ChatGPT, Claude, Perplexity, Gemini).

Edit `sonu/skills/content-seo/SKILL.md` to tune it.

### `self-review` — point attention at the riskiest parts

A skill, not a command — there's nothing to invoke. Once the plugin is installed, Claude runs it automatically at two moments: before handing back from `/sonu:build` (so you know where to look before you run `/sonu:ship`), and before creating the PR in `/sonu:ship` (so the riskiest items are embedded in the PR body for traceability and surfaced in the final report).

What it produces: a plain-language list of the **3–5 spots in the diff** that a reviewer should look hardest at — subtle logic, security-relevant surfaces, data integrity risk, broad blast radius, untested edges, silent behavior changes. One line per item with `file:line` where helpful.

What it explicitly is **not**: a score, a grade, a gate, or an approval. Self-scoring rubber-stamps the model's own work; the value is directing *your* eyes to the corners that will otherwise get skimmed. The list ends with a plain statement to that effect.

The same reasoning applies when you ask for a self-review manually — "what should I look at?", "what's risky here?", "self-review this." If the diff is genuinely low-risk, it says so rather than inventing items to fill the list.

### `design-tree` — decide by branching, not by marching

A skill, not a command — there's nothing to invoke in plan mode, it fires automatically when you're designing or planning an implementation approach. The explicit counterpart is `/sonu:design-tree` (above), which you can call manually at any time.

It encodes a design methodology built around one core idea: design is traversing a branching tree, not marching a line. What that looks like in practice:

- **Interview first.** Before branching anything, ask 2–4 targeted questions to confirm intent, constraints, success criteria, and non-goals. Designing the right problem saves more context and tokens than anything else.
- **Find the real forks.** Only decision points where the design could genuinely go in ≥2 consequential ways — not trivia, not forced choices.
- **Enumerate genuine alternatives** at every fork. No strawmen invented to be knocked down.
- **Record the chosen branch** with a decisive reason (a real constraint, trade-off, or irreversibility) — not a vague preference.
- **Keep the rejected branches** with the reason each lost. Stops silent relitigation; preserves real forks to return to.
- **Backtrack deliberately** to a recorded fork when a downstream decision invalidates an earlier choice, rather than patching forward.

The tree is written as a compact nested-bullet notation (`✓ chosen — reason`, `✗ rejected — why`) that's scannable in seconds. In plan mode it becomes a `## Design Tree` section in the plan file.

Edit `sonu/skills/design-tree/SKILL.md` to make it yours — it's plain Markdown.

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
├── .claude-plugin/
│   └── marketplace.json     # Claude Code marketplace manifest (name: prabhdeep-tools)
├── .cursor-plugin/
│   └── marketplace.json     # Cursor marketplace manifest (same plugin, official Cursor path)
├── LICENSE                  # MIT
└── sonu/                    # the "sonu" plugin (your personal namespace)
    ├── .claude-plugin/
    │   └── plugin.json      # Claude Code plugin manifest
    ├── .cursor-plugin/
    │   └── plugin.json      # Cursor plugin manifest (mirrors the Claude one)
    ├── commands/
    │   ├── build.md         # the /sonu:build command (conductor)
    │   ├── ship.md          # the /sonu:ship command
    │   ├── design-tree.md   # the /sonu:design-tree command
    │   └── tdd.md           # the /sonu:tdd command
    └── skills/              # auto-applied skills (nothing to invoke)
        ├── code-standards/
        │   └── SKILL.md     # how code gets written
        ├── tdd/
        │   └── SKILL.md     # test-driven development — red-green-refactor
        ├── seo-standards/
        │   └── SKILL.md     # technical SEO for web pages
        ├── content-seo/
        │   └── SKILL.md     # editorial SEO for published prose
        ├── design-tree/
        │   └── SKILL.md     # design by branching tree, not linear narrative
        └── self-review/
            └── SKILL.md     # 3–5 riskiest things in the diff — pointer, not a score
```

## License

[MIT](LICENSE) © Prabhdeep Singh
