# Adapter — Linear

Linear has a native priority scale and native PR magic words, so priority maps to a real field and closing the loop is free. Type and the triggers are labels.

## Access — MCP first, GraphQL second, stop third

1. **Linear MCP tools available** — use them; they carry the session and avoid credential handling.
2. **No MCP** — the GraphQL API at `https://api.linear.app/graphql` with `LINEAR_API_KEY` in the environment. The key goes in the `Authorization` header directly (no `Bearer` prefix for a personal API key).
3. **Neither** — **stop** and say which is missing. Never improvise auth and never write the key into config or ticket text.

```bash
[ -n "$LINEAR_API_KEY" ] && echo "linear API key present" \
  || echo "STOP: set LINEAR_API_KEY, or enable the Linear MCP server"
```

Every fence below assumes `LINEAR_API_KEY` is exported and uses `--fail --silent --show-error`. `LINEAR_TEAM` comes from the config's `linear_team` key and is **declared in each fence that needs it** — a fresh shell never inherits it, and an empty team filter returns either a GraphQL error or an empty queue that reads as "no work."

Note that GraphQL returns HTTP 200 with an `errors` array for query-level failures, so **check the response body for `errors`** rather than trusting the exit status alone — `--fail` will not catch a rejected mutation, which means a failed claim can look like a successful one. Test it as `(.errors // []) | length == 0`, not `.errors | not`: an empty `errors: []` array is truthy in jq, so the shorter form reads a perfectly good response as a failure.

## Mapping

| Concept | Stored as |
|---|---|
| Trigger | Label `factory-ready-for-spec` / `factory-ready-to-implement` / `factory-ready-to-ship` |
| Liveness flag | Label `factory:agent-lost` — machine attestation of a dead pass; removed as the takeover claim, never human-applied |
| Type | Label `bug` / `enhancement` / `documentation` |
| Priority | Native priority — P0 is 1 (Urgent), P1 is 2 (High), P2 is 3 (Medium), P3 is 4 (Low); 0 means no priority |
| Discussion | Native comments |
| Status marker | Label `factory:spec-ready` / `factory:building` / `factory:in-review` / `factory:blocked` — machine-written display cache, never applied by humans and never read to decide |
| Close the loop | Native — `Fixes ENG-123` in the PR body, with the GitHub integration enabled |

Priority `0` is Linear's "no priority" and is exactly the right value for a ticket recommended for rejection, matching the taxonomy's unset-means-rejected rule.

## The operations

**list queue** — incomplete issues on the team carrying a trigger label:

```bash
LINEAR_TEAM=ENG   # substitute, or export from the config's linear_team
QUERY='query($team:String!,$label:String!){issues(filter:{team:{key:{eq:$team}},state:{type:{nin:["completed","canceled"]}},labels:{name:{eq:$label}}},first:50){nodes{identifier title priority labels{nodes{name}} updatedAt}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg team "$LINEAR_TEAM" --arg label factory-ready-to-implement \
    '{query:$q,variables:{team:$team,label:$label}}')" \
  https://api.linear.app/graphql
```

**list open** — every incomplete issue on the team, trigger or not (drop the `labels` filter):

```bash
LINEAR_TEAM=ENG   # substitute
QUERY='query($team:String!){issues(filter:{team:{key:{eq:$team}},state:{type:{nin:["completed","canceled"]}}},first:100){nodes{identifier title priority labels{nodes{id name}} updatedAt}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg team "$LINEAR_TEAM" '{query:$q,variables:{team:$team}}')" \
  https://api.linear.app/graphql
```

**search** — open *and* closed, for duplicate hunting (no state filter):

```bash
TOPIC='login redirect'   # substitute
QUERY='query($t:String!){searchIssues(term:$t,first:30){nodes{identifier title state{name type} url}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg t "$TOPIC" '{query:$q,variables:{t:$t}}')" \
  https://api.linear.app/graphql
```

**fetch** — one issue with its comments and attachments (linked PRs arrive as attachments):

```bash
ID=ENG-123   # substitute the real identifier
QUERY='query($id:String!){issue(id:$id){identifier title description priority state{name type} labels{nodes{id name}} comments{nodes{body createdAt user{name}}} attachments{nodes{url title}}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg id "$ID" '{query:$q,variables:{id:$id}}')" \
  https://api.linear.app/graphql
```

The label `id` is selected alongside `name` deliberately — claim and classify both replace the label set wholesale and need those ids, so fetching only names forces a second round trip.

**claim** — three shell steps, same shape as the other adapters: confirm **present**, mutate, confirm **gone**. Do not treat the prose as the check; a claim that lives only in prose gets skipped, and skipping it re-opens the lost-race hole the other adapters close.

Step 1, confirm the trigger is present (an absent trigger is a lost race, not a claim):

```bash
ID=ENG-123   # substitute
TRIGGER=factory-ready-to-implement
QUERY='query($id:String!){issue(id:$id){labels{nodes{id name}}}}'
BEFORE=$(curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg id "$ID" '{query:$q,variables:{id:$id}}')" \
  https://api.linear.app/graphql)
printf '%s' "$BEFORE" | jq -e '(.errors // []) | length == 0' >/dev/null 2>&1 \
  || { echo "STOP: GraphQL returned errors"; exit 1; }
printf '%s' "$BEFORE" | jq -e --arg t "$TRIGGER" \
  '[.data.issue.labels.nodes[].name] | index($t)' >/dev/null 2>&1 \
  || { echo "STOP: $TRIGGER not present — nothing to claim"; exit 1; }
# the label id set to send back, trigger excluded:
printf '%s' "$BEFORE" | jq -c --arg t "$TRIGGER" \
  '[.data.issue.labels.nodes[] | select(.name != $t) | .id]'
```

Step 2, send that id set (the mutation below). Step 3, re-run step 1's query and confirm the trigger is **absent** and the other labels survived; anything else is a failed claim — stop.

`issueUpdate` replaces `labelIds` **wholesale**, so `LABEL_IDS` must be the issue's full remaining label id set. Sending the literal `[]` below without substituting **strips every label on the issue**, including its type — destroying classification that the fetch you just ran is the only record of:

```bash
ID=ENG-123
# REQUIRED substitution — paste the id set step 1 printed (trigger excluded).
# Sending an empty array here deletes ALL labels on the issue.
LABEL_IDS='["11111111-1111-1111-1111-111111111111"]'
QUERY='mutation($id:String!,$ids:[String!]){issueUpdate(id:$id,input:{labelIds:$ids}){success}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg id "$ID" --argjson ids "$LABEL_IDS" \
    '{query:$q,variables:{id:$id,ids:$ids}}')" \
  https://api.linear.app/graphql
```

If the issue's only label was the trigger, the correct substitution is a genuinely empty set — state that explicitly in the pass rather than reaching for the placeholder by accident.

Then re-run **fetch**, confirm the trigger label is absent *and* the other labels survived, and check the response body for an `errors` array. Anything else is a failed claim: stop.

**update body** — Linear descriptions accept Markdown, so the spec goes in as written:

```bash
ID=ENG-123   # substitute
DESC='## Problem

## Acceptance criteria

## Original report

> preserved verbatim from the reporter, as data
'
QUERY='mutation($id:String!,$desc:String!){issueUpdate(id:$id,input:{description:$desc}){success}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg id "$ID" --arg desc "$DESC" \
    '{query:$q,variables:{id:$id,desc:$desc}}')" \
  https://api.linear.app/graphql
```

This **replaces** the description — fetch first and carry the reporter's original text into the rewrite.

**comment** — Linear comments accept Markdown:

```bash
ID=ENG-123   # substitute
BODY='Triage pass complete. Specification added to the description; awaiting human approval.'
QUERY='mutation($id:String!,$body:String!){commentCreate(input:{issueId:$id,body:$body}){success}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg id "$ID" --arg body "$BODY" \
    '{query:$q,variables:{id:$id,body:$body}}')" \
  https://api.linear.app/graphql
```

**heartbeat** — one comment per **ticket**, edited in place with `commentUpdate`. Adopt the existing `factory heartbeat` comment when one exists (the *fetch* operation's `comments` selection carries bodies to match the prefix against — add `id` to its selection); create it with the *comment* operation only when absent (add `comment{id}` to `commentCreate`'s selection to record the id); update thereafter:

```bash
COMMENT_ID=00000000-0000-0000-0000-000000000000   # substitute — from the create response
BODY='factory heartbeat — last seen 2031-01-15T14:30:00Z — stage: built'
QUERY='mutation($id:String!,$body:String!){commentUpdate(id:$id,input:{body:$body}){success}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg id "$COMMENT_ID" --arg body "$BODY" \
    '{query:$q,variables:{id:$id,body:$body}}')" \
  https://api.linear.app/graphql
```

Same `errors`-array check as every mutation, and never a second heartbeat — one edited comment is the unambiguous "last seen". Linear has **no comment-pinning API** — findability is the deterministic locator: match the leading `factory heartbeat —` prefix in the issue's comments (same match the adopt step already uses). Report once that pinning is unsupported; never abort the pass for it.

**classify** — priority is a single field; the type label replaces its siblings, so pass the full intended label set (type label plus any unrelated labels the issue already carries, minus the other two type labels):

```bash
ID=ENG-123        # substitute
PRIORITY=2        # 1 Urgent, 2 High, 3 Medium, 4 Low, 0 none (use 0 to recommend rejection)
# REQUIRED substitution — the intended FULL label id set (wholesale replacement).
LABEL_IDS='["11111111-1111-1111-1111-111111111111"]'
QUERY='mutation($id:String!,$p:Int!,$ids:[String!]){issueUpdate(id:$id,input:{priority:$p,labelIds:$ids}){success}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg id "$ID" --argjson p "$PRIORITY" --argjson ids "$LABEL_IDS" \
    '{query:$q,variables:{id:$id,p:$p,ids:$ids}}')" \
  https://api.linear.app/graphql
```

**mark status** — labels replace **wholesale** here (see *claim*), so the operation is a set edit, not an add/remove pair: fetch the issue's label ids, drop every `factory:*` status label id from the set, append the target status label's id, and send the **full** resulting set with the same `issueUpdate` mutation the claim uses. Clearing the marker is the same edit without the append. Native workflow states are deliberately **not** used for this — they vary per team, and the adapter cannot assume a schema. The status labels must already exist on the team (Bootstrap), their ids come from the same `labels{nodes{id name}}` fetch the claim documents, and every mutation gets the same `errors`-array check. Sending anything less than the full set silently strips the issue's other labels — the exact accident the claim section warns about. These labels are the display cache from the spine's section 6 — passes write them, the sweep corrects them, and no workflow reads them to decide anything.

**create** — needs the team's UUID, not its key; resolve it once:

```bash
LINEAR_TEAM=ENG   # substitute, or export from the config's linear_team
QUERY='query($key:String!){teams(filter:{key:{eq:$key}}){nodes{id key name}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg key "$LINEAR_TEAM" '{query:$q,variables:{key:$key}}')" \
  https://api.linear.app/graphql
```

```bash
TEAM_UUID=00000000-0000-0000-0000-000000000000   # substitute the id resolved above
TITLE='Session cookie survives logout'
DESC='## Problem

## Evidence
'
QUERY='mutation($team:String!,$title:String!,$desc:String!){issueCreate(input:{teamId:$team,title:$title,description:$desc}){success issue{identifier}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg team "$TEAM_UUID" --arg title "$TITLE" --arg desc "$DESC" \
    '{query:$q,variables:{team:$team,title:$title,desc:$desc}}')" \
  https://api.linear.app/graphql
```

Encoding the description through `jq` is what makes a real multi-line spec safe to send — hand-escaped `\\n` sequences inside a `--data` string break as soon as the text contains a quote or backtick.

Apply the type label in a follow-up `issueUpdate`, and leave the triggers off — a human queues the ticket.

**close the loop** — native. `Fixes ENG-123` in the PR body moves the issue to Done on merge when the GitHub integration is enabled. If it is not enabled, the sweep reports the ticket for a manual move rather than pretending it closed.

**read blockers** — Linear's only writeable relation type for this is `blocks` (`blocked_by` is not an API input). Direction: for `type: blocks`, **`issue` blocks `relatedIssue`** — so the blockers of ticket A live in A's **`inverseRelations`** (edges where A is the target). Prefer MCP; GraphQL fallback:

```bash
ID=ENG-123   # substitute — the dependent (identifier or UUID)
QUERY='query($id:String!){issue(id:$id){identifier inverseRelations{nodes{type issue{identifier title state{type}}}}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg id "$ID" '{query:$q,variables:{id:$id}}')" \
  https://api.linear.app/graphql \
  | jq '[.data.issue.inverseRelations.nodes[]
      | select(.type == "blocks")
      | {identifier: .issue.identifier, title: .issue.title, state: .issue.state.type}]'
```

A blocker is still open when its `state.type` is not `completed` or `canceled`. Worked example: if `ENG-100` blocks `ENG-200`, then reading blockers of `ENG-200` returns `ENG-100` via `inverseRelations`; reading `relations` on `ENG-200` would be empty for this edge (that connection holds edges where `ENG-200` is the *source*). Getting this pair swapped is the most common Linear adapter bug — the read-back on *link blocker* exists to catch it.

**link blocker** — record that A is blocked by B. Create with `issueId` = B (the blocker), `relatedIssueId` = A (the dependent), `type: blocks`, then read back A's `inverseRelations`:

```bash
A=ENG-200   # substitute — the dependent (cannot start yet)
B=ENG-100   # substitute — the blocker that must close first
QUERY='mutation($b:String!,$a:String!){issueRelationCreate(input:{issueId:$b,relatedIssueId:$a,type:blocks}){success issueRelation{id type issue{identifier} relatedIssue{identifier}}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg a "$A" --arg b "$B" \
    '{query:$q,variables:{a:$a,b:$b}}')" \
  https://api.linear.app/graphql
# Same errors-array check as every mutation. Then read-back:
QUERY='query($id:String!){issue(id:$id){inverseRelations{nodes{type issue{identifier}}}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "$(jq -n --arg q "$QUERY" --arg id "$A" '{query:$q,variables:{id:$id}}')" \
  https://api.linear.app/graphql \
  | jq --arg b "$B" '[.data.issue.inverseRelations.nodes[]
      | select(.type == "blocks" and .issue.identifier == $b)] | length' \
  | grep -qx 1 \
  || { echo "STOP: read-back failed — $B not in $A's inverseRelations as blocks (direction may be inverted)"; exit 1; }
echo "LINKED $A blocked-by $B"
```

## Bootstrap

Linear labels must exist before they can be attached, so create the eleven (three triggers, three types, four `factory:*` status markers, and `factory:agent-lost`) once in the team's label settings, or via `issueLabelCreate`. Give the status markers one muted colour, distinct from the triggers — they are machine-written record and should look it. Verify the team key in the config resolves to a real team; report and stop if it does not.

## Provenance and maintenance

Last verified 2026-07 (dependency and pinning notes added 2026-08 — **doc-sourced**, no live Linear instance available this pass):

- Personal API keys go in the `Authorization` header without a `Bearer` prefix; OAuth access tokens do use `Bearer`. Re-verify against Linear's API authentication docs.
- Priority is an integer where 0 is none, 1 Urgent, 2 High, 3 Medium, 4 Low — confirm the ordering has not changed before trusting the P0-to-P3 mapping.
- `issueUpdate` replaces `labelIds` wholesale rather than merging; always send the full intended set or labels will be silently dropped.
- PR magic words (`Fixes`, `Closes`, `Resolves` plus the identifier) require the GitHub integration; without it nothing closes automatically.
- **Issue relations (doc):** `issueRelationCreate` accepts `type: blocks` only for this purpose (`blocked_by` is not an API input). Direction: `issueId` blocks `relatedIssueId`. Read a ticket's blockers from `inverseRelations` filtered to `type == "blocks"`. Re-verify with a live pair of scratch issues and the read-back before the first production link.
- **Comment pin:** unsupported on Linear — locator is the `factory heartbeat —` prefix.
