---
name: debugging
description: The scientific debugging loop — reproduce first, pull the real production event from the observability stack (Sentry, Datadog, Azure Application Insights, CloudWatch, or whatever the repo uses), read the actual error, one hypothesis → one change → one observation, revert failed attempts, escalate instead of thrash. INVOKE PROACTIVELY whenever diagnosing a bug, an error message, a stack trace, a failing or flaky test, a crash, a regression, or a production incident — even when the user just pastes an error with no instructions. Not for writing new features ([[tdd]] + [[code-standards]]) — this governs finding out WHY something is broken; the fix then follows the normal rules.
---

# Debugging — hypothesis testing, not patch roulette

Debugging is a science experiment, not a repair job. You form a theory of what's wrong, make the *one* change that theory predicts will alter the outcome, and observe. Most debugging time is wasted by skipping that discipline: patching where the error *appeared* instead of where it *originated*, changing three things at once, or trying fixes in a loop and keeping the wreckage of the ones that failed. The rules below are the difference between an hour and a day — and between a fix and a fix-shaped bug.

## How to apply this

Run the loop in order: reproduce (for a production report, pull the real event first — section 2 feeds section 1) → read → locate → hypothesize → test the hypothesis → fix → prove. Don't skip ahead to "fix" — a fix you can't connect to a confirmed cause is a guess with good posture.

---

## 1. Reproduce it first — no reproduction, no fix

You cannot verify a fix for something you cannot make happen. Before theorizing, make the failure occur on demand: the exact command, input, and state that triggers it. Then shrink it — smallest input, fewest steps — because every element you remove is a suspect eliminated.

If you *can't* reproduce it, that's not a dead end; **reproducing it is now the task.** Gather what varies (environment, data, timing, version) and close in. A "fix" shipped against an unreproduced bug is a coin flip you can't even watch land.

## 2. Production bug? Pull the real event — never debug a paraphrase

A bug report that arrives as words — "checkout is broken for some users" — is a lossy copy of an error that exists somewhere in full fidelity. If the project has an observability stack, the actual event carries what the reporter can't tell you: the exact exception and stack with the release it happened on, the request/breadcrumb context (that's section 1's reproduction input, handed to you), how often it fires, when it first appeared, and who it hits. Get the event before theorizing.

**Discover what the project uses** — read the repo, not your assumptions:

| Signals in the repo | Platform |
|---|---|
| `@sentry/*` / `sentry-sdk` deps, `SENTRY_DSN` env, `sentry.properties` | Sentry |
| `dd-trace` / `datadog-*` deps, `DD_API_KEY` / `DD_SITE` env | Datadog |
| `applicationinsights` dep, `APPLICATIONINSIGHTS_CONNECTION_STRING` | Azure Application Insights |
| `newrelic` / `rollbar` / `bugsnag` / `honeybadger` deps | those respective services |
| AWS deploy configs with no APM dep | CloudWatch Logs |
| None of the above | plain server/container logs are the source — ask where they land |

**Pull the event.** Prefer a connected MCP server for the platform if the session has one (check the available tools); otherwise use the API/CLI with credentials already in the environment. Command shapes (starting points — verify flags against the platform's current docs, see Provenance):

```bash
# Sentry: recent unresolved issues, then the latest event for one of them
curl -sH "Authorization: Bearer $SENTRY_AUTH_TOKEN" \
  "https://sentry.io/api/0/projects/<org>/<project>/issues/?query=is:unresolved&sort=freq"
curl -sH "Authorization: Bearer $SENTRY_AUTH_TOKEN" \
  "https://sentry.io/api/0/organizations/<org>/issues/<issue-id>/events/latest/"

# Datadog: search recent error logs
curl -s -X POST "https://api.${DD_SITE:-datadoghq.com}/api/v2/logs/events/search" \
  -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  -H "Content-Type: application/json" \
  -d '{"filter":{"query":"status:error","from":"now-1h"},"page":{"limit":20}}'

# Azure Application Insights: recent exceptions via KQL
az monitor app-insights query --app <app-id> \
  --analytics-query 'exceptions | order by timestamp desc | take 20'
```

**No access? Ask — never improvise.** No credentials, no MCP server, no dashboard reachable → ask the user to grant access or paste the full event JSON, and say why: debugging from a paraphrase when the real event exists is choosing to work blind. Do not fabricate an error shape from the description and proceed as if you'd read it.

**What to extract from the event:**

- Exact exception + stack + **release/commit tag** → where to look — and section 6's bisect gets its endpoints for free (first-seen release vs the one before it).
- **Request payload / breadcrumbs / user-agent** → the reproduction input for section 1.
- **Frequency and first-seen** → new regression (what deployed then?) versus long-standing bug (why is it surfacing now?).
- **Affected cohort** (all users, or one browser/region/tenant?) → environmental hypothesis versus logic hypothesis.

**Handle it like production data, because it is.** Telemetry events routinely carry user emails, tokens, and payloads. Use them to debug; never paste them into code comments, tests, commit messages, or PRs ([[code-standards]] sections 8 and 10 govern what may leave the session).

## 3. Read the error — the actual error

Read the message verbatim, the full stack trace, and the *first* error in the log (later errors are usually knock-on noise). Don't pattern-match three words and jump to a familiar diagnosis — "oh, that's probably the cache again" is how you spend an afternoon fixing the wrong thing. The error names a file, a line, a value, an expectation violated. Extract every fact it offers before adding any theory of your own. If the message is genuinely ambiguous, improve it first — better instrumentation is progress (section 6).

## 4. Locate the origin, not the surface

Where the error *explodes* is rarely where things went *wrong* — a null blows up three calls after the function that returned it. Trace backward from the explosion to the first place the state became wrong; that first place is the bug's home, and the only place a fix belongs. Patching at the surface (a null check where it crashed) silences this crash and leaves the wrong state free to surface somewhere else. Ask "where did this bad value come from?" repeatedly until the answer is "here — this is where it was made wrong."

## 5. One hypothesis, one change, one observation

State the hypothesis so it predicts something: *"If the parser drops the last chunk when input isn't newline-terminated, then adding a trailing newline to this failing input will make it pass."* Then make exactly the one change the prediction requires, run, and observe.

- **Prediction confirmed** → hypothesis survives; proceed to the fix.
- **Prediction wrong** → the hypothesis is dead. Don't rescue it with epicycles — form a new one from the new evidence.
- **Never change two things at once.** If you alter the code *and* the config *and* re-run, a changed outcome tells you nothing about which one mattered — you've spent a run and learned zero bits.

## 6. Instrument instead of guessing

When you can't see what's happening, *make* it visible rather than theorizing harder: targeted log lines at the suspect boundary (print the value you *think* is fine — that's the assumption worth testing), a debugger breakpoint, or a bisection. Bisect ruthlessly wherever the search space is linear: `git bisect` across commits, halving the input, disabling half the pipeline. Bisection turns "somewhere in these 200 commits" into eight runs.

Instrumentation added for the hunt is scaffolding, not product — remove it when done ([[code-standards]] bans debris in the diff).

## 7. Revert dead ends completely

When a hypothesis dies, put the code back exactly as it was before you tested it — *then* try the next idea. Layering attempt B on top of half-reverted attempt A creates a chimera nobody can reason about: new bugs from the combination, and a diff that lies about what the fix was. `git stash`/`git checkout -p` are your friends. The final diff should read as: the minimal fix, and nothing else — as if you'd known the answer from the start.

## 8. Prove the fix — and pin it

A fix is proven when: the reproduction from section 1 now passes, **and** you can say *why* in one sentence that connects cause to symptom ("the parser dropped the final chunk because X; feeding it Y exposed it"). If you can't say why it works, it probably doesn't — you've suppressed the symptom, not the cause.

Then pin it with a regression test written *before* you consider the work done — the failing-test-first mechanics live in [[tdd]]'s bug-fix reflex; don't restate them here, follow them there. And per [[tdd]]'s "the test is innocent" rule: if your investigation started from a failing test, the test is not the thing to fix.

## 9. Know when to stop

Three consecutive dead hypotheses means the problem is misframed — stop generating fixes and go back: re-read the error (section 3), re-shrink the reproduction (section 1), question an assumption you've been treating as fact ("the config IS being loaded… have I actually verified that?"). If a timebox expires or the bug needs access/context you don't have, **escalate with a structured summary**: what's observed, the exact reproduction, what was tried, what each attempt ruled out, and the current best hypothesis. That summary is a deliverable, not an admission of failure — it saves the next person the day you just spent, and writing it frequently reveals the answer by itself.

---

## Self-check before you call it done

- Can you make the failure happen on demand — and did the fix make that exact reproduction pass?
- If this was a production report: did you pull the actual event from the observability stack (or explicitly ask for access / the pasted event) rather than debugging the reporter's paraphrase — and did no PII from it leak into code, tests, commits, or PRs?
- Did you read the actual error text and trace to the *origin* of the bad state, or did you patch where it exploded?
- Was every experiment one hypothesis → one change → one observation — never two variables at once?
- Are all dead-end attempts fully reverted, and all hunt-time instrumentation removed?
- Can you state in one sentence why the fix works, connecting cause to symptom?
- Is there a regression test that failed before the fix and passes after ([[tdd]])?
- If you're stopping without a fix: does your escalation summary carry the reproduction, everything ruled out, and your best current hypothesis?

---

## Provenance and maintenance

The methodology (sections 1, 3–9) is durable. Section 2's observability specifics are not — last verified **2026-07**:

- **Platform detection signals** (the deps/env-vars table) — new platforms appear and SDK package names change; extend the table when you meet one it doesn't cover.
- **API/CLI command shapes** (Sentry issues/events endpoints, Datadog `v2/logs/events/search`, `az monitor app-insights query`) — re-verify flags against each platform's current API docs before leaning on an exact invocation. The *principle* — discover the stack from the repo, pull the real event, ask for access instead of improvising — survives any command drift.
