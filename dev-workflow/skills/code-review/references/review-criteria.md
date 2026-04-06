# Review Criteria — Expanded Examples

Expanded examples and edge cases for each review tier. The tiers themselves are defined in SKILL.md — this file helps when you're unsure how to classify a finding.

---

## Tier 1 Edge Cases

### Correctness

**Flag:**
- `if (items.length > 0)` when the logic requires `>= 0` (off-by-one)
- Mutating a shared object inside a loop that's read elsewhere
- `await` missing on an async call where the result is used later
- Comparison with `==` in a language where `===` matters and the types differ

**Don't flag:**
- A slightly unconventional algorithm that still produces correct results
- Performance differences unless they cause correctness issues (e.g., timeout)

### Breakage

**Flag:**
- A function parameter renamed but callers in other files still pass the old name
- A type narrowing removed, causing downstream code to break on `null`
- An import path changed but not updated in re-exports

**Don't flag:**
- A function renamed where all callers in the diff are also updated (complete refactor)
- Deprecated API usage that still works and isn't newly introduced

### Security

**Flag:**
- String concatenation in SQL queries, even with "trusted" input
- `dangerouslySetInnerHTML` with user-provided content
- `chmod 777` or world-readable permissions on config files
- API key in a constant, even if the file is gitignored (it wasn't before)

**Don't flag:**
- Using `eval()` in a build script that only runs in CI with controlled input
- Test fixtures with hardcoded "secret" values clearly marked as test data

---

## Tier 2 Edge Cases

### Error Handling

**Flag:**
- `catch (e) {}` — empty catch with no comment explaining why
- `catch (e) { return null }` when callers don't expect null
- Logging the error but not re-throwing when the caller needs to know

**Don't flag:**
- `catch` with a comment like `// expected when connection is closed` if genuinely expected
- Error handling that matches an established pattern in the codebase

### Incomplete Changes

**Flag:**
- Renamed `getUserName()` to `getDisplayName()` but three callers still use the old name
- Feature flag added but never checked in the code path it gates
- Migration file added but schema docs not updated

**Don't flag:**
- Dead code that exists in the base branch and isn't touched by this PR
- A TODO with an issue link (tracked follow-up is fine)

### Missing Test Coverage

**Flag:**
- New error handling path with no test for the error case
- New branch in conditional logic (`if/else`) where only one branch is tested
- Validation logic added but no test with invalid input

**Don't flag:**
- Trivial getters/setters
- Code already covered by integration tests (check before flagging)
- Test file changes that are clearly part of a test-focused PR

---

## Tier 3 Edge Cases

### Naming

**Flag:**
- `data` as the name for a specific domain object (e.g., `data` is actually a `UserProfile`)
- `handleClick` on a function that submits a form, makes an API call, and navigates
- Boolean named `flag` or `status` with no indication of what it represents

**Don't flag:**
- `i`, `j`, `k` in short loops
- Abbreviations established in the codebase (`ctx`, `req`, `res`)
- Name preference differences (`getUserById` vs `findUser`)

### Simplification

**Flag:**
- 5 nested `if` statements that could be early returns
- Hand-rolled logic that duplicates a standard library function
- A 200-line function doing 4 unrelated things

**Don't flag:**
- Slightly verbose code that's clearer than the clever alternative
- Duplication across 2 call sites (premature abstraction is worse)

---

## The "Should I Flag This?" Checklist

When unsure, run through these questions:

1. **Is this introduced by this PR?** If no → don't flag (pre-existing).
2. **Would a senior engineer on this team flag this?** If no → probably a nitpick.
3. **Can I describe the specific fix?** If no → reconsider.
4. **Would this cause a bug, outage, or security issue in production?** If yes → Tier 1.
5. **Would this make the next person's job harder?** If yes → Tier 2.
6. **Is this just a preference?** If yes → don't flag or Tier 3 at most.
