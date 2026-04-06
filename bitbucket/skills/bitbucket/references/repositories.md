# Repositories & Source Browsing — Bitbucket Cloud REST API

Shorthand in examples:
```bash
BB="https://api.bitbucket.org/2.0"
AUTH="--user ${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}"
```

---

## List Repositories in a Workspace

```bash
curl -s $AUTH "$BB/repositories/{workspace}?pagelen=50" \
  | jq '.values[] | {slug: .slug, name: .name, is_private: .is_private, language: .language, updated_on: .updated_on}'
```

**Query parameters:**
- `role` — filter by your role: `admin`, `contributor`, `member`, `owner`
- `q` — filter expression
- `sort` — sort field (e.g., `-updated_on`)
- `pagelen` — results per page (max 100)

**Scope:** `read:repository:bitbucket`

**Filter examples:**
```bash
?role=admin
?q=name~"backend"
?q=is_private=true
?q=name~"api" AND language="python"
```

---

## Get Repository Details

```bash
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}" \
  | jq '{slug: .slug, full_name: .full_name, is_private: .is_private, description: .description, language: .language, mainbranch: .mainbranch.name, size: .size, created_on: .created_on, updated_on: .updated_on, url: .links.html.href}'
```

**Key response fields:**
- `.full_name` — `workspace/repo_slug`
- `.mainbranch.name` — default branch (e.g., `main`)
- `.links.clone` — array with SSH and HTTPS clone URLs
- `.project.key` — project this repo belongs to
- `.size` — repo size in bytes

**Scope:** `read:repository:bitbucket`

---

## Create a Repository

```bash
curl -s $AUTH -X POST "$BB/repositories/{workspace}/{repo_slug}" \
  -H "Content-Type: application/json" \
  -d '{
    "scm": "git",
    "is_private": true,
    "description": "My new repository",
    "project": {
      "key": "PROJ"
    }
  }'
```

The `{repo_slug}` in the URL becomes the repo's slug. The project must be assigned — if omitted, defaults to the oldest project in the workspace. You can assign by project `key` or `UUID`.

**Scope:** `admin:repository:bitbucket`

---

## Update a Repository

```bash
curl -s $AUTH -X PUT "$BB/repositories/{workspace}/{repo_slug}" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Updated description",
    "is_private": false
  }'
```

Supports both update and creation. If the repo name changes, a `Location` header is returned.

**Scope:** `admin:repository:bitbucket`

---

## Delete a Repository

```bash
curl -s $AUTH -X DELETE "$BB/repositories/{workspace}/{repo_slug}"
```

Returns `204 No Content`. **Irreversible** — the repo and all data are permanently deleted. Forks are not affected.

Optional: `?redirect_to=new-repo-slug` to show a redirect message to visitors.

**Scope:** `delete:repository:bitbucket`

---

## List Repository Forks

```bash
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/forks" \
  | jq '.values[] | {full_name: .full_name, owner: .owner.display_name}'
```

Supports `role`, `q`, `sort` query parameters.

## Fork a Repository

```bash
curl -s $AUTH -X POST "$BB/repositories/{workspace}/{repo_slug}/forks" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-fork",
    "workspace": { "slug": "my-workspace" }
  }'
```

---

## Source Browsing

### Get root directory (main branch)

```bash
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/src" \
  | jq '.values[] | {type, path, size}'
```

This redirects to the root directory listing of the main branch. Equivalent to hitting `/src/{main_branch}/` directly.

**Scope:** `read:repository:bitbucket`

### Get file or directory contents

```bash
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/src/{commit}/{path}"
```

- `{commit}` — commit hash, branch name, or tag (e.g., `main`, `develop`, `v1.0.0`, `abc123`)
- `{path}` — file or directory path

**When path is a file:** Returns raw file contents. Content-Type is derived from filename extension. Content-Disposition is `attachment`. If the file is managed by LFS, returns a 301 redirect.

**When path is a directory:** Returns a paginated list of `commit_file` and `commit_directory` objects.

**Query parameters:**
- `format=meta` — returns file/directory metadata as JSON instead of raw content
- `format=rendered` — returns HTML-rendered markup (for supported file types)
- `q` — filter expression (e.g., `size>1024 AND attributes="binary"`)
- `sort` — sort field (e.g., `-size`)
- `max_depth` — recurse into subdirectories (breadth-first, may time out if too large → 555)

**Important:** When listing root directory, a trailing slash is required in the URL.

**Examples:**

```bash
# List root directory of main branch
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/src/main/" \
  | jq '.values[] | {type, path}'

# Get raw file content
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/src/main/README.md"

# Get file at a specific commit
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/src/abc123def/src/config.js"

# Get file at a tag
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/src/v1.2.0/package.json"

# Get file metadata instead of content
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/src/main/README.md?format=meta" \
  | jq '{path, size: .size, type, attributes}'

# List directory recursively (depth 2)
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/src/main/src/?max_depth=2" \
  | jq '.values[] | {type, path}'

# Filter: only binary files over 1kb
curl -s -G $AUTH "$BB/repositories/{workspace}/{repo_slug}/src/main/src/" \
  --data-urlencode 'q=size>1024 AND attributes="binary"'
```

### Create a commit by uploading a file

```bash
# Add or overwrite a file
curl -s $AUTH -X POST "$BB/repositories/{workspace}/{repo_slug}/src" \
  -F /path/to/file.txt=@local-file.txt \
  -F message="Add file.txt" \
  -F branch="main"

# Upload plain text
curl -s $AUTH -X POST "$BB/repositories/{workspace}/{repo_slug}/src" \
  --data-urlencode "/path/to/file.txt=file contents here" \
  --data-urlencode "message=Update file" \
  --data-urlencode "author=Name <email@example.com>"

# Delete files
curl -s $AUTH -X POST "$BB/repositories/{workspace}/{repo_slug}/src" \
  -F files=/file/to/delete.txt \
  -F message="Remove file"
```

**Query parameters (sent as form fields):**
- `message` — commit message
- `author` — commit author (`Name <email>`)
- `branch` — target branch (defaults to main)
- `parents` — parent commit hash
- `files` — paths of files to delete

**Scope:** `write:repository:bitbucket`

### List commits that modified a file

```bash
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/filehistory/{commit}/{path}" \
  | jq '.values[] | {path, date: .commit.date, hash: .commit.hash}'
```

Returns commits in reverse chronological order. Follows renames by default (disable with `?renames=false`).

Supports `q`, `sort` query parameters.

---

## Downloads

Repository download artifacts (release files, build outputs, etc.).

### List download artifacts

```bash
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/downloads"
```

**Scope:** `read:repository:bitbucket`

### Upload a download artifact

```bash
curl -s $AUTH -X POST "$BB/repositories/{workspace}/{repo_slug}/downloads" \
  -F files=@hello.txt
```

Uploading a file with the same name overwrites the existing artifact.

**Scope:** `write:repository:bitbucket`

### Get a download artifact

```bash
# Returns a 302 redirect to the actual file
curl -s -L $AUTH "$BB/repositories/{workspace}/{repo_slug}/downloads/{filename}"
```

Use `-L` to follow the redirect and get the actual file contents.

### Delete a download artifact

```bash
curl -s $AUTH -X DELETE "$BB/repositories/{workspace}/{repo_slug}/downloads/{filename}"
```

Returns `204 No Content`.

**Scope:** `write:repository:bitbucket`

---

## Watchers

```bash
curl -s $AUTH "$BB/repositories/{workspace}/{repo_slug}/watchers" \
  | jq '.values[] | {display_name, account_id}'
```

---

## Code Search (workspace-wide)

```bash
curl -s $AUTH "$BB/workspaces/{workspace}/search/code?search_query={term}&pagelen=10" \
  | jq '.values[] | {file: .file.path, repo: .file.links.self.href, matches: [.content_matches[].lines[].text]}'
```

Searches code across all repositories in the workspace.

**Query parameters:**
- `search_query` — the search term (required)
- `pagelen` — results per page
