---
name: observability
description: >-
  What a service must emit to be diagnosable: the four signals, bounded metric labels, the two health endpoints, error capture and symptom-based alerts. Use when adding a service, endpoint, background job, instrumentation or alert rule. Log-line rules: [[code-standards]]; reading telemetry while diagnosing: [[debugging]].
---

# Observability — instrument for the question you will ask during an incident

The project's own instructions win: where its AGENTS.md, CLAUDE.md or established conventions say otherwise, follow them and skip that part of this.

[[debugging]] starts a production investigation by pulling the real event, which only works if the service recorded it. The bar is that a service can answer four questions without anyone logging into a machine: is it up, is it erroring, is it slow, and for whom. A project with no telemetry stack, such as a local-only tool, skips this skill; say so in the hand-off ([[self-review]]) when a service ships without one.

## 1. Four signals per operation

For each endpoint, job or consumer, emit traffic (a counter), errors split by kind (a counter with a failure label), latency as a distribution (a histogram, because averages hide the slow tail) and, for pools, queues and memory, saturation (a gauge). Beyond that floor, add a signal only when it answers a question an on-call engineer would ask about this feature.

## 2. Logs

Log at boundaries (request in and out, external call, job start and finish) and at decisions (retry scheduled, fallback taken, permission denied). Levels, structured context, correlation ids and the no-secrets rule are in [[code-standards]] section 8.

## 3. Metrics

Put the unit in the metric name, because a `timeout` that is secretly milliseconds gets misread by a factor of a thousand.

Every distinct label combination is a separate stored time series. Labels are for bounded sets such as endpoint, status class and region. A user id, order id, email, request id, raw error text or raw URL path in a label multiplies series without limit until the backend or the bill fails. Per-entity data belongs in logs and traces.

## 4. Traces

Accept the incoming trace context and pass it on every outbound call, including queue message metadata. Open a span for each external call. Put the correlation id on both logs and traces so one can be reached from the other. Wire the platform SDK's auto-instrumentation first and add manual spans where it has gaps.

## 5. Health checks

Liveness answers "is the process alive" and checks nothing external. Readiness answers "can I serve traffic now", checks the dependencies the service cannot work without, and gates load-balancer membership. Keeping them separate prevents a restart storm: when the database blips, readiness removes instances from rotation instead of liveness restarting the fleet. Readiness returns which dependency failed, in machine-readable form. Both endpoints are unauthenticated, so they expose dependency names and nothing more.

## 6. Error capture

- Tag every event with the release or commit. First-seen release turns a search through history into a short bisect.
- Attach the correlation id and non-sensitive request context only.
- Do not capture expected outcomes (validation rejections, 404s, user cancellations) as errors. A feed that is mostly noise goes unread.

## 7. Alerts

- Alert on what users feel: error rate, latency percentile, availability. Causes such as CPU belong on dashboards.
- Every alert has a defensible threshold, an owner and a note on what to check.
- Use two severities only: page (a human acts now) and ticket (someone acts this week). A third tier becomes the pile everyone ignores.
- An alert that fires and gets dismissed routinely is re-thresholded, moved to a dashboard or deleted. Flag existing ones to the owner; do not delete alert rules outside your diff.

## 8. Ship it with the feature, then verify it

A new endpoint, job or service includes its signals, error capture with release tagging, trace propagation and, for a new service, the two health endpoints. Then check the telemetry itself: force an error in a non-production environment and find it in the tracker by correlation id, confirm the metric series appear with the expected labels, and fire each new alert once by lowering its threshold. The test is that an induced failure can be located from telemetry alone.

## Maintenance

SDK auto-instrumentation coverage and release-tagging configuration change with vendor releases (last checked 2026-07). The platform detection table is in the debugging skill's reference file.
