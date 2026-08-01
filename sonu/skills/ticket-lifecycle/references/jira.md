# Adapter — Jira

Jira has native type and priority fields, so those dimensions map onto real fields rather than labels; the triggers stay labels because they are this flow's own vocabulary. Closing the loop is **not** native — a merged GitHub PR does not transition a Jira issue unless the org has wired an integration — so the factory sweep transitions it explicitly.

## Access — MCP first, REST second, stop third

1. **Atlassian MCP tools available** — use them. They carry the authenticated session, handle pagination, and avoid credential handling entirely. Prefer them for every operation below.
2. **No MCP** — REST with these environment variables, all three required: `JIRA_SITE` (e.g. `yourco.atlassian.net`), `JIRA_EMAIL`, `JIRA_API_TOKEN`.
3. **Neither available** — **stop** and tell the user exactly what is missing (the MCP server, or which of the three variables). Never improvise auth, never prompt for a token to paste into a shell command, and never write credentials into the config file or any ticket.

```bash
[ -n "$JIRA_SITE" ] && [ -n "$JIRA_EMAIL" ] && [ -n "$JIRA_API_TOKEN" ] \
  && echo "jira REST credentials present" \
  || echo "STOP: set JIRA_SITE, JIRA_EMAIL, JIRA_API_TOKEN, or enable the Atlassian MCP server"
```

Every REST fence below assumes those three variables are exported and uses `--fail --silent --show-error` so an auth or permission failure is loud rather than a quietly empty result. `JIRA_PROJECT` comes from the config's `jira_project` key, and every fence that needs it **declares it at the top** — a fresh shell means an inherited value is never there, and an empty project key turns a JQL query into `project =  AND …`, which fails as a syntax error at best and returns the wrong project's tickets at worst.

## Mapping

| Concept | Stored as |
|---|---|
| Trigger | Label `factory-ready-for-spec` / `factory-ready-to-implement` / `factory-ready-to-ship` |
| Liveness flag | Label `factory:agent-lost` — machine attestation of a dead pass; removed as the takeover claim, never human-applied |
| Type | Issue type — `bug` maps to Bug, `enhancement` to Story, `documentation` to Task plus a `documentation` label |
| Priority | Priority field — P0/P1/P2/P3 map to Highest/High/Medium/Low |
| Discussion | Native comments |
| Status marker | Label `factory:spec-ready` / `factory:building` / `factory:in-review` / `factory:blocked` — machine-written display cache, never applied by humans and never read to decide |
| Close the loop | Issue key in branch name and PR title, then a Done transition during the sweep |

Two mapping caveats, both real in practice. A project without a Story type (common on service-desk projects) takes Task for `enhancement` — check the project's issue types once and record the choice in the config's prose section. And a project with a customized priority scheme may lack Highest or Lowest; map to the nearest existing value and say so in the pass report rather than failing the whole pass over a field name.

## The operations

**list queue** — JQL, scoped to the configured project. The endpoint is `/rest/api/3/search/jql`, and POST avoids URL-length limits and JQL encoding entirely:

```bash
JIRA_PROJECT=ABC   # substitute, or export from the config's jira_project
TRIGGER=factory-ready-to-implement
# Quote the label value: JQL parses a bare hyphen as an operator, so an
# unquoted factory-ready-to-implement is a syntax error, not a filter.
JQL="project = $JIRA_PROJECT AND statusCategory != Done AND labels = \"$TRIGGER\" ORDER BY priority DESC, updated DESC"
curl --fail --silent --show-error --request POST \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg jql "$JQL" '{jql:$jql,fields:["summary","labels","priority","issuetype","status"],maxResults:50}')" \
  "https://$JIRA_SITE/rest/api/3/search/jql"
```

Three things this endpoint requires that the old one did not, each a silent-wrong-answer if missed: the **`fields` array is mandatory** (it returns almost nothing by default), paging is **cursor-based via `nextPageToken`** rather than `startAt`, and there is no `total` count. If the queue is larger than `maxResults`, follow `nextPageToken` — do not assume one page is the whole queue.

**list open** — every open ticket in the project, trigger or not (drop the `labels` clause from the queue query):

```bash
JIRA_PROJECT=ABC   # substitute
JQL="project = $JIRA_PROJECT AND statusCategory != Done ORDER BY priority DESC, updated DESC"
curl --fail --silent --show-error --request POST \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg jql "$JQL" '{jql:$jql,fields:["summary","labels","priority","issuetype","status"],maxResults:100}')" \
  "https://$JIRA_SITE/rest/api/3/search/jql"
```

**search** — open *and* closed, for duplicate hunting (no `statusCategory` filter):

```bash
JIRA_PROJECT=ABC          # substitute
TOPIC='login redirect'    # substitute
JQL="project = $JIRA_PROJECT AND text ~ \"$TOPIC\" ORDER BY updated DESC"
curl --fail --silent --show-error --request POST \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg jql "$JQL" '{jql:$jql,fields:["summary","status","resolution"],maxResults:30}')" \
  "https://$JIRA_SITE/rest/api/3/search/jql"
```

**fetch** — the issue with its comments and any linked development work:

```bash
KEY=ABC-123   # substitute the real key
curl --fail --silent --show-error \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY?fields=summary,description,labels,priority,issuetype,status,comment"
```

**claim** — confirm the trigger is **present**, remove it, then confirm it is **gone**. An unconfirmed claim is a failed claim, and a trigger that was already absent is a **lost race, not a successful claim** — removing an absent label succeeds, so without the present-check a second session would read a clean result and start duplicate work:

```bash
KEY=ABC-123   # substitute
TRIGGER=factory-ready-to-implement
BEFORE=$(curl --fail --silent --show-error \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY?fields=labels")
# Exact label membership via jq — a raw grep matches substrings, so a
# neighbouring label like factory-ready-to-implement-backup would read
# as the trigger and turn a successful claim into a false failure.
printf '%s' "$BEFORE" | jq -e --arg t "$TRIGGER" '.fields.labels | index($t)' >/dev/null 2>&1 \
  || { echo "STOP: $TRIGGER not present — nothing to claim"; exit 1; }
curl --fail --silent --show-error --request PUT \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "{\"update\":{\"labels\":[{\"remove\":\"$TRIGGER\"}]}}" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY" \
  || { echo "STOP: label update failed"; exit 1; }
LABELS=$(curl --fail --silent --show-error \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY?fields=labels")
if printf '%s' "$LABELS" | jq -e --arg t "$TRIGGER" '.fields.labels | index($t)' >/dev/null 2>&1; then
  echo "STOP: trigger still present — claim failed, do not proceed"
else
  echo "CLAIMED $KEY"
fi
```

**update body** — the description is an ADF document, same as a comment, so build it with `jq`. One paragraph node per block keeps the spec readable; do not paste Markdown into a single text node, since ADF renders it literally:

```bash
KEY=ABC-123   # substitute
SPEC='Problem: expired sessions bounce between login and dashboard.'
curl --fail --silent --show-error --request PUT \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg s "$SPEC" \
    '{fields:{description:{type:"doc",version:1,content:[{type:"paragraph",content:[{type:"text",text:$s}]}]}}}')" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY"
```

This **replaces** the description, so fetch it first and carry the reporter's original text into the new document — a spec that silently deletes what the reporter wrote loses the only first-hand account of the problem.

**comment** — the v3 API takes Atlassian Document Format, not raw Markdown. Keep comment text to plain paragraphs; a Markdown table pasted into an ADF `text` node renders as literal pipes. Build the payload with `jq` so the text is JSON-encoded — a spec comment full of quotes, backticks, and newlines pasted straight into a `--data` string breaks the JSON or truncates the comment:

```bash
KEY=ABC-123   # substitute
TEXT="Triage pass complete. Specification added to the description; awaiting human approval."
curl --fail --silent --show-error --request POST \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg t "$TEXT" '{body:{type:"doc",version:1,content:[{type:"paragraph",content:[{type:"text",text:$t}]}]}}')" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY/comment"
```

**heartbeat** — one comment per **ticket**, edited in place via the comment's own endpoint. Adopt the existing `factory heartbeat` comment when one exists (list comments and match the prefix); create it with the *comment* operation only when absent, recording the id from the response; update with a PUT — same ADF shape, same jq encoding:

```bash
KEY=ABC-123          # substitute
COMMENT_ID=10042     # substitute — from the create response, or list comments and match the "factory heartbeat" prefix
TEXT="factory heartbeat — last seen 2031-01-15T14:30:00Z — stage: built"
curl --fail --silent --show-error --request PUT \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg t "$TEXT" '{body:{type:"doc",version:1,content:[{type:"paragraph",content:[{type:"text",text:$t}]}]}}')" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY/comment/$COMMENT_ID"
```

Never post a second heartbeat — the single edited comment is the unambiguous "last seen" every liveness detector reads. Jira has **no comment-pinning API** — findability is the deterministic locator: search the comment stream for the leading `factory heartbeat —` prefix (same match the adopt step already uses). Report once that pinning is unsupported; never abort the pass for it.

**classify** — type and priority are single-valued fields, so setting them removes the conflicting value inherently. Any `documentation` label for the docs type is a separate label edit:

```bash
KEY=ABC-123   # substitute
curl --fail --silent --show-error --request PUT \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"fields":{"issuetype":{"name":"Bug"},"priority":{"name":"High"}}}' \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY"
```

**Unset priority** — for a ticket recommended for rejection, clear the field rather than picking a low value:

```bash
KEY=ABC-123   # substitute
curl --fail --silent --show-error --request PUT \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"fields":{"priority":null}}' \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY"
```

If the project's field configuration makes priority required (so `null` is rejected), say so in the pass report and leave the existing value rather than inventing a rank — the report is where the "recommend rejection" signal then lives.

**mark status** — one status marker at a time, and deliberately two calls: first remove every `factory:*` status label (removing an absent label is a no-op), then add the target. A single update carrying the target in both the remove and add lists bets on Jira's processing order, and losing that bet strips the marker you meant to set. Native workflow transitions are deliberately **not** used for this — status schemes vary per project, and the adapter cannot assume one:

```bash
KEY=ABC-123   # substitute
curl --fail --silent --show-error --request PUT \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"update":{"labels":[{"remove":"factory:spec-ready"},{"remove":"factory:building"},{"remove":"factory:in-review"},{"remove":"factory:blocked"}]}}' \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY"
```

```bash
KEY=ABC-123               # substitute
STATUS=factory:building   # one of: factory:spec-ready factory:building factory:in-review factory:blocked
curl --fail --silent --show-error --request PUT \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "{\"update\":{\"labels\":[{\"add\":\"$STATUS\"}]}}" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY"
```

Clearing the marker is the first call alone. These labels are the display cache from the spine's section 6 — passes write them, the sweep corrects them, and no workflow reads them to decide anything.

**create**:

```bash
JIRA_PROJECT=ABC   # substitute, or export from the config's jira_project
SUMMARY="Session cookie survives logout"
curl --fail --silent --show-error --request POST \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg p "$JIRA_PROJECT" --arg s "$SUMMARY" \
    '{fields:{project:{key:$p},summary:$s,issuetype:{name:"Bug"}}}')" \
  "https://$JIRA_SITE/rest/api/3/issue"
```

**close the loop** — transition IDs are per-project, so resolve the Done transition by name instead of hardcoding a number:

```bash
KEY=ABC-123   # substitute
curl --fail --silent --show-error \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY/transitions"
# then POST the id whose name matches the project's done state
TRANSITION_ID=31   # substitute the id resolved above
curl --fail --silent --show-error --request POST \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "{\"transition\":{\"id\":\"$TRANSITION_ID\"}}" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY/transitions"
```

Put the issue key in the branch name (`ticket/ABC-123-slug`) and the PR title so a human can trace the PR to the ticket even when no integration exists.

**read blockers** — Jira stores edges in `fields.issuelinks`. Prefer MCP when available; REST fallback below. A link whose `type.inward` is `is blocked by` and whose `outwardIssue` is present means *this* ticket is blocked by that issue (the blocker sits in `outwardIssue` when *this* issue is the dependent). Match the default English link-type name; the name is **instance-configurable**, and a renamed type degrades to today's behavior (no edges visible) rather than failing the pass:

```bash
KEY=ABC-123   # substitute — the dependent
curl --fail --silent --show-error \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/$KEY?fields=issuelinks" \
  | jq '[.fields.issuelinks[]
      | select(.type.inward == "is blocked by" and .outwardIssue != null)
      | {key: .outwardIssue.key,
         category: .outwardIssue.fields.status.statusCategory.key,
         summary: .outwardIssue.fields.summary}]'
```

Direction, load-bearing: when *this* issue is the dependent (inward side of a `Blocks` link), the blocker appears in **`outwardIssue`**, not `inwardIssue`. Swapping those fields reads the wrong side of every edge. A blocker is still open when its `statusCategory.key` is not `done` — use the category, never the localized `status.name`. List-wide form: JQL `labels = "factory-ready-to-implement" AND issueLinkType = "is blocked by"` for the dependency-blocked authorized set, and the same without the link clause for the full trigger queue — subtract to get the frontier. If the link-type name has been renamed on the instance, skip the JQL form and fall back to per-issue *read blockers* over the trigger queue.

**link blocker** — record that A is blocked by B (`Blocks` link: outward = blocker, inward = dependent), then read back. Prefer MCP; REST:

```bash
A=ABC-123   # substitute — the dependent (cannot start yet)
B=ABC-45    # substitute — the blocker that must close first
curl --fail --silent --show-error --request POST \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg a "$A" --arg b "$B" \
    '{type:{name:"Blocks"},inwardIssue:{key:$a},outwardIssue:{key:$b}}')" \
  "https://$JIRA_SITE/rest/api/3/issueLink" \
  || { echo "STOP: link blocker failed for $A blocked-by $B"; exit 1; }
# Read-back — part of the operation. On A (the dependent), the blocker is outwardIssue.
curl --fail --silent --show-error \
  --user "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/$A?fields=issuelinks" \
  | jq --arg b "$B" '[.fields.issuelinks[]
      | select(.type.inward == "is blocked by" and .outwardIssue.key == $b)] | length' \
  | grep -qx 1 \
  || { echo "STOP: read-back failed — $B not in $A's blockers (direction or link-type name may differ)"; exit 1; }
echo "LINKED $A blocked-by $B"
```

Worked example: linking `ABC-200` blocked-by `ABC-100` means create with `outwardIssue` = `ABC-100` (blocker) and `inwardIssue` = `ABC-200` (dependent). Reading blockers of `ABC-200` then finds `ABC-100` in **`outwardIssue`**. Swapping create or read makes `ABC-200` appear to block `ABC-100` — which is why the read-back is mandatory.

## Bootstrap

Jira labels are created implicitly on first use, so there is nothing to pre-create. Verify once instead: the configured project exists, its issue types cover the three type values (noting the Story-or-Task decision), and its priority scheme covers the four values. Also confirm the label field accepts the `factory:*` status names by setting and clearing one on a scratch issue — Jira rejects spaces in labels, and if a workspace's configuration rejects the colon too, fall back to `factory-status-*` names and record the substitution in the config's prose section. Report any gap — do not invent fields, types, or statuses.

## Provenance and maintenance

Last verified 2026-07 (dependency and pinning notes added 2026-08 — **doc-sourced**, no live Jira instance available this pass):

- REST v3 comment bodies require Atlassian Document Format; the v2 API accepted plain text. Re-verify against the current Jira Cloud REST reference before switching to plain strings.
- The queue query is `POST /rest/api/3/search/jql`. The older `/rest/api/3/search` has been **removed** from Jira Cloud and returns `410 Gone` pointing at this replacement — if a snippet anywhere still targets it, that snippet is already broken. The replacement requires an explicit `fields` array and pages by `nextPageToken` (no `startAt`, no `total`). Re-verify against Atlassian's issue-search API group.
- Transition IDs differ per project workflow — always resolve by name at runtime.
- Basic auth with an API token is the documented Cloud mechanism; Jira Data Center or Server deployments differ and are out of scope for this adapter.
- Jira Cloud labels reject spaces; colons in label names are accepted on standard configurations, but this is workspace-configurable — the Bootstrap's scratch-issue check is the re-verification, and `factory-status-*` is the recorded fallback naming.
- **Issue links (doc):** `POST /rest/api/3/issueLink` with `type.name: "Blocks"`, `outwardIssue` = blocker, `inwardIssue` = dependent. Read via `fields.issuelinks` filtered on `type.inward == "is blocked by"`. The link-type display name is instance-configurable — a rename yields empty blocker lists (safe degrade), never a hard fail. Re-verify against a live project's issue-link types before the first production link.
- **Comment pin:** unsupported on Jira Cloud — locator is the `factory heartbeat —` prefix.
