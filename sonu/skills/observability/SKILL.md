---
name: observability
description: Producing telemetry worth having — instrumentation (metrics, traces, error capture), health-check endpoints, and alert quality for any service or endpoint. INVOKE PROACTIVELY whenever creating a new service, API endpoint, background job, or scheduled task; adding metrics, tracing, error tracking (Sentry/Datadog/App Insights SDKs), health/liveness/readiness endpoints, dashboards, or alert rules; or when the user says "add monitoring/logging/alerting." Log-line mechanics live in [[code-standards]] section 8; the CONSUMING side — pulling events while diagnosing — is [[debugging]].
---

# Observability — instrument for the question you'll ask at 2am

The [[debugging]] skill's first production move is "pull the real event." That move only works if someone instrumented the service *before* it broke — and that someone is you, now, while writing it. You can't debug what you didn't record, and you can't retrofit telemetry onto an incident that already happened. The bar: every service answers four questions without anyone SSHing anywhere — **is it up, is it erroring, is it slow, and for whom?**

## How to apply this

Instrumentation is part of the feature, not a follow-up: a new endpoint ships with its metrics, error capture, and trace context wired, the same way it ships with tests. Run the self-check before calling any service change done.

---

## 1. The four questions drive what you instrument

For every operation a service performs (each endpoint, each job, each consumer), emit enough to answer:

- **Traffic** — how often is this happening? (a counter)
- **Errors** — how often is it failing, split by kind? (a counter with a failure label)
- **Latency** — how long does it take, as a *distribution*? (a histogram — averages hide the slow tail where users actually live)
- **Saturation** — for resources (pools, queues, memory): how close to full? (a gauge)

If an operation emits these four, most incidents are diagnosable from a dashboard. If it emits none, every incident starts with "let's add logging and wait for it to happen again" — the most expensive sentence in operations.

## 2. Logs: at boundaries and decisions, not every line

The mechanics — levels, structured key/value context, correlation ids, the no-secrets rule — have one home: [[code-standards]] section 8. What belongs *around* them: log at **boundaries** (request in/out, external call made, job started/finished) and **decisions** (retry scheduled, fallback taken, permission denied), not as a narration of every line executed. A log stream that records everything is a log stream nobody can read during an incident.

## 3. Metrics: counters, histograms, and the cardinality trap

- **Counters** for things that happen (`orders_created_total`), **histograms** for durations and sizes (`checkout_duration_seconds`), **gauges** for current levels (`queue_depth`). Name with the unit in the name — a metric called `timeout` that's secretly milliseconds causes a 1000× misread.
- **The cardinality trap — the one metrics mistake that takes the platform down.** Every distinct label combination is a separate stored time series. Labels are for *low-cardinality dimensions*: endpoint, status class, region — a bounded set you could list. Putting `user_id`, `order_id`, an email, or a raw URL path (with ids embedded) in a label multiplies series without bound until the metrics backend falls over or the bill does. Per-entity data belongs in logs and traces, which are built for it — never in metric labels.

```js
// Avoid: unbounded label — one time series per user, forever
metrics.increment('checkout_completed', { user_id: user.id });

// Prefer: bounded dimensions; the user id lives in the log line / trace
metrics.increment('checkout_completed_total', { plan: user.plan, region });
logger.info('Checkout completed', { user_id: user.id, request_id: req.id });
```

## 4. Traces: propagate the context or lose the story

One user action fans out across services, and without propagation each service logs its fragment into a void. The rules: accept the incoming trace context and pass it on every outbound call (HTTP headers, queue message metadata); open a span per meaningful unit — especially every external call (DB, HTTP, queue); and put the correlation/request id from [[code-standards]] section 8 on both the logs *and* the trace, so you can pivot between them. Most platform SDKs (App Insights, Datadog, OpenTelemetry) auto-instrument the common frameworks — wire that up first and add manual spans only where the auto view has gaps.

## 5. Health checks: two endpoints, two different questions

- **Liveness** — "is the process alive?" Checks *nothing external* — no database ping, no downstream call. It exists so the orchestrator can restart a genuinely wedged process.
- **Readiness** — "can I serve traffic right now?" Checks the dependencies the service can't work without, and gates load-balancer membership.

Keeping them separate is what prevents the restart storm: when the database blips, readiness pulls instances out of rotation (recoverable in seconds) instead of liveness restarting the whole fleet — whatever the orchestrator (App Service, ECS, a load balancer's health probe), the semantics are the same. Return machine-readable detail from readiness (`{ "db": "ok", "cache": "degraded" }`) — during an incident, *which* dependency failed is the entire question. And the health endpoints themselves are unauthenticated infrastructure: they must not leak internals beyond dependency names ([[code-standards]] section 10).

## 6. Error capture: with context, without noise

Wire the error tracker (Sentry, App Insights, Datadog — whatever the project uses) so unhandled failures are captured with the context [[debugging]] section 2 will need later:

- **Tag every event with the release/commit.** First-seen-release is the single highest-value debugging fact — it turns "somewhere in history" into a two-commit bisect.
- Attach the correlation id and non-sensitive request context. The no-PII discipline from [[code-standards]] sections 8/10 applies to telemetry payloads too.
- **Don't capture expected outcomes as errors.** Validation rejections, 404s, user cancellations — these are traffic, not failures. An error feed where 95% of events are noise is an error feed nobody reads, which is how the real 5% ships to the front page.

## 7. Alerts: page on symptoms, and only when a human must act

- **Alert on what users feel** — error rate, latency percentile, availability — not on causes like CPU or memory. High CPU with healthy latency is a Tuesday; paging on it teaches people to ignore pages. Cause-level signals belong on dashboards, where they explain a symptom alert.
- **Every alert is actionable and owned**: a threshold someone can defend, a responder who knows it's theirs, and a linked runbook or at minimum a "check X, then Y" note. An alert with no action attached is spam with a pager.
- **Alert fatigue is a system failure, not a personnel issue.** An alert that fires routinely and gets dismissed routinely must be re-thresholded, converted to a dashboard, or deleted — a channel full of ignored red is *worse* than silence, because it hides the real one. (Same failure shape as coverage theater in [[tdd]]: signal that doesn't mean anything trains people to skip signal entirely.)

## 8. Instrumentation ships with the feature

The definition of done for a new endpoint, job, or service includes: the four signals (section 1), error capture with release tagging (section 6), trace propagation (section 4), and — for a new service — the two health endpoints (section 5). Retrofitting is 10× the cost and always happens *after* the incident that needed it. If the project has no observability stack at all, say so explicitly and put the gap in the hand-off notes ([[self-review]]) rather than silently shipping a black box.

---

## Self-check before you call a service change done

- Can the four questions (traffic, errors, latency, saturation) be answered for every new operation from emitted telemetry alone?
- Any metric label that could grow without bound (user ids, entity ids, raw paths)? Move it to logs/traces.
- Are durations histograms (not averages), with units in the metric name?
- Is trace context propagated on every outbound call, and does the correlation id appear on both logs and traces?
- Liveness checks only the process; readiness checks dependencies and returns which one is unhealthy — and neither leaks internals?
- Are captured errors tagged with release/commit, carrying non-PII context only — and are expected outcomes (validation, 404s) excluded from the error feed?
- Does every new alert page on a user-facing symptom, with an owner and an action — and did you delete or demote anything that fires-and-gets-ignored?
- If the project has no observability stack: did you say so in the hand-off instead of shipping a black box silently?

## Provenance and maintenance

The principles (four signals, cardinality, liveness/readiness split, symptom-based alerting) are durable. The SDK specifics — auto-instrumentation coverage in App Insights/Datadog/OpenTelemetry, release-tagging config per platform — drift with vendor releases; last sanity-checked 2026-07. Verify SDK wiring against the platform's current docs; the platform *detection* table lives in [[debugging]] section 2 (one home).
