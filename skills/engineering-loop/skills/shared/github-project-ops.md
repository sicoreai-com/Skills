# Shared Reference — GitHub Project Operations

All loop skills read and write the same GitHub Projects (v2) board. This file
is the single source of truth for how to do that with the `gh` CLI. Do not
improvise different commands per skill — use these recipes.

## 0. Load config

Read `loop-config.md` from the repo root. Args passed to the skill
(`org=...`, `project=...`) override config values. If `org` or `project` is
still a placeholder (`YOUR_ORG`), stop and ask the human — never guess.

```bash
ORG=$(awk -F': *' '$1=="org"{print $2}' loop-config.md)
PROJECT_NAME=$(awk -F': *' '$1=="project"{print $2}' loop-config.md)
REPO=$(awk -F': *' '$1=="repo"{print $2}' loop-config.md)
```

## 1. Resolve project number, project ID, field IDs (once per run)

Cache the results in your scratchpad — do NOT re-query per item.

```bash
# Project number + node ID
gh project list --owner "$ORG" --format json --limit 100 \
  | jq -r --arg t "$PROJECT_NAME" '.projects[] | select(.title==$t) | "\(.number) \(.id)"'

# All fields with their IDs and single-select option IDs
gh project field-list "$NUM" --owner "$ORG" --format json > /tmp/fields.json
```

Fields you will need (look them up by name in fields.json; names must match
the board exactly): `Status`, `Priority`, `Effort`, `Estimate Token`,
`Budget Token`, `Analysis Token Usage`, `Implementation Token Usage`,
`Retrospective Token Usage`, `CI Token Usage`, `Detailed Description`,
`Failure Log`.

Status options: Create, Open, Pending merge, Merged, Resolved, Closed,
Feedback, Rejected, Review, Approved, Failed, Escalation.

## 2. Read all items with status, issue type, and field values

`gh project item-list` does not expose the built-in issue **Type**
(Feature/Bug/Task), so use this GraphQL query as the canonical read. Paginate
with `endCursor` until `hasNextPage` is false.

```bash
gh api graphql -f org="$ORG" -F number="$NUM" -f query='
query($org: String!, $number: Int!, $cursor: String) {
  organization(login: $org) {
    projectV2(number: $number) {
      id
      items(first: 100, after: $cursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          content {
            ... on Issue {
              id number title url body state createdAt
              issueType { name }
            }
          }
          fieldValues(first: 30) {
            nodes {
              ... on ProjectV2ItemFieldSingleSelectValue {
                name field { ... on ProjectV2FieldCommon { name } } }
              ... on ProjectV2ItemFieldNumberValue {
                number field { ... on ProjectV2FieldCommon { name } } }
              ... on ProjectV2ItemFieldTextValue {
                text field { ... on ProjectV2FieldCommon { name } } }
            }
          }
        }
      }
    }
  }
}'
```

If the project is user-owned rather than org-owned, swap
`organization(login:)` for `user(login:)`.

Filter in `jq`, not by re-querying (e.g. items where
`issueType.name=="Bug" or =="Feature"` and Status field value `=="Open"`).

## 3. Update fields

```bash
# Single-select (Status / Priority / Effort) — use the option ID from fields.json
gh project item-edit --project-id "$PROJECT_ID" --id "$ITEM_ID" \
  --field-id "$FIELD_ID" --single-select-option-id "$OPTION_ID"

# Number fields (all token fields)
gh project item-edit --project-id "$PROJECT_ID" --id "$ITEM_ID" \
  --field-id "$FIELD_ID" --number 12345

# Text fields (Detailed Description / Failure Log)
gh project item-edit --project-id "$PROJECT_ID" --id "$ITEM_ID" \
  --field-id "$FIELD_ID" --text "$NEW_TEXT"
```

**Appending to a text field**: text fields have no append operation. Read the
current value from the item query (step 2), concatenate
`current + "\n\n---\n[<skill> <ISO timestamp>]\n" + addition`, and write the
whole thing back with `--text`. Never overwrite existing content.

**HARD LIMIT — 1024 characters per text field** (GraphQL rejects longer with
"Column value must be a valid value for text column"). Before writing, check
the combined length. If it would exceed 1024: post the FULL accumulated
history as a comment on the issue (`gh issue comment`), then set the field to
a compact form — latest entry only, prefixed with
`(history in issue comments)`. Keep individual entries terse so this happens
rarely.

**Accumulating a number field** (e.g. CI Token Usage across sweeper rounds):
read current value, add, write the sum.

**Appending to the issue body**:

```bash
CUR=$(gh issue view "$ISSUE_NUM" --repo "$REPO" --json body -q .body)
printf '%s\n\n%s\n' "$CUR" "PR: $PR_URL" | gh issue edit "$ISSUE_NUM" --repo "$REPO" --body-file -
```

## 4. Create an issue with a Type and add it to the project

```bash
URL=$(gh issue create --repo "$REPO" --title "$TITLE" --body "$BODY")

# Set the built-in issue type (Feature/Bug/Task)
TYPE_ID=$(gh api graphql -f owner="${REPO%%/*}" -f name="${REPO##*/}" -f query='
  query($owner:String!,$name:String!){ repository(owner:$owner,name:$name){
    issueTypes(first:10){ nodes{ id name } } } }' \
  | jq -r '.data.repository.issueTypes.nodes[] | select(.name=="Task") | .id')
ISSUE_ID=$(gh issue view "$URL" --json id -q .id)
gh api graphql -f id="$ISSUE_ID" -f typeId="$TYPE_ID" -f query='
  mutation($id:ID!,$typeId:ID!){ updateIssue(input:{id:$id, issueTypeId:$typeId}){ issue{id} } }'

# Add to the project (returns the project item ID for field edits)
gh project item-add "$NUM" --owner "$ORG" --url "$URL" --format json | jq -r .id
```

## 5. STATUS.md gate

Skills that are gated check the file at the top of every run **and** between
work items. Control words only count at the **start of a line** (`STOP` or
`STOP: reason`) — this keeps documentation/comments inside the file from
tripping the gate:

```bash
grep -qE '^STOP\b'      STATUS.md && echo blocked-stop
grep -qE '^(STOP|CI)\b' STATUS.md && echo blocked-stop-or-ci   # implementation gate
```

When writing a control word, always write it at column 0 on its own line.

If blocked: log one line saying which word blocked you and exit cleanly.
Skipping a round is normal operation, not an error. Never remove a control
word you did not write (exception: the CI skill removes `CI` lines it wrote;
only humans remove `STOP`).

## 6. Token usage accounting

Every skill records the tokens it spent per issue into that issue's stage
field (`Analysis` / `Implementation` / `Retrospective` / `CI` Token Usage):

- When work was done by sub-agents, use the token counts the harness reports
  for those agent runs.
- For work done inline, estimate from the volume of context read and output
  produced, and round up.
- Fields are cumulative across rounds: read current value, add, write back.
- Record the number as an integer number of tokens (not thousands).

## 7. General rules

- Batch reads (one paginated item query per run), then edit item by item.
- Every status transition must follow the state machines in `LOOP.md` —
  never invent a transition.
- On a failed `gh` call, retry once; if it fails again, report the item as
  skipped in your run summary rather than aborting the whole run.
