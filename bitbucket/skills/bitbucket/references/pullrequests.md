# Pull Requests — Bitbucket Cloud REST API

All endpoints use base path: `/repositories/{workspace}/{repo_slug}/pullrequests`

Shorthand in examples:
```bash
BB="https://api.bitbucket.org/2.0"
AUTH="--user ${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}"
REPO="$BB/repositories/{workspace}/{repo_slug}"
```

---

## List Pull Requests

```bash
curl -s $AUTH "$REPO/pullrequests?state=OPEN&pagelen=25" | jq '.values[] | {id, title, state, author: .author.display_name}'
```

**Query parameters:**
- `state` — `OPEN`, `MERGED`, `DECLINED`, `SUPERSEDED` (default: `OPEN`)
- `pagelen` — results per page (max 50)
- `q` — filter expression
- `sort` — sort field (e.g., `-updated_on` for most recently updated first)

**Filter examples:**
```bash
# PRs by a specific author
?q=author.display_name="Alice Smith"

# PRs targeting main branch
?q=destination.branch.name="main"

# PRs updated in the last 7 days
?q=updated_on>2024-01-01T00:00:00

# Combined filters
?q=state="OPEN" AND destination.branch.name="main" AND author.display_name~"Alice"
```

**Useful jq patterns:**
```bash
# Table-like output: ID, title, author, source -> dest
jq -r '.values[] | "\(.id)\t\(.title)\t\(.author.display_name)\t\(.source.branch.name) -> \(.destination.branch.name)"'

# Just count
jq '.size'
```

---

## Get a Pull Request

```bash
curl -s $AUTH "$REPO/pullrequests/{pull_request_id}" | jq '.'
```

**Key response fields:**
- `.id` — PR number
- `.title`, `.description`
- `.state` — OPEN, MERGED, DECLINED, SUPERSEDED
- `.source.branch.name`, `.destination.branch.name`
- `.author.display_name`
- `.reviewers[]` — assigned reviewers
- `.participants[]` — everyone involved, with `.role` and `.approved`
- `.merge_commit.hash` — set after merge
- `.close_source_branch` — whether source branch deletes on merge
- `.created_on`, `.updated_on`
- `.comment_count`, `.task_count`
- `.links.html.href` — web URL

---

## Create a Pull Request

```bash
curl -s $AUTH -X POST "$REPO/pullrequests" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Add login feature",
    "description": "Implements OAuth2 login flow.\n\nJIRA: PROJ-123",
    "source": {
      "branch": { "name": "feature/login" }
    },
    "destination": {
      "branch": { "name": "main" }
    },
    "reviewers": [
      { "account_id": "557058:abc123" },
      { "account_id": "557058:def456" }
    ],
    "close_source_branch": true
  }'
```

**Required fields:**
- `title` — PR title
- `source.branch.name` — source branch

**Optional fields:**
- `destination.branch.name` — defaults to repo's main branch
- `description` — supports Markdown
- `reviewers` — array of `{ "account_id": "..." }` objects
- `close_source_branch` — `true` to delete source branch after merge

**Finding reviewer account IDs:**

To add reviewers, you need their `account_id`. Look it up via the workspace members endpoint:
```bash
curl -s $AUTH "$BB/workspaces/{workspace}/members" | jq '.values[] | {display_name: .user.display_name, account_id: .user.account_id}'
```

Or if you know their username:
```bash
curl -s $AUTH "$BB/users/{username}" | jq '.account_id'
```

---

## Update a Pull Request

```bash
curl -s $AUTH -X PUT "$REPO/pullrequests/{pull_request_id}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Updated title",
    "description": "Updated description",
    "reviewers": [
      { "account_id": "557058:abc123" }
    ],
    "destination": {
      "branch": { "name": "develop" }
    }
  }'
```

Only include the fields you want to change. Omitted fields remain unchanged.

---

## Merge a Pull Request

```bash
curl -s $AUTH -X POST "$REPO/pullrequests/{pull_request_id}/merge" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "pullrequest",
    "merge_strategy": "squash",
    "close_source_branch": true,
    "message": "Squash merge: Add login feature (#42)"
  }'
```

**Merge strategies:**
- `merge_commit` — standard merge commit (default)
- `squash` — squash all commits into one
- `fast_forward` — fast-forward only (fails if not possible)

**Optional fields:**
- `message` — custom commit message
- `close_source_branch` — override the PR's setting

**Handling merge conflicts:**

If merge returns `409 Conflict`, the PR has merge conflicts that need to be resolved in code before retrying.

---

## Approve a Pull Request

```bash
# Approve
curl -s $AUTH -X POST "$REPO/pullrequests/{pull_request_id}/approve"

# Unapprove (remove your approval)
curl -s $AUTH -X DELETE "$REPO/pullrequests/{pull_request_id}/approve"
```

No request body needed. Returns the approval object on success.

---

## Decline a Pull Request

```bash
curl -s $AUTH -X POST "$REPO/pullrequests/{pull_request_id}/decline"
```

This closes the PR without merging. The source branch is preserved.

---

## List PR Comments

```bash
curl -s $AUTH "$REPO/pullrequests/{pull_request_id}/comments?pagelen=50" \
  | jq '.values[] | {id: .id, author: .user.display_name, content: .content.raw, created_on}'
```

Comments include both general comments and inline (file-level) comments. Inline comments have an `.inline` object with `path`, `from` (old line), and `to` (new line).

```bash
# Show only inline comments with file context
jq '.values[] | select(.inline) | {author: .user.display_name, file: .inline.path, line: .inline.to, content: .content.raw}'
```

---

## Add a Comment

### General comment

```bash
curl -s $AUTH -X POST "$REPO/pullrequests/{pull_request_id}/comments" \
  -H "Content-Type: application/json" \
  -d '{
    "content": {
      "raw": "Looks good overall! One question about the error handling in auth.js."
    }
  }'
```

### Inline comment (on a specific file and line)

```bash
curl -s $AUTH -X POST "$REPO/pullrequests/{pull_request_id}/comments" \
  -H "Content-Type: application/json" \
  -d '{
    "content": {
      "raw": "Consider using a constant for this timeout value."
    },
    "inline": {
      "path": "src/auth.js",
      "to": 42
    }
  }'
```

**Inline fields:**
- `path` — file path relative to repo root
- `to` — line number in the new version of the file
- `from` — line number in the old version (for commenting on deleted lines)

### Reply to a comment

```bash
curl -s $AUTH -X POST "$REPO/pullrequests/{pull_request_id}/comments" \
  -H "Content-Type: application/json" \
  -d '{
    "content": {
      "raw": "Good point, fixed in the latest commit."
    },
    "parent": {
      "id": 12345
    }
  }'
```

---

## Get PR Diff

### Raw diff (unified diff format)

```bash
curl -s $AUTH "$REPO/pullrequests/{pull_request_id}/diff"
```

Returns a standard unified diff. Can be large for big PRs.

### Diffstat (summary of changed files)

```bash
curl -s $AUTH "$REPO/pullrequests/{pull_request_id}/diffstat" \
  | jq '.values[] | {status, path: (.new.path // .old.path), lines_added: .lines_added, lines_removed: .lines_removed}'
```

**Diffstat fields per file:**
- `.status` — `added`, `removed`, `modified`, `renamed`
- `.old.path` / `.new.path` — file paths
- `.lines_added`, `.lines_removed` — line counts

Get totals:

```bash
curl -s $AUTH "$REPO/pullrequests/{pull_request_id}/diffstat" \
  | jq '{files: (.values | length), added: ([.values[].lines_added] | add), removed: ([.values[].lines_removed] | add)}'
```

### Get source and destination commit hashes

Useful for fetching full file contents at either side of the PR:

```bash
curl -s $AUTH "$REPO/pullrequests/{pull_request_id}" \
  | jq '{source_hash: .source.commit.hash, dest_hash: .destination.commit.hash}'
```

### View file content at a specific commit

Fetch the full file at the PR's source or destination commit for surrounding context:

```bash
# File at source (the PR's changes)
curl -s $AUTH "$REPO/src/{source_commit_hash}/{file_path}"

# File at destination (before the PR's changes)
curl -s $AUTH "$REPO/src/{destination_commit_hash}/{file_path}"
```

---

## Common Patterns

### Get all open PRs assigned to me for review

```bash
curl -s $AUTH "$REPO/pullrequests?state=OPEN&pagelen=50" \
  | jq --arg me "$BITBUCKET_USERNAME" '.values[] | select(.reviewers[]?.account_id == $me) | {id, title, author: .author.display_name}'
```

### Paginate through all PRs

```bash
url="$REPO/pullrequests?state=MERGED&pagelen=50"
while [ "$url" != "null" ] && [ -n "$url" ]; do
  response=$(curl -s $AUTH "$url")
  echo "$response" | jq '.values[] | {id, title}'
  url=$(echo "$response" | jq -r '.next // "null"')
done
```

### Quick PR summary

```bash
PR_ID=42
pr=$(curl -s $AUTH "$REPO/pullrequests/$PR_ID")
echo "$pr" | jq '{
  id: .id,
  title: .title,
  state: .state,
  author: .author.display_name,
  branch: "\(.source.branch.name) -> \(.destination.branch.name)",
  reviewers: [.reviewers[].display_name],
  approvals: [.participants[] | select(.approved) | .user.display_name],
  comments: .comment_count,
  url: .links.html.href
}'
```
