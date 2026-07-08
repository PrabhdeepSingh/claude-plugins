---
name: blast-radius
description: Consumer-impact discipline for contract changes — before changing the shape, format, or semantics of anything other code consumes, mechanically enumerate every consumer, classify which ones degrade silently, and verify one downstream path end-to-end. INVOKE PROACTIVELY whenever a change alters a function's return value or type, an API/tool response body, a serialized payload (JSON/XML/protobuf), a DB column read elsewhere, a log or telemetry field, an event/queue message, a config value, an env var, or CLI/stdout output that anything parses — even when the change "just wraps," "just renames," or "just adds an envelope around" existing data. Skip it for purely internal changes with no consumers outside the edited function and for strictly additive fields nothing is required to read. (Prevention pairs with [[code-standards]]'s loud-failure rule at data boundaries; verifying the downstream path is [[tdd]] territory; DB schema seams follow [[safe-migrations]].)
---

# Blast Radius — who reads the thing you're changing?

A change can be locally correct, fully tested, and still break production — because the correctness of a *producer* says nothing about the assumptions of its *consumers*. And the most expensive version of that break isn't the loud one; it's the silent one, where a downstream `catch` or fallback converts the breakage into missing data that nobody notices for weeks, long after the deploy that caused it has stopped being a suspect. This skill exists to make "who reads this?" a mandatory step in the change, not an instinct you hope the author has.

## When to apply this

Apply it the moment a change alters the **shape, format, or semantics of anything consumed outside the edited code**: a function's return value or type, an API or tool response body, a serialized payload, a database column other code reads, a log or telemetry field, an event or queue message, a config value or env var, or CLI/stdout output that anything parses. Wrapping counts. Renaming counts. Adding an envelope around existing data counts — the bytes a consumer parses are different even though "the data is still there."

Skip it for changes with genuinely no consumers outside the edited function, and for strictly additive optional fields nothing is required to read.

The test is not "did I change an interface file" — it's **"does anything outside this diff read the bytes, fields, or values I'm changing?"** If you haven't checked, the answer is unknown, and unknown means yes.

Run sections 1–6 in order. They're cheap — a few searches and one real observation — and each exists because skipping it has a specific failure mode.

---

## 1. Name the seam

Before editing, state in one sentence what contract is changing: the producer, the data shape, and the transport (return value / response body / column / log field / event / stdout). Why: an unnamed seam can't be searched for. "I'm changing how search results come back" is a vibe; "the tool's text channel currently carries raw JSON that callers parse" is a searchable surface with findable consumers.

## 2. Enumerate consumers mechanically — never from memory

Search for the seam; don't recall it. Grep for the symbol, the field names, the parse sites (`JSON.parse`, `json.loads`, `Unmarshal`, `deserialize`), the column name, the topic or queue name — in this repo, and in known sibling repos or clients when the seam crosses a service boundary. Memory produces the consumers you wrote; search produces the consumers that exist.

**The consumer you forget is never a call site — it's a pipeline.** Loggers, telemetry helpers, analytics/ETL jobs, dashboards, alert rules, and export scripts read production data shapes without ever appearing in the producer's call graph. Explicitly search the logging and observability layer for the seam's field names before declaring the consumer list complete. Why: call-graph intuition finds callers; it structurally cannot find out-of-band readers — and those are exactly the ones whose breakage nobody is watching for.

## 3. Classify each consumer by how it fails

For every consumer found, put it in one of three buckets:

- **Unaffected** — it reads fields or bytes the change doesn't touch. Say so explicitly per consumer; don't hand-wave the list.
- **Breaks loudly** — it throws, 500s, or fails CI. This is the good kind: it will be caught before or immediately after ship.
- **Degrades silently** — the killer class. Hunt specifically for `try/catch → default`, `|| []`, `?? null`, optional chaining, and "return empty on error" downstream of the seam. Why this bucket gets its own hunt: silent degradation is invisible to tests, error dashboards, and users in the moment — it surfaces weeks later as corrupted analytics or quietly missing data, when the causing deploy is no longer under suspicion.

## 4. Decide per consumer, before shipping

Every affected consumer gets one of exactly three dispositions, chosen deliberately:

1. **Update it in the same change** — the default for consumers you own.
2. **Version the contract** — publish the new shape alongside the old (a new field next to the old one), migrate consumers, then retire the old shape deliberately. This is [[safe-migrations]]'s expand → migrate → contract, and it generalizes from schemas to every data seam.
3. **Accept the break and write it down** — legitimate only when it's explicit in the change record, never by omission.

Why the ceremony: the only wrong option is the implicit one — a consumer that was never dispositioned is a decision made by accident.

## 5. Verify one downstream path end-to-end

Run the *consumer*, not just the producer's tests: exercise the real flow, then read the actual logged row / emitted event / dashboard value and confirm it is populated and correct. Why: the producer's suite proves the producer; only observed downstream output proves the seam.

**Avoid:** wrap a tool's text output in a security envelope; the tool's own tests are green; ship it. A telemetry helper elsewhere `JSON.parse`s that text, throws, and returns `null` from its catch — and every event logs null result counts for a week with zero errors flagged, because the helper's fallback made the breakage look like data.

**Prefer:** name the seam ("the tool's text output — is it parsed downstream?"), grep for parse sites, find the telemetry helper, update it to read the structured field instead, then run one real query and confirm the logged row shows a non-null count. Same change, one extra hour, no silent week-long outage.

## 6. Record the impact

End the change with a one-paragraph **Downstream impact** statement: the seam, the consumers found, their classification, and what was verified end-to-end. It feeds [[self-review]]'s risk list and the PR description. Why: the analysis is only durable if the reviewer can see it was done — an undocumented consumer audit is indistinguishable from a skipped one.

---

## Self-check before you call it done

- Did you name the seam in one sentence before editing?
- Did you enumerate consumers by *searching* (symbol, field names, parse sites, column/topic names) — not from memory?
- Did you explicitly search the logging/telemetry/analytics layer for the seam's field names?
- Is every silent-degradation site downstream of the seam (`catch → default`, `|| []`, `?? null`) identified and dispositioned?
- Does every affected consumer have an explicit disposition — updated, contract versioned, or break accepted in writing?
- Did you verify at least one downstream consumer end-to-end and *observe* its real output (logged row, emitted event, dashboard value)?
- Is the Downstream impact statement written where the reviewer will see it?
