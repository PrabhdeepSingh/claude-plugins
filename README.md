# claude-plugins

Prabhdeep (Sonu) Singh's personal [Claude Code](https://claude.com/claude-code) commands, packaged as a marketplace so they work on any repo and any machine I sign into — and so I can share them.

## Install

```
/plugin marketplace add PrabhdeepSingh/claude-plugins
/plugin install sonu@prabhdeep-tools
```

Run those once per device. After that, `/sonu:ship` is available in every repo on that machine, and the **code-standards** skill rides along automatically — no command to run, it just shapes how code gets written. To pull updates later:

```
/plugin marketplace update prabhdeep-tools
```

> Commands from a plugin are namespaced as `/<plugin>:<command>` — that's why it's `/sonu:ship`, not `/ship`. The slash menu autocompletes, so typing `ship` finds it.

## Commands

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

## Skills

### `code-standards` — code the way I do

A skill, not a command — there's nothing to invoke. Once the plugin is installed, Claude consults it automatically before writing, generating, or refactoring code in any repo, so AI-written code lands in my style instead of generic boilerplate.

It encodes the foundation across eleven areas:

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
│   └── marketplace.json     # marketplace manifest (name: prabhdeep-tools)
├── LICENSE                  # MIT
└── sonu/                    # the "sonu" plugin (your personal namespace)
    ├── .claude-plugin/
    │   └── plugin.json
    ├── commands/
    │   └── ship.md          # the /sonu:ship command
    └── skills/
        └── code-standards/
            └── SKILL.md     # the code-standards skill (auto-applied)
```

## License

[MIT](LICENSE) © Prabhdeep Singh
