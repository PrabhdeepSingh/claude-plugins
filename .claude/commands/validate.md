---
description: Mechanically validate the sonu plugin repo before a PR — manifest sync, YAML frontmatter, shell-fence syntax, named-source and AI-attribution scans, cross-reference integrity. Only meaningful inside the claude-plugins repo; in any other repo, say so and stop.
argument-hint: ""
allowed-tools: Bash, Read, Grep, Glob
---

# /validate — the CI this repo doesn't have

Run every check below in order and report a **PASS/FAIL table** at the end. A FAIL doesn't block you mid-run — collect them all, then fix or justify each. The checks are copy-paste snippets; run them from the repo root.

**Guard first.** This command validates the `claude-plugins` repo itself. If the current repo isn't it, stop:
```bash
[ -f .claude-plugin/marketplace.json ] && [ -d sonu/skills ] \
  && echo "OK: claude-plugins repo" || echo "STOP: not the claude-plugins repo — /validate has nothing to check here"
```

## 1. Manifests parse and stay in sync

All four JSON manifests must parse; the two `plugin.json` files must be identical; versions must match everywhere they appear:
```bash
for f in .claude-plugin/marketplace.json .cursor-plugin/marketplace.json \
         sonu/.claude-plugin/plugin.json sonu/.cursor-plugin/plugin.json; do
  python3 -m json.tool "$f" >/dev/null && echo "PARSE OK: $f" || echo "FAIL: $f does not parse"
done
diff <(python3 -m json.tool sonu/.claude-plugin/plugin.json) \
     <(python3 -m json.tool sonu/.cursor-plugin/plugin.json) \
  >/dev/null && echo "SYNC OK: plugin.json pair identical" || echo "FAIL: the two plugin.json files differ"
```

## 2. Every description mentions every component (and counts them honestly)

The plugin description must name every command and skill (this is the drift that has actually happened — twice). It must also not *lie about how many* there are: the description pitches "N skills", and a hardcoded number goes stale the moment a skill is added, so the count is checked against reality too.

**All four description homes are checked, not just the marketplaces.** The two `plugin.json` files carry the same user-facing text; checking only the marketplaces would let the pair drift and still pass:
```bash
python3 - <<'EOF'
import json, os, re, sys
commands = sorted(f[:-3] for f in os.listdir('sonu/commands') if f.endswith('.md'))
skills   = sorted(d for d in os.listdir('sonu/skills') if os.path.isdir(f'sonu/skills/{d}'))
homes = {
    '.claude-plugin/marketplace.json':  lambda d: d['plugins'][0]['description'],
    '.cursor-plugin/marketplace.json':  lambda d: d['plugins'][0]['description'],
    'sonu/.claude-plugin/plugin.json':  lambda d: d['description'],
    'sonu/.cursor-plugin/plugin.json':  lambda d: d['description'],
}
fails = 0
texts = {}
for mf, pick in homes.items():
    desc = pick(json.load(open(mf)))
    texts[mf] = desc
    missing = [c for c in commands if c not in desc] + [s for s in skills if s not in desc]
    if missing:
        fails += 1
        print(f"FAIL: {mf} description omits: {', '.join(missing)}")
    else:
        print(f"OK: {mf} mentions all {len(commands)} commands + {len(skills)} skills")
    claimed = re.search(r'(\d+) skills', desc)
    if not claimed:
        print(f"OK: {mf} claims no skill count (nothing to drift)")
    elif int(claimed.group(1)) != len(skills):
        fails += 1
        print(f"FAIL: {mf} says '{claimed.group(1)} skills' but {len(skills)} exist")
    else:
        print(f"OK: {mf} skill count ({len(skills)}) matches reality")
if len(set(texts.values())) != 1:
    fails += 1
    print("FAIL: the four descriptions are not identical — /release Phase 2 syncs them")
else:
    print("OK: all four description homes carry identical text")
sys.exit(1 if fails else 0)
EOF
```

## 3. Skill frontmatter parses and carries name + description

```bash
python3 - <<'EOF'
import glob, sys
try:
    import yaml
except ImportError:
    sys.exit("SKIP: PyYAML not installed — eyeball each frontmatter block instead (fence pairs, name:, description:)")
fails = 0
for path in sorted(glob.glob('sonu/skills/*/SKILL.md') + glob.glob('.claude/skills/*/SKILL.md')) + sorted(glob.glob('sonu/commands/*.md') + glob.glob('.claude/commands/*.md')):
    text = open(path).read()
    if not text.startswith('---\n'):
        print(f"FAIL: {path} has no frontmatter"); fails += 1; continue
    block = text.split('---\n')[1]
    try:
        meta = yaml.safe_load(block)
    except yaml.YAMLError as e:
        print(f"FAIL: {path} frontmatter does not parse: {e}"); fails += 1; continue
    required = ('name', 'description') if 'skills' in path else ('description',)
    missing = [k for k in required if not meta.get(k)]
    if missing:
        print(f"FAIL: {path} missing {missing}"); fails += 1
    else:
        print(f"OK: {path}")
sys.exit(1 if fails else 0)
EOF
```

## 4. Shell fences pass syntax check in bash AND zsh

Extracts every ```bash fence and runs `bash -n` + `zsh -n`. Placeholders like `<PR number>` are angle-bracket substitution markers — the script strips those lines before checking. **`references/*.md` is in scope**: a heavy skill's adapter snippets are the most operational shell in the repo, and leaving them unchecked was exactly the gap that let a zsh-only parse failure sit in a reference file:
```bash
python3 - <<'EOF'
import glob, re, subprocess, sys, tempfile, textwrap
fails = 0
for path in sorted(glob.glob('sonu/commands/*.md') + glob.glob('.claude/commands/*.md')) + sorted(glob.glob('sonu/skills/*/SKILL.md') + glob.glob('.claude/skills/*/SKILL.md') + glob.glob('sonu/skills/*/references/*.md') + glob.glob('.claude/skills/*/references/*.md')):
    for i, block in enumerate(re.findall(r'^ *```bash\n(.*?)^ *```', open(path).read(), re.S | re.M)):
        # Dedent first: fences inside markdown list items are indented, and an indented
        # heredoc terminator is a real zsh parse failure the executor must also avoid.
        block = textwrap.dedent(block)
        cleaned = '\n'.join(l for l in block.splitlines() if '<' not in l or '>' not in l or l.lstrip().startswith('#'))
        with tempfile.NamedTemporaryFile('w', suffix='.sh', delete=False) as f:
            f.write(cleaned); tmp = f.name
        for shell in ('bash', 'zsh'):
            r = subprocess.run([shell, '-n', tmp], capture_output=True, text=True)
            if r.returncode != 0:
                print(f"FAIL: {path} fence #{i+1} ({shell} -n): {r.stderr.strip().splitlines()[0]}")
                fails += 1
print("OK: all bash fences pass bash -n and zsh -n" if not fails else f"{fails} fence failure(s)")
sys.exit(1 if fails else 0)
EOF
```

## 5. No named sources in skills (house rule 1)

Recursive (`-r`), so this already covers any skill's `references/*.md` — no separate reference-file scan needed. Heuristic scan — a hit is not automatically a violation; judge each one. **Expected hits: exactly one** — the rule statement in plugin-dev §2 (house rule 1's own text mentions "studies"). Any *other* hit needs judging against house rule 1. Do not reword plugin-dev's rule to dodge the regex:
```bash
grep -rniE '\b(study|studies|paper|professor|university|according to [A-Z][a-z]+ [A-Z])\b' sonu/skills/ .claude/skills/ \
  && echo "REVIEW: expected = 1 hit (plugin-dev house rule 1); judge anything beyond that" \
  || echo "UNEXPECTED: zero hits — plugin-dev's rule statement should match; was it reworded?"
```

## 6. No AI attribution (house rule 3)

Recursive over `sonu/`, so this already covers any skill's `references/*.md` too. The only legitimate mentions in the tree are the *rules forbidding it* (ship.md's contract, plugin-dev) and this check's own grep lines — the scan excludes itself. Anything else is a violation:
```bash
grep -rn 'Co-Authored-By\|Generated with Claude' sonu/ .claude/skills/ .claude/commands/ README.md \
  | grep -v 'commands/validate.md' \
  | grep -vi 'no ai attribution\|do not add\|no `co-authored-by`\|attribution to commits' \
  && echo "REVIEW: hits above must all be rule statements, not actual attribution" || echo "OK"
# History probe: ONE pre-rule commit (70d1cf3, before the no-attribution rule existed) carries the
# trailer and cannot be rewritten — a hit at that commit is expected. Only NEWER commits are violations:
git log --grep='Co-Authored-By' --oneline | head -5
```

## 7. Registry and cross-reference integrity

Every `BOT_RE=` declaration in ship.md must be identical (the registry is copy-pasted into self-contained fences by design — they must never diverge), and every `sonu:<name>` and double-bracket skill reference must resolve to a real component:
```bash
n=$(grep -h "BOT_RE='" sonu/commands/ship.md | sed 's/^[[:space:]]*//' | sort -u | wc -l | tr -d ' ')
[ "$n" = "1" ] && echo "OK: single consistent BOT_RE" || { echo "FAIL: $n distinct BOT_RE variants in ship.md:"; grep -n "BOT_RE='" sonu/commands/ship.md; }

# Cross-references are scanned OUTSIDE fenced code blocks (a ``[[name]]`` quoted inside an
# example fence is illustration, not a reference — scanning fences produced false trips), and
# fences are stripped BEFORE matching so an example can neither satisfy nor fail the check:
python3 - <<'PYEOF'
import glob, os, re, sys
fails = 0
for path in sorted(glob.glob('sonu/**/*.md', recursive=True) + glob.glob('.claude/skills/**/*.md', recursive=True) + glob.glob('.claude/commands/*.md')):
    text = re.sub(r'^ *```.*?^ *```', '', open(path).read(), flags=re.S | re.M)
    for ref in set(re.findall(r'sonu:([a-z-]+)', text)):
        if not (os.path.isdir(f'sonu/skills/{ref}') or os.path.isfile(f'sonu/commands/{ref}.md')):
            print(f'FAIL: {path}: sonu:{ref} resolves to no skill or command'); fails += 1
    for ref in set(re.findall(r'\[\[([a-z-]+)\]\]', text)):
        if not os.path.isdir(f'sonu/skills/{ref}'):
            print(f'FAIL: {path}: [[{ref}]] resolves to no skill'); fails += 1
print('OK: cross-references resolve (fences stripped)' if not fails else f'{fails} cross-reference failure(s)')
sys.exit(1 if fails else 0)
PYEOF
```

## 8. Reference-file pointers resolve, and no orphans

For any skill that has split heavy content into `references/*.md` (plugin-dev §4): every backtick-wrapped `references/…` path mentioned in its `SKILL.md` must resolve to a real file, and every file actually present under `references/` must be pointed at from somewhere in `SKILL.md` (the per-rule pointer or the index table) — an orphan file is dead weight nobody will ever read:
```bash
python3 - <<'EOF'
import glob, os, re, sys
fails = 0
for skill_md in sorted(glob.glob('sonu/skills/*/SKILL.md') + glob.glob('.claude/skills/*/SKILL.md')):
    skill_dir = os.path.dirname(skill_md)
    text = open(skill_md).read()
    mentioned = set(re.findall(r'`(references/[\w.-]+\.md)`', text))
    actual = set(
        os.path.relpath(p, skill_dir)
        for p in glob.glob(f'{skill_dir}/references/*.md')
    )
    for ref in sorted(mentioned - actual):
        print(f"FAIL: {skill_md} points at {ref}, which doesn't exist"); fails += 1
    for ref in sorted(actual - mentioned):
        print(f"FAIL: {skill_dir}/{ref} exists but {os.path.basename(skill_md)} never mentions it (orphan)"); fails += 1
    if actual and not (mentioned - actual) and not (actual - mentioned):
        print(f"OK: {skill_md} — {len(actual)} reference file(s), all pointers resolve, no orphans")
sys.exit(1 if fails else 0)
EOF
```

## 9. README inventory completeness

```bash
for c in sonu/commands/*.md; do
  name=$(basename "$c" .md)
  grep -q "sonu:$name" README.md || echo "FAIL: README never mentions /sonu:$name"
done
for s in sonu/skills/*/; do
  name=$(basename "$s")
  grep -q "$name" README.md || echo "FAIL: README never mentions skill '$name'"
done
echo "(README inventory scan done — silence above means OK)"
```

## Report

Print a table: check number, one-line name, PASS / FAIL / REVIEW / SKIP, and for each FAIL the one-line reason (SKIP = a check couldn't run, e.g. PyYAML missing in check 3 — report it as SKIP with the manual fallback done or not, never as PASS). If everything passes, say exactly that in one line — don't pad. If any FAIL touches a manifest or version field, point the user at `/release` for the sync procedure.

**When NOT to use this:** in any repo other than claude-plugins (the guard catches it), or as a substitute for reading a diff — this checks mechanics, not meaning. Content-level review is still `/sonu:build`'s self-review phase and the PR review in `/sonu:ship`.
