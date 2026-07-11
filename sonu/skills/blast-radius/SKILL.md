---
name: blast-radius
description: Consumer-impact discipline for contract changes — before changing the shape, format, semantics, or published identity of anything other code consumes, mechanically enumerate every consumer, classify which ones degrade silently, and verify one downstream path end-to-end. INVOKE PROACTIVELY whenever a change alters a function's return value or type, an API/tool response body, a serialized payload (JSON/XML/protobuf), a DB column read elsewhere, a log or telemetry field, an event/queue message, a config value, an env var, or CLI/stdout output that anything parses — or whenever it renames or removes an identifier a caller addresses by name, such as an MCP or plugin tool name, an API route or URL, a CLI command or flag, an env var key, or an event type, especially when the callers are external clients you don't control and may have cached the old name. Even when the change "just wraps," "just renames," or "just adds an envelope around" existing data or identity, it counts. Skip it for purely internal changes with no consumers outside the edited function and for strictly additive optional fields that no consumer is required to read. (Prevention pairs with [[code-standards]]'s loud-failure rule at data boundaries; verifying the downstream path is [[tdd]] territory; DB schema seams follow [[safe-migrations]].)
---

# Blast Radius — who reads the thing you're changing?

A change can be locally correct, fully tested, and still break production — because the correctness of a *producer* says nothing about the assumptions of its *consumers*. And the most expensive version of that break isn't the loud one; it's the silent one, where a downstream `catch` or fallback converts the breakage into missing data that nobody notices for weeks, long after the deploy that caused it has stopped being a suspect. This skill exists to make "who reads this?" a mandatory step in the change, not an instinct you hope the author has.

## When to apply this

Apply it the moment a change alters the **shape, format, semantics, or published identity of anything consumed outside the edited code**: a function's return value or type, an API or tool response body, a serialized payload, a database column other code reads, a log or telemetry field, an event or queue message, a config value or env var, CLI/stdout output that anything parses, or an identifier a caller addresses by name (a tool, route, or command). Wrapping counts. Renaming counts — a field or an identifier. Adding an envelope around existing data counts — the bytes a consumer parses are different even though "the data is still there."

Skip it for changes with genuinely no consumers outside the edited function, and for strictly additive optional fields that no consumer is required to read.

The test is not "did I change an interface file" — it's **"does anything outside this diff read the bytes, fields, or values I'm changing, or address something by the name I'm changing?"** If you haven't checked, the answer is unknown, and unknown means yes.

Run sections 1–6 in order. They're cheap — a few searches and one real observation — and each exists because skipping it has a specific failure mode.

---

## 1. Name the seam

Before editing, state in one sentence what contract is changing: the producer, the data shape, and the transport (return value / response body / column / log field / event / stdout). Why: an unnamed seam can't be searched for. "I'm changing how search results come back" is a vibe; "the tool's text channel currently carries raw JSON that callers parse" is a searchable surface with findable consumers.

The seam includes the **address a consumer uses to reach the producer** — a tool name, an API route, a CLI command — not only the payload it parses. Renaming a tool renames the address clients call, not just the data they read; an invocation by the old name now resolves to nothing instead of returning altered data.

## 2. Enumerate consumers mechanically — never from memory

Search for the seam; don't recall it. Grep for the symbol, the field names, the parse sites (`JSON.parse`, `json.loads`, `Unmarshal`, `deserialize`), the column name, the topic or queue name — in this repo, and in known sibling repos or clients when the seam crosses a service boundary. Memory produces the consumers you wrote; search produces the consumers that exist.

**The consumer you forget is never a call site — it's a pipeline.** Loggers, telemetry helpers, analytics/ETL jobs, dashboards, alert rules, and export scripts read production data shapes without ever appearing in the producer's call graph. Explicitly search the logging and observability layer for the seam's field names before declaring the consumer list complete. Why: call-graph intuition finds callers; it structurally cannot find out-of-band readers — and those are exactly the ones whose breakage nobody is watching for.

## 3. Classify each consumer by how it fails

For every consumer found, put it in one of three buckets:

- **Unaffected** — it reads fields or bytes the change doesn't touch. Say so explicitly per consumer; don't hand-wave the list.
- **Breaks loudly** — it throws, 500s, or fails CI. This is the good kind: it will be caught before or immediately after ship.
- **Degrades silently** — the killer class. Hunt specifically for `try/catch → default`, `|| []`, `?? null`, optional chaining, and "return empty on error" downstream of the seam. Why this bucket gets its own hunt: silent degradation is invisible to tests, error dashboards, and users in the moment — it surfaces weeks later as corrupted analytics or quietly missing data, when the causing deploy is no longer under suspicion.

Two more questions cut across all three buckets and change what "affected" means for a published identity. **Can you reach it?** — an owned, in-repo consumer you can update in the same change, versus an external or uncontrolled client whose runtime you don't touch when you ship. **Does it cache the identity or shape?** — a client that cached a tool name, route, or response shape keeps calling what it remembers; the moment the old identity stops resolving, it errors loudly for the client, but that error usually lands in the client's own logs, not yours — unless you've built explicit telemetry for it, your side sees nothing to alert on. A consumer that is both unreachable and caching is the case that turns a rename into an outage — it's why disposition 1 below is off the table for it.

## 4. Decide per consumer, before shipping

Every affected consumer gets exactly one of these three dispositions, chosen deliberately — not every disposition is available for every consumer; a consumer's own traits (reachability, caching) can rule one out, as disposition 1's caveat below shows:

1. **Update it in the same change** — the default, but only for consumers you can reach and deploy in lockstep with the change. An external client you don't control can't be "updated in the same change"; if you can't ship its update alongside yours, this disposition doesn't apply to it.
2. **Version the contract** — publish the new shape alongside the old (a new field next to the old one), migrate consumers, then retire the old shape deliberately. This is [[safe-migrations]]'s expand → migrate → contract, and it generalizes from schemas to every data seam — including identities. When the seam is a published identity with external or caching consumers (a tool name, an API route, a CLI command), this is the *only* safe disposition: keep the old identity **resolving** — an alias or shim that forwards the old name to the new one — through a deprecation window, rather than hard-renaming it. Alias first; announce the deprecation; remove the old identity only after the window has passed and telemetry shows zero calls to it. Never remove the old identity in the same release that adds the new one, and never ship the removal right before you go offline for the weekend or a holiday — the removal is the risky step, and it needs someone watching when it lands.
3. **Accept the break and write it down** — legitimate only when it's explicit in the change record, never by omission, and never the default for a consumer you couldn't actually reach.

Why the ceremony: the only wrong option is the implicit one — a consumer that was never dispositioned is a decision made by accident.

**Avoid:** rename the MCP tool `search` to `searchDocuments`; the server's own tests are green; ship on a Friday afternoon. Every client that had cached the old tool list keeps calling `search`, gets "unknown tool," and errors out — with nobody watching until support tickets arrive Monday. The same pattern breaks a hardcoded endpoint (`/v1/search` renamed to `/v1/documents/search`) or a CLI command (`mytool search` renamed to `mytool find`): a cached client, a bookmarked URL, or a pinned script keeps addressing a name that no longer resolves.

**Prefer:** add `searchDocuments` alongside `search`; keep `search` as an alias that delegates to it; mark `search` deprecated in its own description so new clients see the signal; announce the deprecation window; remove `search` only in a later release, after telemetry confirms nothing is still calling it. Old cached clients keep working through the transition; new clients adopt the new name on their own schedule; the removal is a deliberate, watched step instead of an accident that lands over a weekend. The same pattern covers routes (for a web page, `301` the old path to the new one — [[seo-standards]]'s redirect rule; for an API route, a `301` risks clients silently downgrading a non-`GET` request to `GET`, so serve both paths, or use a method-preserving `308`) and CLI commands (keep the old subcommand as a deprecated alias).

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
- If the change renames or removes a published identifier with external or cached consumers, is the old identity still resolving (alias/shim) through a deprecation window — i.e. not removed in this release?
