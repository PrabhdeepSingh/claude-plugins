# claude-plugins

Prabhdeep (Sonu) Singh's personal [Claude Code](https://claude.com/claude-code) commands, packaged as a marketplace so they work on any repo and any machine I sign into — and so I can share them.

## Install

### Claude Code

```
/plugin marketplace add PrabhdeepSingh/claude-plugins
/plugin install sonu@prabhdeep-tools
```

Run those once per device. After that, `/sonu:build`, `/sonu:ship`, `/sonu:factory`, `/sonu:tdd`, `/sonu:design-tree`, `/sonu:self-review`, `/sonu:memory`, `/sonu:ticket-triage`, `/sonu:classify-tickets`, and `/sonu:bug-finder` are available in every repo on that machine, and the **code-standards**, **tdd**, **debugging**, **blast-radius**, **safe-migrations**, **infra-standards**, **observability**, **seo-standards**, **content-seo**, **design-tree**, **model-tiering**, **self-review**, **memory**, **pr-conventions**, **ticket-lifecycle**, **ticket-triage**, **classify-tickets**, and **bug-finder** skills ride along automatically — no command to run, they just shape how code and content get written. To pull updates later:

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

`/sonu:build`, `/sonu:ship`, and `/sonu:factory` are commands proper — they sequence phases and hold gates. `/sonu:tdd`, `/sonu:design-tree`, `/sonu:self-review`, `/sonu:memory`, `/sonu:interface-review`, `/sonu:ticket-triage`, `/sonu:classify-tickets`, and `/sonu:bug-finder` are the skills themselves invoked directly by name: same syntax, same behavior, but no separate command component (a command and a skill can't share a name — they collide on the harness's one invocation surface).

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

#### Poll mode — the queue without the retyping

`/sonu:factory poll` turns the pass into a standing loop: the session sweeps, ships one, builds one, specs what's ready, then idles and wakes again (15–30 minutes, via the harness's loop facility). Authorization doesn't change — you still apply every trigger; poll only saves you re-running the command. There is still no hosted daemon: the loop is your session, on your machine, and it stops when you close it. Passes maintain a heartbeat comment (one comment, edited in place) so a died session is detectable: the sweep — or an optional GitHub Action that `init` offers to install — flags tickets whose pass stopped answering as `factory:agent-lost`, and a polling session may then take the work over by claiming that flag and rebuilding from the spec. Blocked tickets waiting on your answer are never flagged and never taken over — waiting is not death. And once a ticket is shipped and closed (closure is automatic on merge — your gate was applying the ship trigger), the loop drops that ticket's context entirely and starts the next one fresh from its own thread: the ticket is the durable memory, not the session.

#### Parallel work — a worktree per ticket

Every implement pass builds in its own git worktree (`../myrepo-wt-0001-fix-login-loop` on branch `ticket/0001-fix-login-loop`), unconditionally. That's what makes it safe to run several agents at once, each on a different ticket: separate directories, separate branches, no stomping. The claim happens in the main checkout *before* the worktree exists, so two sessions can never build the same ticket. A dirty main checkout is a hard stop rather than something to build around, and the factory sweep cleans up worktrees and branches for tickets whose PRs have merged. In a sandboxed harness that can't write outside the workspace, it falls back to building in place on a clean tree and says so.

Two things worth knowing before you run agents in parallel. A fresh worktree gets a fresh install and **does not inherit untracked local config** — `.env` and friends stay behind, and a suite that quietly skips tests for missing config will report green while proving nothing, so the pass copies what the suite needs. And on the local file tracker, claims are commits: if your agents run on more than one machine, the claim commit has to reach the remote for the other machine to see it, which is why the pass pushes when a remote exists.

#### Tracker configuration

Five backends. Pick per repo, or once for everything:

| `tracker:` | Backend | Notes |
|---|---|---|
| `github` | GitHub Issues via `gh` | Everything is a label; `Closes #N` closes the ticket when the PR merges **to the default branch** (a merge into a release branch won't). |
| `jira` | Jira via the Atlassian MCP or REST | Native type and priority fields; nothing auto-closes, so the sweep transitions to Done. |
| `linear` | Linear via its MCP or GraphQL | Native priority; `Fixes ENG-123` closes on merge **when Linear's GitHub integration is enabled** — without it, the sweep reports the ticket for a manual move. |
| `local` | Markdown files in the repo | Zero dependencies, works offline, tickets diff in PRs. |
| `custom` | Anything else | `init` interviews you and generates the adapter. |

Configuration lives in `.sonu/factory-config.md` in the repo (committed, so the team shares it), falling back to `~/.sonu/factory-config.md` for a tracker you use everywhere:

```markdown
---
tracker: local
---
Notes for humans go below the frontmatter; workflows read only the frontmatter.
```

Credentials never go in that file — Jira and Linear read them from environment variables (see Requirements).

The **local** backend is the one that needs nothing but git. Tickets are files under `.sonu/tickets/`:

```markdown
---
id: 0001
title: Fix login redirect loop
type: bug
priority: P1
trigger: ready-to-implement
status: open
created: 2031-01-15
---
## Problem
## Scope and non-goals
## Acceptance criteria
## Verification plan
## Discussion
```

Ticket-file edits (claims, specs, classifications, status flips) are committed on their own with a `tickets:` prefix, never mixed into a code commit — so the diff you review stays clean, and a claim is durable the moment it happens. That's the one deliberate exception to "these workflows never commit," and it covers tracker metadata only, never source code.

For a tracker that isn't one of the four, `/sonu:factory init` asks how your tool works — how an agent reaches it, which environment variables hold credentials, what marks the three triggers, how type and priority map, how a merge closes a ticket, whether a comment can be edited in place — and writes an adapter file for you to review. Workflows only ever name an operation from a fixed list (list queue, list open, search, fetch, claim, update body, comment, heartbeat, classify, mark status, create, close the loop), so a tracker is fully supported the moment a document answers all of them; an adapter missing one is a hard stop, never an improvised command (the two display/liveness aids, *mark status* and *heartbeat*, degrade gracefully instead).

### `/sonu:tdd` — drive a change test-first

Runs the red-green-refactor loop on a named feature, bug, or behavior. This is the `tdd` skill (below) invoked directly — it writes test and implementation files to the working tree, not a printed plan.

```
/sonu:tdd                          # apply test-first methodology to the current change
/sonu:tdd cart checkout flow       # drive a specific feature test-first
/sonu:tdd fix the off-by-one bug   # reproduce with a failing test, then fix
```

The `tdd` skill (below) auto-applies the same methodology whenever code is written or changed, without needing an explicit invocation.

### `/sonu:design-tree` — design tree mapper

Maps any design problem as an explicit branching tree instead of one linear narrative. This is the `design-tree` skill (below) invoked directly. Pair it with `/sonu:ship` for complete PR lifecycle coverage: tree the design first, then ship the implementation.

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

The same skill auto-applies in plan mode without an explicit invocation — see its entry under Skills below.

### `/sonu:self-review` — where should a reviewer look?

The `self-review` skill invoked directly: the 3–5 riskiest spots in the current diff (untracked files and multi-commit branches included), in plain language, ending with an explicit "this is a pointer, not an approval." Substantial diffs get the full treatment — independent parallel review lenses synthesized adversarially (see the skill entry below); small diffs get a single inline pass. `/sonu:build` and `/sonu:ship` already run it automatically at the right moments — this invocation is for everywhere else.

### `/sonu:memory` — maintain the learned-rules store

The `memory` skill invoked directly: `/sonu:memory` compacts the cross-repo learned-rules store (dedup, decay, evict-over-cap, and graduation candidates); `/sonu:memory show` just lists the active rules by scope. See the skill entry below for what the store is and why it can't bloat.

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

Auto-applied — once the plugin is installed, Claude follows the red-green-refactor discipline whenever it writes, changes, or tests code in any repo, even when "TDD" or "tests" aren't mentioned. `/sonu:tdd` (above) is this same skill invoked directly by name; there is no separate command component.

It encodes a strict test-first methodology with honest carve-outs (spikes are thrown away and rebuilt test-first; code never lands without tests) across thirteen areas:

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
- **Behavioral evidence** — a green unit suite doesn't prove a screen renders or a flow completes; visible and interactive changes get the real flow exercised and evidence captured, or an explicit statement of what went unverified.
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

Auto-applied — once the plugin is installed, Claude runs it at two moments without being asked: before handing back from `/sonu:build` (so you know where to look before you run `/sonu:ship`), and in `/sonu:ship`'s pre-PR fix loop (each pass reviews, the loop fixes, and the final pass's list is embedded in the PR body for traceability and surfaced in the final report). `/sonu:self-review` (above) is this same skill invoked directly by name.

How it reviews scales to the diff. A small diff (under ~100 changed code lines) gets one inline pass. A substantial one gets the fan-out: **six independent review lenses run in parallel as read-only subagents** — correctness, security surfaces, data integrity and migration, blast radius and consumer impact, test adequacy, silent behavior change — each reading the diff cold, with no access to the conversation that produced the code. A seventh **interface** lens joins them only when the diff actually touches user-facing interface files, applying `interface-review`'s domains to catch the regressions a reviewer can't see by reading diff text. The author reviewing its own work is the least reliable judge; fresh eyes that never saw the intent don't inherit the blind spots. The session then synthesizes adversarially: **findings are rejected by default** unless they cite a concrete `file:line` with an articulable failure mechanism, duplicates from independent lenses merge (and rank higher for being co-flagged), and pure style nits die. Lenses run on a cheaper model tier per `model-tiering`; every accept/reject decision stays on the session. On a harness without subagents, it degrades to the single inline pass — same output shape, nothing breaks.

It also compounds: scope-matched rules from the `memory` store (below) ride into every review as extra checks, and confirmed findings that generalize beyond the repo flow back as candidate rules.

What it produces: a plain-language list of the **3–5 spots in the diff** that a reviewer should look hardest at. One line per item with `file:line` where helpful.

What it explicitly is **not**: a score, a grade, a gate, or an approval. Self-scoring rubber-stamps the model's own work; the value is directing *your* eyes to the corners that will otherwise get skimmed. The list ends with a plain statement to that effect.

The same reasoning applies when you ask for a self-review manually — "what should I look at?", "what's risky here?", "self-review this." If the diff is genuinely low-risk, it says so rather than inventing items to fill the list.

### `memory` — lessons that graduate or decay, never pile up

Skills are the plugin's long-term memory — durable, reviewed, versioned. But a lesson learned mid-review ("this class of bug keeps happening") used to evaporate with the session. The `memory` skill owns the staging area between the two: one cross-repo file, `~/.sonu/memory/learned-rules.md`, where confirmed, generalizable rules accumulate evidence until they either **graduate** into a real skill through a normal PR or **decay** out.

The design is built so the store cannot bloat into thousands of entries nobody reads:

- **Bounded on write** — every write dedups first (a near-duplicate bumps the existing rule's `hits` count instead of adding an entry), and hard caps hold at **50 active rules total, 10 per scope**; over-cap entries are evicted lowest-value-first to an auditable archive.
- **Bounded on read** — skills load only the **top 5 scope-matched** rules per review, never the whole file.
- **Graduate or decay** — a rule confirmed 3+ times becomes a candidate to be promoted into the skill it belongs to (via a normal PR to this repo); a rule unused for 90 days gets archived. `/sonu:memory` runs that maintenance pass.

If the store doesn't exist, everything skips it silently — nothing depends on it, it just makes reviews sharper over time when it's there.

### `design-tree` — decide by branching, not by marching

Auto-applied — it fires automatically whenever you're designing or planning an implementation approach, especially in plan mode. `/sonu:design-tree` (above) is this same skill invoked directly by name at any time; there is no separate command component.

It encodes a design methodology built around one core idea: design is traversing a branching tree, not marching a line. What that looks like in practice:

- **Interview first.** Before branching anything, ask 2–4 targeted questions to confirm intent, constraints, success criteria, and non-goals. Designing the right problem saves more context and tokens than anything else.
- **Load the standards as constraints.** `code-standards` always, plus `safe-migrations`/`seo-standards`/`content-seo`/`infra-standards`/`observability`/`blast-radius` and the interface bars (`accessibility`, `layout`, `ui-polish`, and `typography`/`colors`/`ux-writing`) when the surface matches — a fork a standard already settles is stated and cited, never treed. The plan-vs-standards conflict gets resolved at design time, not discovered by the executor mid-build.
- **Find the real forks.** Only decision points where the design could genuinely go in ≥2 consequential ways — not trivia, not forced choices.
- **Enumerate genuine alternatives** at every fork. No strawmen invented to be knocked down.
- **Record the chosen branch** with a decisive reason (a real constraint, trade-off, or irreversibility) — not a vague preference.
- **Keep the rejected branches** with the reason each lost. Stops silent relitigation; preserves real forks to return to.
- **Backtrack deliberately** to a recorded fork when a downstream decision invalidates an earlier choice, rather than patching forward.

The tree is written as a compact nested-bullet notation (`✓ chosen — reason`, `✗ rejected — why`) that's scannable in seconds. In plan mode it becomes a `## Design Tree` section in the plan file, with a `Constraints:` line naming the standards the design was drawn under — and the plan itself must be **executor-ready**: exact file paths, conventions settled in place, a verification check per step, no judgment call left for a smaller model in a fresh session to guess at.

Edit `sonu/skills/design-tree/SKILL.md` to make it yours — it's plain Markdown.

### `model-tiering` — route the work, keep the judgment

A skill, not a command — it fires when a plan is being written or executed. On a strong session model (one with a trustworthy tier below it on the capability ladder), it turns planning model-aware: each plan step that clears the executor-ready bar gets graded `[delegate]` (mechanical, transcription-grade → the cheapest trustworthy tier) or `[delegate-heavy]` (substantive-but-contained → the strongest trustworthy tier between the light grade's tier and the session, or a same-tier subagent when that gap is empty — a Fable session sends heavy steps to Opus, an Opus session uses Opus subagents laterally). At execution, tagged steps run as subagents whose output the session verifies itself; untagged steps and all review stay inline.

The routing is deliberately conservative — the motivation is quality via focus, not cost. Anything ambiguous, architectural, integrative, security-sensitive, or downstream of an open `[?]` fork stays on the strong model, and doubt always resolves upward. On a session model with no trustworthy tier below it, the skill notes that in one line and no-ops — it never escalates upward or routes across model families. Tags are advisory by design: a harness without subagents (Cursor today) executes the same plan inline, unchanged.

Edit `sonu/skills/model-tiering/SKILL.md` to make it yours — it's plain Markdown.

### `ticket-lifecycle` — the tracker is the control plane

The rulebook the queue workflows share, and the reason a tracker swap isn't a rewrite. It fires whenever tickets are being read or written in a queue-driven flow, and it owns four things in one place so five backends and four workflows can't drift apart:

- **Tracker resolution** — `.sonu/factory-config.md`, then `~/.sonu/factory-config.md`, then stop. Never a guessed tracker, because a wrong guess writes ticket state into the wrong system.
- **The operations contract** — list queue, list open, search, fetch, claim, update body, comment, heartbeat, classify, mark status, create, close the loop. Workflows name operations; adapters supply mechanics. An adapter missing an operation is a hard stop that names the gap rather than an improvised command — except the two display/liveness aids (*mark status*, *heartbeat*), which degrade gracefully for older custom adapters.
- **The taxonomy** — exactly one type (`bug`/`enhancement`/`documentation`) and one evidence-based priority (`P0`–`P3`, unset meaning "recommended for rejection"). Size never sets priority; volume never sets priority.
- **Authorization and trust** — only humans apply triggers, the workflow removes one as its claim before any work (that's the concurrency guard), status is derived from the ticket's own artifacts rather than stored as a field anyone has to maintain (the `factory:*` labels and the local tracker's `status:` field are a display cache of those artifacts — written at defined seams, corrected by the sweep, and never read by a workflow to decide anything), and all ticket content is untrusted data that can never redirect a workflow.

Five adapters ship as reference files (`github`, `jira`, `linear`, `local`, `custom`), and only the resolved one gets read.

### `ticket-triage` — make the ticket good enough to build from

Directly invocable as `/sonu:ticket-triage 123`, and what `/sonu:factory` runs for a spec pass. It claims the ticket, then reads the ticket completely, checks open *and* closed tickets for duplicates or an existing fix, inspects the actual code (naming real files, not plausible guesses), and reproduces reported bugs when practical. Out of that it writes the spec: problem and outcome, bounded scope with explicit non-goals, acceptance criteria a test could assert, constraints and affected areas, a verification plan, and the open decisions. Then it routes — ready for approval, blocked on the smallest unblocking question, rejected with evidence, or reproduction failed.

What it never does: write code, open a PR, or apply the implement trigger. The gate's value is that a human reads a spec written by something with no stake in building it.

### `classify-tickets` — a clean backlog is a queryable one

Directly invocable as `/sonu:classify-tickets`, and `/sonu:factory classify`. A sweep over the open backlog that changes **two fields** and nothing else: one type, one evidence-based priority per ticket. It validates every field and value against the tracker before the first write (a failed validation means zero changes, not a half-applied sweep), never invents labels or fields, leaves already-correct tickets untouched, and reports the evidence behind every P0 and P1. Titles, bodies, comments, closures, triggers, and code are all explicitly out of bounds — the narrowness is what makes it safe to run over a hundred tickets at once.

### `bug-finder` — one proven defect beats twenty maybes

Directly invocable as `/sonu:bug-finder [area]`, and `/sonu:factory bugs`. It hunts where defects actually concentrate — recently changed code, error and fallback paths, untrusted input boundaries, process and persistence seams, concurrency and cleanup, weak coverage — tracing real execution paths rather than reading for style.

Then it holds a hard evidence bar: observable incorrect behavior, the triggering path and conditions, why the behavior is *wrong* (citing a contract, test, doc, or call-site expectation), a reproduction, the expected behavior, and a verification approach. Speculation, style opinions, and missing features are not reportable. It dedups against open *and* closed work, files at most **one** ticket per pass with type `bug` and no trigger (queueing stays your call), and when nothing clears the bar it files nothing and says where it looked. It never fixes what it finds — discovery and repair are separate decisions with separate gates. Complement to `debugging`: that one diagnoses a symptom you already have, this one goes looking.

### `interface-review` — the whole screen, not six separate audits

Directly invocable as `/sonu:interface-review [quick|full] [scope]`. A cross-discipline review of a screen, flow, or feature that coordinates the six interface skills below, then consolidates everything into **one** ranked findings table with a single severity scale and one verdict (`Block` / `Needs changes` / `Approve`). It owns orchestration only — every rule belongs to the domain skill that owns it, and a finding is assigned to exactly one owner rather than reported six times.

The discipline is in what it refuses to do: `quick` mode caps at 5 findings and drops `LOW`, `full` at 15; every finding cites `path/to/file:line` with the current implementation; a visual claim inferred only from source gets marked **Not verified** rather than promoted to a finding; a domain whose owning skill is unavailable is reported `Not reviewed` by name instead of quietly counted as covered; and a **Considered but Rejected** table makes restraint visible, so you can see what it looked at and deliberately let stand. Reviews are read-only unless you also ask for the fixes.

### The six interface domains — the bars that fire while you build

Skills, not commands. Like `code-standards`, there's nothing to invoke: they fire automatically on interface work, and `/sonu:build` activates them as design constraints in Phase 1 and build bars in Phase 2 whenever the change touches components, screens, styles, templates, or interface copy. Each one is a spine of rules with reference files carrying the depth, and each defers to its siblings rather than duplicating them.

- **`accessibility`** — the floor, not a compliance pass. Native elements before ARIA, `:focus-visible` rings, a keyboard path for every pointer interaction, focus trap-and-restore, hit-area minimums, labeled and typed controls, errors that actually announce, live regions, alt text by purpose, and surviving 200% zoom and 320px reflow.
- **`layout`** — structure communicates before a word is read. Group with space rather than lines (2× the intra-group gap), controls that look interactive, shared alignment edges, logical properties over physical left/right, visible cues for hidden content, breakpoints that come from the content instead of device presets, and layouts that survive translation and RTL mirroring.
- **`ui-polish`** — the compounding details. Concentric border radius, optical over geometric alignment, shadows for elevation and borders for structure, interruptible transitions, restrained enter/exit motion, exact icon-animation and press-feedback values, never `transition: all`, and icon stroke weight matched to text weight.
- **`typography`** — mostly restraint. `.woff2`, high-level CSS properties over raw OpenType tags, a small semantic type scale, line-height by role, a capped measure (60–75 characters), deliberate wrapping and truncation, tabular numbers on changing values, and 16px inputs so iOS doesn't zoom.
- **`colors`** — OKLCH as a design control, with the project's existing tokens respected rather than converted on sight. Perceptually uniform palettes without hue drift, contrast measured on the rendered pair (APCA and WCAG) and *reported* rather than silently changed, gamut awareness with sRGB fallbacks, and one meaning per color.
- **`ux-writing`** — copy that disappears into the interface. One voice with tone that flexes to the stakes, verb-first buttons that repeat the consequence ("Delete project", never "OK"), one flow vocabulary, links that describe their destination, toggles labeled for the ON state, errors that say how to fix the problem next to where it broke, and empty states that point forward.

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
    └── skills/              # auto-applied; tdd, design-tree, self-review, memory, interface-review, and the ticket skills also invoke directly as /sonu:<name>
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
        ├── memory/
        │   └── SKILL.md     # cross-repo learned-rules store — capped, deduped; rules graduate into skills or decay
        ├── pr-conventions/
        │   └── SKILL.md     # per-type PR templates, living description, reply wording
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
