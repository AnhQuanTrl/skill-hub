---
name: code-review
description: Review pull request code changes or local diffs. Triggers on "review PR", "review my changes", "code review", "check the diff", "look at the changes". Works with GitHub (gh CLI) and Bitbucket (REST API). Uses local git for reading diffs — only calls platform APIs for posting comments and approvals.
---

# Code Review

## Philosophy

- **Review the change, not the codebase.** Only flag issues introduced or worsened by this diff. Pre-existing problems are out of scope — if it was there before, it's not this PR's job.
- **Quality over quantity.** 3 real issues beat 15 nitpicks. A noisy review gets ignored.
- **Be specific and actionable.** Every finding needs: what's wrong, why it matters, what to do instead. "This looks wrong" is not a review comment.
- **Assume competence.** The author probably had a reason. If something looks odd, check context (`git blame`, full file) before flagging.
- **Filter before reporting.** If you're not confident an issue is real, don't report it. False positives destroy trust.

## Criteria Loading

1. Load `references/review-criteria.md` as baseline
2. Load the project's `CLAUDE.md` and `.claude/rules/`
3. **Project conventions override defaults on conflict, extend on new rules** — if the project says "we allow `any` in test files", that overrides the default type-safety rule for tests

## How to Think About the Change

Before matching patterns from a checklist, reason about the code:

1. **What does this code do?** Understand the operations, data flow, and side effects introduced by the diff.
2. **What could go wrong in production?** Think about failure modes, edge cases, and states the code can reach. Consider: what happens if any step fails partway through? What assumptions does this code make about its inputs? What does a caller see when this fails?
3. **What's the blast radius?** If this code is wrong, what breaks? Auth, payments, and data mutations have higher stakes than logging or formatting.

This is where findings come from — not from scanning a list, but from understanding the change and reasoning about its consequences. The tiers below are for **classifying and filtering** what you found, not for generating findings.

## Severity Tiers

Use these to classify findings and decide what to report. They are examples of common patterns, not an exhaustive checklist.

### Tier 1 — must-fix (block the PR)

Issues a senior engineer would always flag: correctness errors (logic bugs, wrong conditions, off-by-one, race conditions, incorrect state mutations), breakage (syntax/type errors, missing imports, broken callers after rename/removal), security (injection, hardcoded secrets, auth loosening, unsafe deserialization).

### Tier 2 — should-fix (flag, may not block alone)

Error handling gaps (silent failures, swallowed errors, missing propagation), incomplete changes (partial refactors, dead code, untracked TODOs), missing test coverage for new code paths or edge cases visible in the diff.

### Tier 3 — suggestion (never block)

Genuinely confusing naming, misleading comments, obviously overcomplicated code where a simpler approach exists.

For expanded examples and edge cases, read `references/review-criteria.md`.

## What NOT to Flag

- **Style and formatting** — linters handle this
- **Pre-existing issues** not introduced or worsened by this change
- **Speculative issues** — "might break if someone passes X" without evidence it can happen
- **Nitpicks** — minor preferences a senior engineer wouldn't mention
- **Missing features** — the PR does what it says; don't scope-creep the review
- **Things project rules explicitly allow** — if CLAUDE.md says it's fine, it's fine

## Assessing Findings

Before reporting a finding, verify it:

1. **Is this real?** Read the full file context (`git show {hash}:path`) if the diff alone is ambiguous. Check git blame — is this pre-existing?
2. **Is this intentional?** Could the author have done this on purpose? Check the PR description, comments, related code.
3. **Is this actionable?** Can you describe the specific fix? If you can't suggest what to do instead, reconsider whether it's worth flagging.
4. **Assign severity** using the tiers above. If it doesn't fit neatly, use your judgment — the tiers are a guide, not a rulebook.

If confidence is low on any finding, drop it. A review with zero comments is a valid review.

## How to Read the Diff

1. **Understand intent first.** Read the PR title and description before looking at code.
2. **Start with diffstat** (`git diff --stat`) to see scope — which areas are touched, how large is the change.
3. **Save the diff to disk** so you can reference it repeatedly without re-running commands.
4. **Work file-by-file** for large diffs (>2000 lines). Don't try to hold the entire diff in context.
5. **Get surrounding context** when a hunk is unclear — read the full file, not just changed lines.
6. **Check test coverage last.** After understanding the code changes, verify tests match.

For the exact git commands, platform detection, and API calls for posting comments, read `references/workflow.md`.

## Output Format

**Summary:** 1-2 sentences on scope and overall assessment.

**Findings grouped by severity (must-fix first):**

For each finding:
- File path and line number
- What's wrong and why it matters
- Suggested fix or alternative

**Recommendation:**
- **Approve** — no must-fix issues (may have suggestions)
- **Request changes** — has must-fix issues, summarize what needs to change

If no significant issues found: confirm the review is clean with a brief note on what looks good. Don't invent findings to justify the review.

## Reference Files

- **`references/review-criteria.md`** — Expanded examples and edge cases for each review tier. Consult when unsure whether something is Tier 1 vs Tier 2, or whether to flag at all.
- **`references/workflow.md`** — Git commands for fetching and navigating diffs, platform detection (GitHub/Bitbucket), API calls for posting comments and approvals. Consult when you need the exact commands.
