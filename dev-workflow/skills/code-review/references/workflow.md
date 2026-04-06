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

# Read the full diff (Claude Code pages large outputs to disk automatically)
git diff {dest_hash}...{source_hash}
```

For very large diffs that you need to grep across, save explicitly:
```bash
git diff {dest_hash}...{source_hash} > /tmp/pr-{id}.diff
grep -n "^diff --git" /tmp/pr-{id}.diff   # find file boundaries
```

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

**Every posted comment body must end with the attribution footer** (see `references/output-format.md`). The examples below assume `body` already contains the footer appended.

### Get the approving user's identity first

```bash
# GitHub
USERNAME=$(gh api user --jq .login)

# Bitbucket
USERNAME="${BITBUCKET_USERNAME:-$(git config user.name)}"

FOOTER=$'\n\n---\n🤖 Drafted by Claude Code, reviewed and approved by @'"${USERNAME}"
```

### GitHub

```bash
# Inline comment on a specific line
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  -f body="${COMMENT_TEXT}${FOOTER}" \
  -f commit_id="{source_hash}" \
  -f path="path/to/file" \
  -F line={line_number} \
  -f side="RIGHT"

# Approve
gh pr review {pr_number} --approve \
  --body "Looks good — minor suggestions left as comments.${FOOTER}"

# Request changes
gh pr review {pr_number} --request-changes \
  --body "Summary of required changes.${FOOTER}"
```

### Bitbucket

```bash
BB="https://api.bitbucket.org/2.0"
AUTH="--user ${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}"

# Inline comment (use jq to safely build JSON with the footer)
COMMENT_BODY=$(jq -n --arg text "${COMMENT_TEXT}${FOOTER}" --arg path "path/to/file" --argjson line {line_number} \
  '{content: {raw: $text}, inline: {to: $line, path: $path}}')

curl -s $AUTH -X POST \
  "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/comments" \
  -H "Content-Type: application/json" \
  -d "$COMMENT_BODY"

# Approve (Bitbucket has no approve message — post a separate comment with the footer first if needed)
curl -s $AUTH -X POST \
  "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/approve"
```

### Self-review

No API calls. Report findings directly to the user and offer to fix issues locally.

---

## Subagent Dispatch (Parallel Review)

For large PRs where the main agent has confirmed review aspects with the user (see SKILL.md step 3), dispatch one subagent per agreed aspect in parallel.

### Pattern

Spawn all subagents in a single message with multiple Task tool calls so they run concurrently. Each subagent prompt should contain:

1. **The aspect** — the single review angle this subagent owns (e.g., "security & authorization", "correctness & state")
2. **The PR context** — title, description, source/dest branches, commit hashes
3. **The diff** — either embedded inline, or instructions to fetch it (`git diff {dest_hash}...{source_hash}`)
4. **The relevant criteria** — paste the matching tier from `references/review-criteria.md` plus any project rules from `CLAUDE.md` that apply to this aspect
5. **The philosophy reminder** — "review the change not the codebase, quality over quantity, filter before reporting, drop low-confidence findings"
6. **Output instructions** — return findings as a structured list: `severity | file:line | what's wrong | why it matters | suggested fix`

### Example subagent prompt skeleton

```
You are reviewing a PR for one specific aspect: {ASPECT}.

PR title: {TITLE}
PR description: {DESCRIPTION}
Source: {SOURCE_BRANCH} ({SOURCE_HASH})
Target: {DEST_BRANCH} ({DEST_HASH})

Diff:
{DIFF or "Run: git diff {DEST_HASH}...{SOURCE_HASH}"}

Review criteria for this aspect:
{PASTE FROM review-criteria.md + project rules}

Philosophy: review the change not the codebase. Quality over quantity. Be specific
and actionable. Drop low-confidence findings. A review with zero findings for this
aspect is a valid result — do not invent issues.

Return your findings as a list:
- severity (must-fix / should-fix / suggestion)
- file:line
- what's wrong
- why it matters
- suggested fix
```

### Aggregation (main agent)

After all subagents return:

1. **Collect** all findings into a single list.
2. **Deduplicate** — the same code issue may surface from multiple aspects (e.g., a missing null check could come from "correctness" and "error handling"). Keep the version with the clearest reasoning.
3. **Reclassify if needed** — if two subagents disagree on severity, use your judgment based on the tiers.
4. **Present** the unified review using the Output Format above.
