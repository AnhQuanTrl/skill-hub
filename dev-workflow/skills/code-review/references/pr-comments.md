# Posting Review Comments to a PR

When the user asks to post review findings as PR comments, create **one comment per finding** — not a single summary comment. The goal is maximum granularity so the PR author can address each issue individually.

## Contents

- [What to comment on](#what-to-comment-on) — which findings get a comment
- [Comment format](#comment-format) — body shape, priority labels, examples
- [Attribution footer (required)](#attribution-footer-required) — every comment ends with this
- [Posting on GitHub](#posting-on-github) — `gh` + Reviews API; jump here when remote is `github.com`
- [Posting on Bitbucket](#posting-on-bitbucket) — REST API + curl; jump here when remote is `bitbucket.org`

**Detect the platform first**: `git remote get-url origin`. Then jump to the matching posting section. Format and attribution rules above apply to both platforms — only the API commands differ.

---

## What to comment on

Create a comment for **every** item from the review output:

- Every Critical issue
- Every Important issue
- Every Suggestion
- Every Silent Failure Audit entry
- Every Test Gap
- Every dimension rating that scored below 9/10 (as a comment on the most relevant file/line for that deduction)
- Every type rating that scored below 9/10 on any dimension (as a comment on the type definition)

**More comments are better.** Each finding should be its own comment so it can be resolved independently.

---

## Comment format

Every comment must include the **aspect** (which reviewer raised it) and the **priority** as a prefix, and end with the [attribution footer](#attribution-footer-required):

```
**[aspect]** **priority**

<description>

**Fix:** <recommendation>

---
🤖 Drafted by Claude Code, reviewed and approved by @{username}
```

### Priority labels

| From agent | Priority mapping |
|---|---|
| review-correctness | `🔴 Critical` (90–100) or `🟠 Important` (80–89) |
| review-security | `🔴 Critical`, `🟠 High`, or `🟡 Medium` |
| review-reliability | `🔴 Critical`, `🟠 High`, or `🟡 Medium` |
| review-tests | `🔴 Critical` (9–10) or `🟠 Important` (7–8) |
| review-design | `🔴 Critical` (90–100) or `🟠 Important` (80–89) |
| review-types | `🟠 Important` (any dimension below 7) or `🟡 Suggestion` (7–8) |
| Any suggestion | `💬 Suggestion` |

### Examples

A security finding:
```
**[Security]** **🔴 Critical**

No authorization checks on image endpoints. Any authenticated user can upload to any brand, read any image, and delete any image.

**Fix:** Add `@CurrentUser()`, pass principal through commands, add brand-scoped permission checks in handlers.

---
🤖 Drafted by Claude Code, reviewed and approved by @arthur
```

A design rating comment:
```
**[Design]** **🟠 Important** — Architecture adherence: 7/10

Route design is inconsistent: POST uses `/brands/:brandId/uploads/images`, GET uses `/images/:imageId`, DELETE uses `/brands/:brandId/images/:imageId`. This makes the API surface harder to reason about.

**Fix:** Unify under a consistent resource-oriented pattern.

---
🤖 Drafted by Claude Code, reviewed and approved by @arthur
```

A type rating comment:
```
**[Types]** **🟡 Suggestion** — Image entity: Encapsulation 5/10

Entity exposes all fields as public properties. `mimeType` accepts `string` but should be a union of valid MIME types. `size` has no non-negative constraint at the type level.

**Fix:** Use a narrow union for `mimeType`. Consider a branded `Bytes` type for size.

---
🤖 Drafted by Claude Code, reviewed and approved by @arthur
```

A test gap:
```
**[Tests]** **🟠 Important** — Criticality 7/10

No test for oversized file upload. Acceptance criteria requires a 413 response test but none exists.

**Fix:** Add integration test uploading a file > 5MB and asserting 413 with `file-too-large` error code.

---
🤖 Drafted by Claude Code, reviewed and approved by @arthur
```

---

## Attribution footer (required)

Every comment posted to a remote PR — inline, standalone, approval, request-changes — **must end with**:

```
---
🤖 Drafted by Claude Code, reviewed and approved by @{username}
```

Get `{username}` based on platform:
- **GitHub**: `gh api user --jq .login`
- **Bitbucket**: `${BITBUCKET_USERNAME:-$(git config user.name)}`

The attribution is non-negotiable. It tells readers the comment was AI-drafted but a human reviewed and approved it before it landed. Do not paraphrase, omit, or hide it.

---

## Posting on GitHub

### Constraints

**Use the PR reviews API — never issue comments.** GitHub splits PR comments across three API streams:

- `/pulls/:pr/comments` — inline review comments (tied to file+line)
- `/pulls/:pr/reviews` — review-level comments (standalone, no file anchor)
- `/issues/:pr/comments` — general issue comments (NOT part of any review)

A downstream reader (e.g., `/resolve-pr-comments`) only scans the first two. Posting via `gh pr comment` creates issue comments that are invisible to that workflow and can silently be missed. **Never use `gh pr comment` for review findings.**

### Inline comments (finding has a specific file+line)

**Constraints:**
- The `line` must be **inside a diff hunk** (visible in `gh pr diff`). Lines outside any hunk will be rejected with `"line could not be resolved"`. If a finding targets a line not in the diff, post it as a standalone review instead.
- The `line` field must be an integer. The `gh` CLI's `-f` flag sends strings, so use JSON input via `--input -` to ensure correct types.

```bash
USERNAME=$(gh api user --jq .login)

cat <<ENDJSON | jq --arg sha "$(gh pr view {pr_number} --json headRefOid --jq '.headRefOid')" '.commit_id = $sha' \
  | gh api repos/{owner}/{repo}/pulls/{pr_number}/comments --method POST --input - --jq '.id'
{
  "body": "<comment body, ending with attribution footer for @${USERNAME}>",
  "path": "<file path>",
  "commit_id": "PLACEHOLDER",
  "line": <line_number>,
  "side": "RIGHT"
}
ENDJSON
```

### Standalone review comments (finding has no specific line)

For missing tests, general ratings, cross-cutting concerns, or anything without a natural file anchor, post as a standalone review (body only, no inline comments array):

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --method POST \
  -f event="COMMENT" \
  -f body="<comment body, ending with attribution footer>"
```

This creates one review per finding, keeping the "one comment per finding" granularity.

### Approve / request changes

```bash
# Approve
gh pr review {pr_number} --approve \
  --body "Looks good — minor suggestions left as comments.

---
🤖 Drafted by Claude Code, reviewed and approved by @${USERNAME}"

# Request changes
gh pr review {pr_number} --request-changes \
  --body "Summary of required changes.

---
🤖 Drafted by Claude Code, reviewed and approved by @${USERNAME}"
```

### Workflow (GitHub)

1. Get the PR's head commit SHA: `gh pr view {num} --json headRefOid --jq '.headRefOid'`
2. Get the username for the attribution footer: `USERNAME=$(gh api user --jq .login)`
3. For each finding with a specific `file:line`, **verify the line is inside a diff hunk** before posting. Fetch the branch and check with `git show FETCH_HEAD:<file> | grep -n <pattern>`, then cross-reference against the diff hunks. If the line is outside all hunks, fall back to a standalone review.
4. Post inline comments via `/pulls/:pr/comments` (JSON input, integer `line`).
5. For dimension ratings below 9/10, post an inline comment on the most relevant line **that is in the diff**.
6. For findings without a specific line, or where the target line is not in the diff (missing tests, general suggestions, cross-cutting design notes), post as a **standalone review** via `/pulls/:pr/reviews` with `event: "COMMENT"` and body only.
7. **Never use `gh pr comment`** for review findings — it creates an issue comment outside the reviews API.
8. Report back how many comments were posted.

---

## Posting on Bitbucket

### Constraints

**All review feedback goes through the PR comments API.** Bitbucket Cloud doesn't have GitHub's three-stream split — there's a single comments endpoint and an `inline` field that decides whether the comment is anchored to a file+line or stands alone.

- Inline comments need an `inline` object with `path` + `to` (line number on the destination/new side)
- Standalone comments simply omit the `inline` field
- Approve and request-changes are separate POST endpoints, **not body-bearing** — if you want a summary message attached to an approval, post it as a standalone comment first, then call approve

**Auth**: `BITBUCKET_USERNAME` + `BITBUCKET_API_TOKEN` env vars (HTTP Basic). The username for attribution may differ from auth username — prefer `git config user.name` if it's the human author, otherwise fall back to `BITBUCKET_USERNAME`.

### Inline comments (finding has a specific file+line)

The `to` field is the line on the new (destination) side of the diff. For lines on the old side, use `from` instead. Line must be inside a diff hunk; if not, post as a standalone comment.

```bash
BB="https://api.bitbucket.org/2.0"
AUTH="--user ${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}"
USERNAME="${BITBUCKET_USERNAME:-$(git config user.name)}"

COMMENT_BODY=$(jq -n \
  --arg text "<comment body, ending with attribution footer for @${USERNAME}>" \
  --arg path "<file path>" \
  --argjson line <line_number> \
  '{content: {raw: $text}, inline: {to: $line, path: $path}}')

curl -s $AUTH -X POST \
  "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/comments" \
  -H "Content-Type: application/json" \
  -d "$COMMENT_BODY"
```

### Standalone comments (finding has no specific line)

Same endpoint, just omit the `inline` field:

```bash
COMMENT_BODY=$(jq -n \
  --arg text "<comment body, ending with attribution footer>" \
  '{content: {raw: $text}}')

curl -s $AUTH -X POST \
  "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/comments" \
  -H "Content-Type: application/json" \
  -d "$COMMENT_BODY"
```

### Approve / request changes

Bitbucket's approve and request-changes endpoints don't accept a body. To attach a summary message, post a standalone comment first, then call the action endpoint.

```bash
# Approve
curl -s $AUTH -X POST \
  "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/approve"

# Request changes
curl -s $AUTH -X POST \
  "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}/request-changes"
```

### Workflow (Bitbucket)

1. Get PR metadata including the source/destination commit hashes:
   ```bash
   curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/pullrequests/{pr_id}" \
     | jq '{title, source_hash: .source.commit.hash, dest_hash: .destination.commit.hash}'
   ```
2. Set the username for the attribution footer: `USERNAME="${BITBUCKET_USERNAME:-$(git config user.name)}"`
3. For each finding with a specific `file:line`, verify the line is inside a diff hunk:
   ```bash
   git diff {dest_hash}...{source_hash} -- <file> | grep -n .
   ```
   If outside any hunk, fall back to a standalone comment.
4. Post inline comments with `inline: {to: <line>, path: <file>}`.
5. For dimension ratings below 9/10, post an inline comment on the most relevant in-diff line.
6. For findings without a specific line, post a standalone comment (omit the `inline` field).
7. If the user wants to approve or request changes, post a standalone summary comment first, then call the action endpoint.
8. Report back how many comments were posted.
