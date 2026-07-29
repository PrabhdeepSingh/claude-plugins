# Adapter — Linear

Linear has a native priority scale and native PR magic words, so priority maps to a real field and closing the loop is free. Type and the two triggers are labels.

## Access — MCP first, GraphQL second, stop third

1. **Linear MCP tools available** — use them; they carry the session and avoid credential handling.
2. **No MCP** — the GraphQL API at `https://api.linear.app/graphql` with `LINEAR_API_KEY` in the environment. The key goes in the `Authorization` header directly (no `Bearer` prefix for a personal API key).
3. **Neither** — **stop** and say which is missing. Never improvise auth and never write the key into config or ticket text.

```bash
[ -n "$LINEAR_API_KEY" ] && echo "linear API key present" \
  || echo "STOP: set LINEAR_API_KEY, or enable the Linear MCP server"
```

Every fence below assumes `LINEAR_API_KEY` is exported and uses `--fail --silent --show-error`. `LINEAR_TEAM` comes from the config's `linear_team` key and is **declared in each fence that needs it** — a fresh shell never inherits it, and an empty team filter returns either a GraphQL error or an empty queue that reads as "no work."

Note that GraphQL returns HTTP 200 with an `errors` array for query-level failures, so **check the response body for `errors`** rather than trusting the exit status alone — `--fail` will not catch a rejected mutation, which means a failed claim can look like a successful one.

## Mapping

| Concept | Stored as |
|---|---|
| Trigger | Label `factory-ready-for-spec` / `factory-ready-to-implement` |
| Type | Label `bug` / `enhancement` / `documentation` |
| Priority | Native priority — P0 is 1 (Urgent), P1 is 2 (High), P2 is 3 (Medium), P3 is 4 (Low); 0 means no priority |
| Discussion | Native comments |
| Close the loop | Native — `Fixes ENG-123` in the PR body, with the GitHub integration enabled |

Priority `0` is Linear's "no priority" and is exactly the right value for a ticket recommended for rejection, matching the taxonomy's unset-means-rejected rule.

## The seven operations

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

**fetch** — one issue with its comments and attachments (linked PRs arrive as attachments):

```bash
ID=ENG-123   # substitute the real identifier
QUERY='query($id:String!){issue(id:$id){identifier title description priority state{name type} labels{nodes{name}} comments{nodes{body createdAt user{name}}} attachments{nodes{url title}}}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "{\"query\":\"$QUERY\",\"variables\":{\"id\":\"$ID\"}}" \
  https://api.linear.app/graphql
```

**claim** — run **fetch** first and confirm the trigger label is actually **present**; if it is absent, this is a lost race, not a claim — stop. Then send the label set with the trigger removed, then re-fetch to confirm it is gone.

`issueUpdate` replaces `labelIds` **wholesale**, so `LABEL_IDS` must be the issue's full remaining label id set. Sending the literal `[]` below without substituting **strips every label on the issue**, including its type — destroying classification that the fetch you just ran is the only record of:

```bash
ID=ENG-123
# REQUIRED substitution — the issue's remaining label ids from fetch, trigger excluded.
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

**comment** — Linear comments accept Markdown:

```bash
ID=ENG-123   # substitute
BODY='Triage pass complete. Specification added to the description; awaiting human approval.'
QUERY='mutation($id:String!,$body:String!){commentCreate(input:{issueId:$id,body:$body}){success}}'
curl --fail --silent --show-error --request POST \
  --header "Authorization: $LINEAR_API_KEY" \
  --header "Content-Type: application/json" \
  --data "{\"query\":\"$QUERY\",\"variables\":{\"id\":\"$ID\",\"body\":\"$BODY\"}}" \
  https://api.linear.app/graphql
```

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

## Bootstrap

Linear labels must exist before they can be attached, so create the five (two triggers plus three types) once in the team's label settings, or via `issueLabelCreate`. Verify the team key in the config resolves to a real team; report and stop if it does not.

## Provenance and maintenance

Last verified 2026-07:

- Personal API keys go in the `Authorization` header without a `Bearer` prefix; OAuth access tokens do use `Bearer`. Re-verify against Linear's API authentication docs.
- Priority is an integer where 0 is none, 1 Urgent, 2 High, 3 Medium, 4 Low — confirm the ordering has not changed before trusting the P0-to-P3 mapping.
- `issueUpdate` replaces `labelIds` wholesale rather than merging; always send the full intended set or labels will be silently dropped.
- PR magic words (`Fixes`, `Closes`, `Resolves` plus the identifier) require the GitHub integration; without it nothing closes automatically.
