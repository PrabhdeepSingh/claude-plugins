# PR body templates — one per change type

Fill the matching template below. Drop any section that genuinely doesn't apply. Never leave a `<placeholder>` unfilled. See `pr-conventions/SKILL.md` Section B for change-type detection and where to embed the issue link.

### Feature / feat

```
## Summary
- <what this adds and why>

## Motivation
- <the problem or user need this solves>

## Changes
- <key implementation decision or component added>

## Risk / reviewer attention
<RISKS list from self-review — 3–5 items>

## Test plan
- <happy-path verification>
- <edge case to verify>

## Screenshots
<!-- Remove this section if no UI changed -->
```

### Bugfix / fix

```
## Summary
- <what broke and what this fixes>

## Root cause
<one or two sentences on why it broke>

## Fix
- <what changed to address the root cause>

## Risk / reviewer attention
<RISKS list from self-review>

## Regression test / test plan
- <failing test added (or why there isn't one)>
- <manual steps to verify the fix>
```

### Hotfix

```
## Summary (URGENT)
- <what is broken in production and what this fixes>

## Impact
- Affected: <scope — users, feature, service>
- Severity: <P0 / P1 / …>

## Root cause
<brief — expand in the incident doc>

## Fix
- <surgical change made>

## Rollback plan
- <how to revert if this makes things worse>

## Verification
- <minimal steps to confirm before merging>

## Risk / reviewer attention
<RISKS list from self-review>
```

### Chore

```
## Summary
- <what maintenance task this performs>

## Changes
- <package / tool / config updated — version if relevant>

## Risk / reviewer attention
<RISKS list — often low; say so explicitly: "Low risk — no logic changed">

## Test plan
N/A — no logic changed  <!-- or: suite ran, all green -->
```

### Refactor

```
## Summary
- <what was restructured; behavior is preserved>

## Changes
- <module / file restructured and how>

## Behavior preserved
- <how verified: existing suite green / added tests for X>

## Risk / reviewer attention
<RISKS list — call out blast radius if a shared utility changed>

## Test plan
- <existing suite green; new tests if coverage expanded>
```

### Docs

```
## Summary
- <what docs were added, updated, or removed>

## What changed
- <file or section and what is different>

No test plan — docs only.
```

### Perf

```
## Summary
- <what was optimized and why it matters>

## Benchmark
| Metric | Before | After |
|--------|--------|-------|
| <metric> | <value> | <value> |

## Changes
- <algorithm / query / cache strategy changed>

## Risk / reviewer attention
<RISKS list>

## Test plan
- <how correctness is verified after the optimization>
```

### Release

```
## Summary
- Releasing v<version>

## Highlights
- <key feature or change>

## Breaking changes
<!-- Remove if none -->
- <change and migration path>

## Migration notes
<!-- Remove if none -->
<steps for callers>

## Verification / rollback
- <smoke test or deploy check>
- Rollback: <revert tag / feature flag / step>

## Risk / reviewer attention
<RISKS list>
```
