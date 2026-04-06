# Output Format

Use this format whether posting to a PR or reporting back for self-review.

**Summary:** 1-2 sentences on scope and overall assessment.

**Findings grouped by severity** (must-fix first). For each finding:
- File path and line number
- What's wrong and why it matters
- Suggested fix or alternative

**Recommendation:**
- **Approve** — no must-fix issues (may have suggestions)
- **Request changes** — has must-fix issues, summarize what needs to change

If no significant issues found: confirm the review is clean with a brief note on what looks good. **Don't invent findings to justify the review.**

---

## Attribution Footer (remote PRs only)

Every comment posted to a remote PR — inline comments, summary comments, and approval/request-changes messages — **must end with this attribution footer**:

```
---
🤖 Drafted by Claude Code, reviewed and approved by @{username}
```

Where `{username}` is the user who approved the review. Get the username from:
- **GitHub:** `gh api user --jq .login`
- **Bitbucket:** the value of `BITBUCKET_USERNAME` env var, or `git config user.name`

The attribution is non-negotiable. It ensures other PR participants know:
1. The comment was AI-drafted (not blindly written by a human)
2. A human reviewed and approved it before posting (not auto-posted)

Do not paraphrase, omit, or hide the attribution. If you cannot determine the username, ask the user before posting.
