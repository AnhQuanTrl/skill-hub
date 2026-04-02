# API Conventions — Bitbucket Cloud REST API

Detailed patterns for pagination, filtering, and sorting. These apply to all list endpoints.

## Pagination

List responses return:
```json
{
  "page": 1,
  "pagelen": 10,
  "size": 42,
  "next": "https://api.bitbucket.org/2.0/...?page=2",
  "previous": "https://api.bitbucket.org/2.0/...?page=1",
  "values": [...]
}
```

- `pagelen` — items per page. Default 10, max 100. Set with `?pagelen=50`.
- `size` — total number of results (not always present on large collections).
- `next` — URL for the next page. Absent on the last page.

### Paginate through all results

```bash
url="https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests?pagelen=50"
while [ "$url" != "null" ] && [ -n "$url" ]; do
  response=$(curl -s --user "${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}" "$url")
  echo "$response" | jq '.values[]'
  url=$(echo "$response" | jq -r '.next // "null"')
done
```

### Get just the count

```bash
curl -s --user "${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}" \
  "$REPO/pullrequests?pagelen=0&state=OPEN" | jq '.size'
```

## Filtering

Most list endpoints accept a `q` query parameter with field expressions.

### Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Exact match | `state="OPEN"` |
| `!=` | Not equal | `state!="DECLINED"` |
| `~` | Contains | `title~"bugfix"` |
| `>` | Greater than | `updated_on>"2024-01-01"` |
| `<` | Less than | `created_on<"2024-06-01"` |
| `>=` | Greater or equal | `updated_on>="2024-01-01"` |
| `<=` | Less or equal | `created_on<="2024-06-01"` |

### Combining filters

Use `AND` and `OR` (uppercase) to combine expressions:
```
?q=state="OPEN" AND destination.branch.name="main"
?q=state="OPEN" AND (author.display_name="Alice" OR author.display_name="Bob")
```

### Common filter fields by resource

**Pull requests:**
- `state` — `OPEN`, `MERGED`, `DECLINED`, `SUPERSEDED`
- `author.display_name`, `author.account_id`
- `destination.branch.name`, `source.branch.name`
- `title`, `description`
- `created_on`, `updated_on`

**Repositories:**
- `name`, `full_name`
- `is_private`
- `language`
- `created_on`, `updated_on`

**Branches:**
- `name`

### URL encoding

Filter values with spaces or special characters must be URL-encoded:
```bash
# Shell handles this if you quote properly:
curl -s --user "${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}" \
  "$REPO/pullrequests?q=author.display_name=%22Alice%20Smith%22"

# Or use --data-urlencode with -G for GET requests:
curl -s -G --user "${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}" \
  "$REPO/pullrequests" \
  --data-urlencode 'q=author.display_name="Alice Smith" AND state="OPEN"'
```

## Sorting

Use the `sort` query parameter. Prefix with `-` for descending order.

```bash
# Most recently updated first
?sort=-updated_on

# Oldest first
?sort=created_on

# By title alphabetically
?sort=title
```

Sortable fields vary by endpoint but generally include `created_on`, `updated_on`, and resource-specific fields like `title` or `name`.

## Error Response Format

Error responses return JSON:
```json
{
  "type": "error",
  "error": {
    "message": "Repository not found",
    "detail": "There is no repository at path ..."
  }
}
```

Parse errors with:
```bash
response=$(curl -s -w "\n%{http_code}" --user "${BITBUCKET_USERNAME}:${BITBUCKET_API_TOKEN}" "$url")
http_code=$(echo "$response" | tail -1)
body=$(echo "$response" | sed '$d')
if [ "$http_code" -ge 400 ]; then
  echo "$body" | jq -r '.error.message'
fi
```
