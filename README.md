# claude-plugins

Prabhdeep Singh's personal [Claude Code](https://claude.com/claude-code) plugins, packaged as a marketplace so they work on any repo and any machine I sign into — and so I can share them.

## Install

```
/plugin marketplace add PrabhdeepSingh/claude-plugins
/plugin install ship@prabhdeep-tools
```

Run those once per device. After that, `/ship` is available in every repo on that machine. To pull updates later:

```
/plugin marketplace update prabhdeep-tools
```

## Plugins

### `ship` — PR babysitter

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
| `/ship` | Auto — light touch on trivial diffs, full panel on big or security-relevant ones. |
| `/ship light` | Minimal Claude review (skips on truly trivial changes); still collects whatever the repo's bots post. |
| `/ship full` | Deep Claude code + security review, full re-review loop. |

Mode words are parsed forgivingly — `quick`/`fast`/`lite` → `light`, and `thorough`/`deep`/`max` (typos included) → `full`. The mode only scales *Claude's own* reviews; the external bots auto-run on the repo regardless, so they cost the same whether `/ship` waits for them or not.

## Requirements

- The [`gh`](https://cli.github.com/) CLI, authenticated (`gh auth status`).
- A GitHub remote on the repo.
- For Claude reviews: the `/code-review` and `/security-review` commands.
- Any AI reviewer bots are picked up automatically if they're enabled on the repo — nothing to configure here.

## A note on trust

`/ship` runs shell commands and **merges PRs on your behalf**. Read the command before you install it (`ship/commands/ship.md`), and only point it at repos where you're comfortable with that. Plugins run with your local permissions.

## Layout

```
claude-plugins/
├── .claude-plugin/
│   └── marketplace.json     # marketplace manifest (name: prabhdeep-tools)
├── LICENSE                  # MIT
└── ship/                    # the "ship" plugin
    ├── .claude-plugin/
    │   └── plugin.json
    └── commands/
        └── ship.md          # the /ship command
```

## License

[MIT](LICENSE) © Prabhdeep Singh
