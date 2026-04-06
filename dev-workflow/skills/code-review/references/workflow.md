# Code Review Workflow

Git commands for reading diffs and platform-specific API calls for posting comments.

---

## Platform Detection

Detect platform from git remote to determine how to post comments:

```bash
remote=$(git remote get-url origin 2>/dev/null)
# Contains github.com → use gh CLI
# Contains bitbucket.org → use Bitbucket REST API (curl)
```

For **self-review** (no PR, just local changes): skip API calls — report findings directly and offer to fix locally.

---

## Getting the Diff

### PR review — get commit hashes

*GitHub:*
```bash
gh pr view {pr_number} --json title,body,headRefName,baseRefName,headRefOid,baseRefOid
```

*Bitbucket:*
```bash
BB="https://api.bitbucket.org/2.0"
AUTH="--user ${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}"

curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}" \
  | jq '{title, description: .description, source_branch: .source.branch.name, dest_branch: .destination.branch.name, source_hash: .source.commit.hash, dest_hash: .destination.commit.hash}'
```

### Self-review — no PR

```bash
base=$(git merge-base HEAD main)
head=$(git rev-parse HEAD)
```

### Fetch and diff

```bash
# Ensure commits are local
git fetch origin {source_branch} {dest_branch}

# Scope check
git diff --stat {dest_hash}...{source_hash}

# Save full diff to disk
git diff {dest_hash}...{source_hash} > /tmp/pr-{id}.diff
```

---

## Navigating Large Diffs

```bash
# Find file boundaries in saved diff
grep -n "^diff --git" /tmp/pr-{id}.diff
```

Use `Read` tool with offset/limit based on boundary line numbers to read one file's diff at a time.

For very large diffs (>2000 lines), work file-by-file rather than reading the entire diff at once.

---

## Full File Context

When the diff alone is ambiguous:

```bash
# New version (source branch)
git show {source_hash}:path/to/file

# Old version (dest/base branch)
git show {dest_hash}:path/to/file
```

---

## Posting Comments

### GitHub

```bash
# Inline comment on a specific line
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  -f body="comment text" \
  -f commit_id="{source_hash}" \
  -f path="path/to/file" \
  -F line={line_number} \
  -f side="RIGHT"

# Approve
gh pr review {pr_number} --approve --body "Looks good — minor suggestions left as comments."

# Request changes
gh pr review {pr_number} --request-changes --body "Summary of required changes."
```

### Bitbucket

```bash
BB="https://api.bitbucket.org/2.0"
AUTH="--user ${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}"

# Inline comment
curl -s $AUTH -X POST \
  "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/comments" \
  -H "Content-Type: application/json" \
  -d '{
    "content": {"raw": "comment text"},
    "inline": {"to": {line_number}, "path": "path/to/file"}
  }'

# Approve
curl -s $AUTH -X POST \
  "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/approve"
```

### Self-review

No API calls. Report findings directly to the user and offer to fix issues locally.
