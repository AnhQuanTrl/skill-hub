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

### Incomplete Changes (including dead code)

**Flag:**
- Renamed `getUserName()` to `getDisplayName()` but three callers still use the old name
- Feature flag added but never checked in the code path it gates
- Migration file added but schema docs not updated
- Functions, imports, or branches reachable only by callers this PR removes
- Commented-out code blocks introduced by the change
- Unused exports introduced by this PR
- Conditionally-dead branches (`if (false)`, code after an early return that always fires)
- Variables assigned but never read

**Don't flag:**
- Dead code that exists in the base branch and isn't touched by this PR (pre-existing)
- A TODO with an issue link (tracked follow-up is fine)
- Unused-but-exported helpers in a library where unused-export is the established norm
- Code reachable only by tests that this PR doesn't touch

### Performance

The bar: would a user notice on realistic input?

**Flag:**
- Defeated `React.memo` from unstable references passed to memoized children (new function/object literal as a prop on every render)
- O(n²) loops over user-controlled or request-sized data
- Blocking the main thread on UI handlers (synchronous parsing of large JSON, sync `fs` in Node request paths)
- N+1 queries on a request path
- Unbounded `useEffect` dependencies that re-run a network call on every render

**Don't flag:**
- Micro-optimizations on cold paths (one-shot config parsing, build scripts)
- Optimization opportunities without a concrete user-visible impact
- Speculative "this could be slow if..." without evidence the input grows

### Type Safety

**Key principle:** type bypasses are load-bearing assertions about runtime behavior with **zero enforcement**. An author comment explaining a bypass is *documentation* of a risk, not *mitigation* of it — the underlying runtime risk is unchanged.

**Flag (TypeScript):**
- `as any`, `as unknown as X` to bypass a real type error
- `// @ts-ignore` / `// @ts-expect-error` without a tracking issue link or justifying comment
- Non-null assertion (`!`) on values that can genuinely be null
- `any` parameters or returns in non-test code
- Type assertions on third-party API return values where the cast is runtime-only and the SDK could rename the field (the code will compile but break at runtime if the field changes)

**Flag (Python):**
- `cast()` to bypass a type error rather than fix it
- `# type: ignore` without a justifying comment
- `Any` in public function signatures
- Missing return type annotations on public functions

**Don't flag:**
- `as` casts in tests where the test setup guarantees the type
- `any` at a genuinely-untyped third-party integration boundary, with a comment explaining the boundary
- Type assertions in generated code

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
2. **Would a senior engineer on this team flag this?** If no → probably a nitpick → don't flag.
3. **Can I describe the specific fix and the concrete problem it prevents?** If no → don't flag.
4. **Is my justification "harmless but…", "minor cleanup", "consider…", "adds noise"?** If yes → don't flag. These words are a tell that you're nitpicking.
5. **Does this bypass type checking, regress performance on a hot path, or leave dead code in the diff?** If yes → flag. The author's explanation, if any, is documentation, not mitigation.
6. **Would this cause a bug, outage, or security issue in production?** If yes → Tier 1.
7. **Would this make the next person's job harder in a concrete way?** If yes → Tier 2.
8. **Is it just a preference or a style call?** → don't flag. Tier 3 is for substantive non-blocking issues (confusing naming, overcomplicated code), not preferences.
