---
name: code-review
description: Review pull request code changes or local diffs. Triggers on "review PR", "review my changes", "code review", "check the diff", "look at the changes". Works with GitHub (gh CLI) and Bitbucket (REST API). Uses local git for reading diffs — only calls platform APIs for posting comments and approvals.
---

# Code Review

## Philosophy

- **Review the change, not the codebase.** Only flag issues introduced or worsened by this diff. Pre-existing problems are out of scope.
- **Quality over quantity.** 3 real issues beat 15 nitpicks. A noisy review gets ignored.
- **Be specific and actionable.** Every finding needs: what's wrong, why it matters, what to do instead.
- **Assume competence.** The author probably had a reason. If something looks odd, check context (`git blame`, full file) before flagging.
- **Filter before reporting.** If you're not confident an issue is real, don't report it. False positives destroy trust.

## Workflow

### 1. Load criteria

- Load `references/review-criteria.md` as baseline.
- Load the project's `CLAUDE.md` and `.claude/rules/`.
- **Project conventions override defaults on conflict, extend on new rules** — if the project says "we allow `any` in test files", that overrides the default type-safety rule for tests.

### 2. Understand intent

- Read the PR title and description.
- Run `git diff --stat` to see scope — which areas are touched, how large is the change.
- Don't read the full diff yet.

### 3. Confirm review aspects with the user (always)

Before reading the diff or producing findings, **propose a list of review aspects and get user confirmation**. This applies to every review, regardless of PR size — the user knows their priorities better than you do, and confirming up front prevents wasted effort.

Build the aspect list from:
- The tier categories in `references/review-criteria.md` (security, correctness, error handling, tests, etc.)
- Project rules from `CLAUDE.md` and `.claude/rules/` (accessibility, performance budgets, type-safety conventions, etc.)
- PR-specific signals from step 2 (e.g., touches database migrations → propose a "schema/migration" aspect; touches auth → propose a "security & authorization" aspect)

Present the list and ask for confirmation or adjustment. Example:

> For this PR I'd like to review along these aspects:
> 1. Security & authorization (touches auth middleware)
> 2. Correctness & state — including partial-state risks
> 3. Error handling & resilience
> 4. Test coverage for new code paths
> 5. Database migration backward-compat (project rule)
>
> Want me to adjust this list, or proceed?

The agreed aspects drive everything that follows: what you look for, how you classify findings, and (for large PRs) how you split parallel subagents.

### 4. Read the diff

- Run `git diff` directly and read the output. Claude Code pages large tool outputs to disk automatically — no need to manually save unless you need to grep across the whole diff.
- For ambiguous hunks, get surrounding context with `git show {hash}:path`.
- Check tests last, after understanding the code changes.

For very large PRs (~3000+ lines, or when you can't reason about the whole change in one pass), use parallel subagents — see [Parallel Review for Large PRs](#parallel-review-for-large-prs) below.

For exact git commands, platform detection, and posting comments, see `references/workflow.md`.

### 5. Reason about what could go wrong

This is where findings come from — not from scanning a checklist. For each agreed aspect, ask:

1. **What does this code do?** Understand the operations, data flow, and side effects.
2. **What could go wrong in production?** Failure modes, edge cases, partial states. What happens if any step fails partway through? What assumptions does the code make about its inputs? What does a caller see when this fails?
3. **What's the blast radius?** Auth, payments, and data mutations have higher stakes than logging or formatting.

The tiers below are for **classifying and filtering** what you found, not for generating findings.

### 6. Filter, then classify

**Filter first.** For each candidate finding, check whether it should be dropped before you even think about severity:

1. **Is this real?** Read full file context if the diff is ambiguous. Is it pre-existing? → drop.
2. **Is this intentional?** Could the author have done it on purpose? Check the PR description and related code. → drop if likely intentional.
3. **Does it match anything in [What NOT to Flag](#what-not-to-flag)?** → drop.
4. **Is your justification "harmless but worth mentioning", "minor", "noise", "consider", "cleanup"?** → drop. These are nitpick words. If you can't say it would actually cause a problem, don't flag it.
5. **Is this actionable?** Can you describe a specific fix? If not → drop.
6. **Is your confidence low?** → drop.

**Then classify** what survives. Use [Severity Tiers](#severity-tiers) to assign must-fix, should-fix, or suggestion. Tier 3 is **not a dumping ground for nitpicks** — it's reserved for substantive issues like genuinely confusing naming or obviously overcomplicated code that don't rise to blocking. If a finding only deserves to exist as Tier 3 because nothing else fits, it probably should have been dropped at the filter step.

A review with zero comments is a valid review.

### 7. Present the review and get approval before posting

Use the structure in `references/output-format.md`.

**For self-review:** Report findings directly to the user and offer to fix them locally. Done.

**For remote PRs:**
1. **Show the full review to the user first.** Never post comments to a remote PR without explicit user approval — the user's name is on the comments and they need to vet them.
2. **Ask the user to confirm.** Wait for approval (or edits) before any API call.
3. **Post with attribution.** Every posted comment must include a footer attributing it to Claude Code with the approving user's identity. See `references/output-format.md` for the exact attribution format.
4. Use the platform API commands in `references/workflow.md`.

## Severity Tiers

Examples of common patterns, not an exhaustive checklist. Use judgment when something doesn't fit neatly.

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
- **No-op or whitespace-only changes** — even if they "add noise" or "muddy git blame", they're harmless. Don't flag.
- **Speculative issues** — "might break if someone passes X" without evidence it can happen
- **Nitpicks** — minor preferences a senior engineer wouldn't mention. If your reasoning starts with "harmless but…", "consider…", "minor cleanup…", "worth noting…" — it's a nitpick.
- **Missing features** — the PR does what it says; don't scope-creep the review
- **Things project rules explicitly allow** — if CLAUDE.md says it's fine, it's fine

## Parallel Review for Large PRs

For small and medium PRs, do the review yourself in a single pass against the agreed aspects.

For large PRs (~3000+ lines, or when the diff is too big to reason about holistically), **dispatch one subagent per agreed aspect in parallel**. Reuse the aspect list confirmed in step 3 — don't re-derive it.

Each subagent gets:
- The full diff (or diffstat + instructions to fetch it)
- The PR title and description
- Its specific aspect, with the relevant criteria from `references/review-criteria.md` and project rules
- An instruction to apply the same philosophy: review the change, quality over quantity, filter before reporting

After all subagents return, the main agent:
- Collects findings from each
- Deduplicates (the same issue may surface from multiple aspects)
- Classifies by severity using the tiers
- Presents a unified review in the standard output format

For the exact subagent dispatch pattern, see `references/workflow.md`.

## Reference Files

- **`references/review-criteria.md`** — Expanded examples and edge cases for each review tier. Consult when unsure how to classify a finding.
- **`references/output-format.md`** — The structure your final review should follow (summary, findings by severity, recommendation). Consult before presenting the review.
- **`references/workflow.md`** — Git commands for fetching diffs, platform detection (GitHub/Bitbucket), API calls for posting comments and approvals, and the subagent dispatch pattern for parallel review.
