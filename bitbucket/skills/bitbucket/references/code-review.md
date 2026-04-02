# Code Review Procedure — Bitbucket Cloud

A step-by-step workflow for reviewing pull requests via the Bitbucket API. This covers how to efficiently navigate diffs of any size and leave useful feedback.

Shorthand used in examples:
```bash
BB="https://api.bitbucket.org/2.0"
AUTH="--user ${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}"
REPO="$BB/repositories/{workspace}/{repo_slug}"
PR_ID={pull_request_id}
```

---

## Step 1: Understand the PR

Fetch metadata to know what you're reviewing — title, description, author, branches, existing approvals.

```bash
curl -s $AUTH "$REPO/pullrequests/$PR_ID" \
  | jq '{
    id: .id, title: .title, description: .description, state: .state,
    author: .author.display_name,
    branch: "\(.source.branch.name) -> \(.destination.branch.name)",
    source_hash: .source.commit.hash, dest_hash: .destination.commit.hash,
    approvals: [.participants[] | select(.approved) | .user.display_name],
    comments: .comment_count
  }'
```

Save the `source_hash` and `dest_hash` — you'll need these for viewing full file contents later.

---

## Step 2: Get the diffstat

The diffstat is lightweight and tells you the full scope — which files changed, how much, what kind of change.

```bash
curl -s $AUTH "$REPO/pullrequests/$PR_ID/diffstat" \
  | jq '.values[] | {status, path: (.new.path // .old.path), lines_added: .lines_added, lines_removed: .lines_removed}'
```

Get totals:

```bash
curl -s $AUTH "$REPO/pullrequests/$PR_ID/diffstat" \
  | jq '{files: (.values | length), added: ([.values[].lines_added] | add), removed: ([.values[].lines_removed] | add)}'
```

---

## Step 3: Save the diff to disk and navigate it

Fetch the full diff once and save to a temp file. This avoids repeated API calls and keeps your working context clean — you can refer back to it as many times as needed without re-fetching.

```bash
curl -s $AUTH "$REPO/pullrequests/$PR_ID/diff" > /tmp/pr-$PR_ID.diff
```

Find file boundaries:

```bash
grep -n "^diff --git" /tmp/pr-$PR_ID.diff
```

Output looks like:
```
1:diff --git a/src/auth.js b/src/auth.js
45:diff --git a/src/config.js b/src/config.js
82:diff --git a/tests/auth.test.js b/tests/auth.test.js
```

Read a specific file's hunk using the Read tool with offset and limit. For example if `auth.js` starts at line 1 and the next file at line 45, read lines 1-44.

Jump to a specific file:

```bash
grep -n "^diff --git.*auth.js" /tmp/pr-$PR_ID.diff
```

Search for patterns across the whole diff:

```bash
grep -n "TODO\|FIXME\|console\.log\|debugger" /tmp/pr-$PR_ID.diff
```

---

## Step 4: Get surrounding context for files

The diff only shows changed lines with a few lines of context. When you need to understand the full function, class, or import structure around a change, fetch the complete file at the source or destination commit.

```bash
# File with the PR's changes applied (source branch)
curl -s $AUTH "$REPO/src/{source_hash}/{file_path}"

# File before the PR's changes (destination branch)
curl -s $AUTH "$REPO/src/{dest_hash}/{file_path}"
```

Use the `source_hash` and `dest_hash` from Step 1. This is especially useful for:
- Seeing the full function body when only a few lines were changed
- Checking imports and dependencies at the top of the file
- Understanding class hierarchies or type definitions
- Verifying that callers of a changed function were updated

---

## Step 5: Review and prioritize

Work through files in priority order:

1. **Core logic** — main application code, business logic, data models
2. **API/interface changes** — public contracts, endpoints, types
3. **Heavily modified files** — most lines changed = most risk
4. **New files** — need careful review for design and correctness
5. **Deleted files** — verify nothing important was dropped
6. **Config/build files** — medium priority
7. **Tests** — verify they cover the changes, but review after production code

What to look for:
- Correctness — does the change actually do what the PR claims?
- Edge cases — null handling, empty inputs, boundary conditions
- Side effects — could this break other functionality?
- Error handling — are failures handled appropriately?
- Missing test coverage — are new code paths tested?

---

## Step 6: Leave comments

### Inline comment on a specific line

```bash
curl -s $AUTH -X POST "$REPO/pullrequests/$PR_ID/comments" \
  -H "Content-Type: application/json" \
  -d '{
    "content": {
      "raw": "This could throw if `response.data` is null."
    },
    "inline": {
      "path": "src/api/handler.ts",
      "to": 42
    }
  }'
```

- `to` — line number in the **new** version (for added/modified lines)
- `from` — line number in the **old** version (for commenting on deleted lines)
- `path` — file path relative to repo root

### Comment on a deleted line

```bash
curl -s $AUTH -X POST "$REPO/pullrequests/$PR_ID/comments" \
  -H "Content-Type: application/json" \
  -d '{
    "content": {
      "raw": "Was this validation intentionally removed?"
    },
    "inline": {
      "path": "src/core/engine.ts",
      "from": 78
    }
  }'
```

### General summary comment

```bash
curl -s $AUTH -X POST "$REPO/pullrequests/$PR_ID/comments" \
  -H "Content-Type: application/json" \
  -d '{
    "content": {
      "raw": "## Review Summary\n\nThe fix looks correct. A couple suggestions inline.\n\n- Consider adding a test for the empty input case\n- The error message could be more descriptive"
    }
  }'
```

---

## Step 7: Approve or hold

If the changes look good:

```bash
curl -s $AUTH -X POST "$REPO/pullrequests/$PR_ID/approve"
```

If changes are needed, don't approve — your inline comments signal what needs to be fixed. Bitbucket Cloud doesn't have a "request changes" state like GitHub. The convention is: leave comments + withhold approval.
