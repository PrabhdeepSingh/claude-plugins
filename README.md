# claude-tools

Personal [Claude Code](https://claude.com/claude-code) plugin marketplace — my own commands, available on any machine I sign into.

## Plugins

- **ship** — PR babysitter. Branch, commit, open PR, gather Claude (code + security) + Copilot reviews, fix/justify/resolve every finding, loop until clean, then merge — autonomously. Repo-agnostic (auto-detects the repo and default branch).

## Install (run once per device)

```bash
/plugin marketplace add PrabhdeepSingh/claude-tools
/plugin install ship@claude-tools
```

Then `/ship` is available in every repo on that device.

## Updating

When this repo changes, pull the latest on each device:

```bash
/plugin marketplace update claude-tools
```

## Layout

```
claude-tools/
├── .claude-plugin/
│   └── marketplace.json     # marketplace manifest (lists plugins)
└── ship/                    # the "ship" plugin
    ├── .claude-plugin/
    │   └── plugin.json      # plugin manifest
    └── commands/
        └── ship.md          # the /ship command
```
