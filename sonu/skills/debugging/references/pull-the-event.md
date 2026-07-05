# Pulling the real event — platform detection and command shapes

Depth for the debugging skill's section 2. Read this when a production bug report has arrived and you are about to locate and pull the actual event.

## Discover what the project uses — read the repo, not your assumptions

| Signals in the repo | Platform |
|---|---|
| `@sentry/*` / `sentry-sdk` deps, `SENTRY_DSN` env, `sentry.properties` | Sentry |
| `dd-trace` / `datadog-*` deps, `DD_API_KEY` / `DD_SITE` env | Datadog |
| `applicationinsights` dep, `APPLICATIONINSIGHTS_CONNECTION_STRING` | Azure Application Insights |
| `newrelic` / `rollbar` / `bugsnag` / `honeybadger` deps | those respective services |
| AWS deploy configs with no APM dep | CloudWatch Logs |
| None of the above | plain server/container logs are the source — ask where they land |

New platforms appear and SDK package names change; extend the table when you meet one it doesn't cover.

## Command shapes

Starting points, not gospel — verify flags against the platform's current API docs before leaning on an exact invocation (verification date: the skill's Provenance section, one home):

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
