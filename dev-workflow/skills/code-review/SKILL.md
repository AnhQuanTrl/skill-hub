---
name: code-review
user_invocable: true
description: >
  Run a comprehensive code review of a pull request, branch, or local diff using
  specialized review agents that each focus on a different engineering concern
  (correctness, security, reliability, tests, design, type design). Use this skill
  when the user asks to "review a PR", "review my changes", "check this before I
  commit", "review the diff", or any variation. Accepts a PR number/URL, a branch
  name, or defaults to the current branch vs main.
---

# Code Review

Orchestrates a multi-perspective review by dispatching specialized agents in parallel and aggregating their reports.

## Review Agents

| Agent | Concern | Scoring |
|---|---|---|
| `review-correctness` | Requirements compliance, logic bugs, contract violations | 0–100 confidence, report ≥ 80 |
| `review-security` | AuthN/authZ, data exposure, input validation | CRITICAL / HIGH / MEDIUM |
| `review-reliability` | Error handling, silent failures, data integrity | CRITICAL / HIGH / MEDIUM + Silent Failure Audit |
| `review-tests` | Test coverage and quality | 1–10 criticality, report ≥ 7 |
| `review-design` | Architecture, conventions, clarity | 3-dim 1–10 ratings + per-finding 0–100 |
| `review-types` | Type design (invariants, encapsulation, usefulness) | 4-dim 1–10 ratings per type |

## Scope Rules

Apply to every agent and to the orchestrator's aggregation. Things on this list are **dropped, not surfaced** — they're out of scope, not low-signal:

- **Style and formatting** — linters handle this
- **Pre-existing issues that are already acknowledged** — tracked in a TODO/issue, called out in a code comment, or covered by a project rule. Unacknowledged pre-existing issues are fair to surface — a PR review is a useful moment to catch them; flag them as such (clearly attributed to the existing code, not the diff) rather than blocking the PR on them.
- **No-op or whitespace-only changes** — even if they "muddy git blame", they're harmless
- **Speculative issues** without evidence (e.g., "might break if someone passes X" — only flag if X can actually happen)
- **Missing features** — the PR does what it says; don't scope-creep
- **Things project rules explicitly allow** — if `CLAUDE.md` or `.claude/rules/` says it's fine, it's fine

## Workflow

### 1. Gather the change set

Parse the user's request:

- **GitHub PR**: `gh pr view <num> --json number,title,headRefName,baseRefName,body,files` + `gh pr diff <num>`
- **Branch**: `git fetch origin main` then `git diff origin/main...<branch>`
- **Local changes**: `git diff HEAD` (fall back to `git diff origin/main...HEAD`)
- **Commit range**: `git diff <range>`

Produce: a **file list** (with added/modified/deleted status) and the **raw diff**.

### 2. Extract requirements context

If the change set is a PR, extract the linked issue or PRD:

- Check the PR body for issue references (`#28`, `Closes #28`, etc.)
- If found, fetch the issue body: `gh issue view <num>`
- Pass the requirements (acceptance criteria, expected behavior) to `review-correctness` so it can verify the implementation matches what was asked for

If no linked issue/PRD, `review-correctness` still reviews logic and contracts but skips requirement compliance.

### 3. Confirm review aspects with the user

Before dispatching, **decide which agents are actually relevant for this diff** and get user confirmation. Don't default to "fire everything" — running agents that have nothing meaningful to review wastes context and produces filler.

Look at the diffstat (and skim file paths) and judge for each agent whether it has real work to do on *this* change. Be lean. A 10-line typo fix doesn't need 6 agents; a 5-line auth change might only need security. Use the agent table at the top of this file to remind yourself what each one cares about.

Then present the proposed set with a one-line reason per agent, and ask for confirmation. Example for a small auth-related change:

> For this PR (12 lines, touches `middleware/auth.ts`) I'd dispatch:
> - review-security — auth middleware change, primary concern
> - review-correctness — verify the new condition is right
>
> Skipping reliability / tests / design / types — nothing material for them in this diff. Sound good?

Or for a larger feature PR:

> For this PR (480 lines across 14 files, new `Image` entity + endpoints + tests) I'd dispatch:
> - review-correctness, review-security, review-reliability, review-tests, review-design — full feature surface
> - review-types — new entity + Zod schema
>
> Anything to add or skip?

The agreed list drives step 4. **Project rules from `CLAUDE.md` and `.claude/rules/`** can also justify a custom emphasis (e.g., a project-mandated accessibility review) — fold those into the proposal.

### 4. Dispatch agents in parallel

Launch all confirmed agents **in one message** using the Agent tool. Each agent receives:
- The file list and raw diff
- Requirements context (for `review-correctness`, if available)
- Instruction to read `CLAUDE.md` and `.claude/rules/` for codebase conventions
- The **Scope Rules** above — agents must drop anything matching them

### 5. Aggregate and report

Parse each agent's structured report and produce a single consolidated output (format below).

**If the review is long** (more than ~5 findings total, or the markdown wouldn't fit on one screen), **write it to a file** and show only a short summary + path in chat. Suggested location:

- `.claude/reviews/pr-<num>-review.md` for PR reviews
- `.claude/reviews/branch-<name>-review.md` for branch reviews

The chat output for long reviews should contain: counts per severity, the top 3 critical findings, and the file path. The user opens the file in their editor to read the full report.

For short reviews (≤ 5 findings), present the full output in chat directly.

Output format:

```markdown
# PR Review: <title>

**Change set**: <N files changed, +additions / -deletions>
**Requirements**: <linked issue/PRD title, or "none linked">
**Reviewers run**: <list>

---

## Ratings

*From review-design:*
- **Architecture adherence**: X/10 — <one-line justification>
- **Convention compliance**: X/10 — <one-line justification>
- **Clarity**: X/10 — <one-line justification>

*From review-types (if run), per type:*
- **`TypeName`**: Encapsulation X/10, Invariant Strength X/10, Usefulness X/10, Enforcement X/10

---

## 🔴 Critical — must fix before merge
- **[agent-name]** `file:line` — <issue> / *Fix:* <recommendation>

## 🟠 Important — should fix
<same format>

## 🟡 Suggestions
<terse, one bullet each>

## Silent Failure Audit
<from review-reliability>

## Strengths
<3–5 bullets max>

## Recommended action
1. Fix N critical issues
2. Address M important issues
3. Consider K suggestions
4. Re-run `/code-review` after fixes
```

## Posting Comments to the PR

When the user asks to post comments (e.g., "add comments to the PR", "post this review", "comment on the PR"):

1. **Show the full review to the user first.** If long, the file from step 5 is enough — summarize what's about to be posted (count per severity, files affected, agent attribution).
2. **Get explicit approval before posting.** Wait for the user to confirm. Never auto-post.
3. **Post with attribution.** Every comment ends with the attribution footer below.

See `references/pr-comments.md` for the per-finding posting format and the GitHub Reviews API commands.

### Attribution footer

Every comment posted to a remote PR — inline comments, standalone reviews, approval/request-changes messages — **must end with**:

```
---
🤖 Drafted by Claude Code, reviewed and approved by @{username}
```

Get `{username}` via `gh api user --jq .login`. The attribution is non-negotiable: it tells readers the comment was AI-drafted and human-approved before posting.

## Orchestrator Rules

- Route and aggregate only — never duplicate an agent's analysis.
- Attribute every finding to its source agent.
- If an agent fails or times out, report which one and continue.
- If the diff is empty, report that and stop.
