---
name: blast-radius
description: >-
  Consumer-impact rules for contract changes: find every consumer, flag the ones that degrade silently, verify one downstream path. Use when a change alters something other code reads or addresses by name. Not for purely internal changes or additive optional fields. Schema changes: [[safe-migrations]]; downstream tests: [[tdd]].
---

# Blast radius — who reads the thing you are changing?

The project's own instructions win: where its AGENTS.md, CLAUDE.md or established conventions say otherwise, follow them and skip that part of this.

A producer can be correct and fully tested and still break its consumers, because its tests say nothing about what they assume. The expensive break is the silent one: a downstream `catch` or fallback turns the breakage into missing data that nobody notices for weeks, long after the deploy that caused it has stopped being a suspect.

## When this applies

It applies when a change alters the shape, format, meaning or published name of anything consumed outside the edited code: a return value or type, an API or tool response, a serialized payload, a database column other code reads, a log or telemetry field, an event or queue message, a config value or env var, CLI output that something parses, or an identifier a caller addresses by name (a tool, route or command). Wrapping existing data in an envelope counts, because the bytes a consumer parses are different. Renaming counts.

Skip it for changes with no consumers outside the edited function, and for strictly additive optional fields that no consumer is required to read. If you have not checked whether anything outside the diff reads what you are changing, treat the answer as yes.

## 1. Name the seam

State in one sentence which contract is changing: the producer, and either the data shape and its transport or the published name callers use. A named seam can be searched for. The seam includes the address a consumer uses to reach the producer: renaming a tool renames what clients call, so a call by the old name resolves to nothing.

## 2. Find consumers by searching

Search for the symbol, the field names, the parse sites, the column name and the topic or queue name, in this repo and in known sibling repos or clients when the seam crosses a service boundary. Memory returns the consumers you wrote; search returns the ones that exist.

The consumer you forget is never a call site. It is a pipeline. Loggers, telemetry helpers, analytics and export jobs, dashboards and alert rules read production data shapes without appearing in the producer's call graph. Search the logging and observability layer for the seam's field names before calling the list complete, because nobody is watching for breakage there.

## 3. Classify each consumer by how it fails

- **Unaffected.** It reads fields the change does not touch. Say so per consumer.
- **Breaks loudly.** It throws, returns a 500 or fails CI. This is caught before or just after shipping.
- **Degrades silently.** Look specifically for `try/catch` returning a default, `|| []`, `?? null`, optional chaining and "return empty on error" downstream of the seam. Tests, error dashboards and users do not see this class when it happens.

Two questions cut across the buckets for a published name. Can you reach the consumer, meaning you can update and deploy it with this change? Does it cache the name or shape? A client that cached a tool name keeps calling it, and its error lands in the client's logs, not yours. A consumer that is both unreachable and caching is the case that turns a rename into an outage.

## 4. Decide per consumer before shipping

Every affected consumer gets one of three dispositions.

1. **Update it in the same change.** This is the default, and it is only available for consumers you can deploy in lockstep.
2. **Version the contract.** Publish the new shape or name beside the old one, migrate consumers, then retire the old one deliberately. This is the expand, migrate, contract sequence of [[safe-migrations]] applied to any data seam. For a published name with external or caching consumers it is the only safe option: keep the old name resolving through an alias for a deprecation window, and remove it in a later release once telemetry shows no calls to it. Do not ship the removal just before nobody is watching, because the removal is the risky step.
3. **Accept the break and write it down.** This is legitimate only when it is explicit in the change record. It is not available for a published name with external or caching consumers.

Three rules govern a deprecation. If you own the thing being deprecated, migrating its consumers is your job: ship the migration path, not an announcement with a deadline. Prefer a warning consumers act on at their own pace over a forced cutoff. Do not deprecate without a replacement. Announcement alone fails because, with enough consumers, every observable behaviour becomes a contract, including bugs, timing, ordering and error text.

For routes: redirect a web page's old path with a `301` (see [[seo]]). For an API route a `301` can downgrade a non-`GET` request to `GET`, so serve both paths or use a `308`. For a CLI, keep the old subcommand as a deprecated alias.

## 5. Verify one downstream path

Run a consumer, not only the producer's tests. Exercise the real flow, then read the logged row, emitted event or dashboard value and confirm it is populated and correct. The producer's suite proves the producer; only observed downstream output proves the seam.

An example of the failure: a tool's text output is wrapped in a security envelope and the tool's tests pass. A telemetry helper elsewhere parses that text as JSON, throws, and returns `null` from its catch. Every event then logs null result counts with no error raised.

## 6. Record the impact

End the change with a short downstream-impact statement: the seam, the consumers found, how each was classified, and what was verified. Put it where the reviewer will read it. In a pull-request flow that is the risk section of the PR body ([[pr-conventions]]); [[self-review]] cites it from there. A consumer audit nobody can see looks the same as one that was skipped.
