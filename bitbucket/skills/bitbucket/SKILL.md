---
name: bitbucket
description: Interact with Bitbucket Cloud via its REST API v2.0. Use this skill whenever the user mentions Bitbucket — pull requests, repositories, pipelines, branches, code review, or when git remotes point to bitbucket.org. Covers PR management, repo operations, pipeline triggers, and branch/commit workflows. Even if the user just says "check my PRs" or "merge that PR" and the context is Bitbucket, this skill applies. Don't use for Bitbucket Server/Data Center (on-prem) — those have a different API.
---

# Bitbucket Cloud REST API

Bitbucket Cloud has no official CLI. All operations go through its REST API v2.0 using `curl` and `jq`.

## Setup

### 1. Check credentials

```bash
if [ -z "$BITBUCKET_USERNAME" ] || [ -z "$BITBUCKET_API_TOKEN" ]; then
  echo "Set BITBUCKET_USERNAME (Atlassian email) and BITBUCKET_API_TOKEN"
  exit 1
fi
```

If missing, tell the user to create an API token at:
Profile > Account Settings > Security > Create API token with scopes.
Minimum scopes: Repositories (Read), Pull Requests (Read/Write). Add Pipelines (Read/Write) for pipeline operations.

### 2. Identify workspace and repo

If the user doesn't specify, detect from git remote:
```bash
remote=$(git remote get-url origin 2>/dev/null)
# git@bitbucket.org:myworkspace/my-repo.git  → workspace=myworkspace repo_slug=my-repo
# https://bitbucket.org/myworkspace/my-repo   → workspace=myworkspace repo_slug=my-repo
```

### 3. Make requests

```bash
curl -s --user "${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}" \
  -H "Content-Type: application/json" \
  "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/..."
```

## API Conventions

- **Base URL:** `https://api.bitbucket.org/2.0`
- **Path pattern:** `/repositories/{workspace}/{repo_slug}/...`
- **Request bodies:** JSON
- **Pagination:** Responses contain `values` array, `pagelen` (default 10, max 100), and `next` URL. Set page size with `?pagelen=50`.
- **Filtering:** `?q=state="OPEN"` — supports `=`, `!=`, `~` (contains), `>`, `<`, `AND`, `OR`.
- **Sorting:** `?sort=-updated_on` — prefix `-` for descending.
- **Errors:** `401` = bad credentials. `403` = insufficient token scopes. `404` = wrong workspace/repo/ID. `409` = merge conflict.

For pagination loops, filter operators, field names per resource, URL encoding patterns, and error parsing — read `references/api-conventions.md`.

## Reference Files

### API References

Endpoint documentation — consult when making specific API calls.

- **`references/pullrequests.md`** — PR endpoints: list, create, update, merge, approve, decline, comment, diff, diffstat, and viewing file contents at specific commits. Use when performing any pull request operation.

Future: repositories & source browsing, pipelines, branches/tags/commits (see `TODO.md`).

### General Patterns

- **`references/api-conventions.md`** — Pagination loops, filter query syntax and operators, field names per resource, URL encoding, sorting, and error response parsing. Use when you need to paginate through results, build complex filter queries, or handle errors beyond the basics in the conventions section above.

### Workflows

Step-by-step procedures that combine multiple API calls for a specific task.

- **`references/code-review.md`** — How to efficiently review a PR's code changes: fetch metadata, assess diff scope, save diff to disk for repeated reference, navigate file-by-file, fetch full file contents for surrounding context, and leave inline comments. Use when the user asks to review, check, or look at code changes in a PR.
